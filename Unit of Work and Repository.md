Se crea una entidad Base, pública y abstracta (porque no vamos a generar instancias de esta clase, solo heredar de ella), que tenga propiedades o atributos comunes a todas las entidades del proyecto
```
  public abstract class EntityBase
      {
          [Key]
          public int Id { get; set; }
          [JsonIgnore]
          public bool IsDeleted { get; set; }
          [Required]
          [DataType(DataType.DateTime)]
          public DateTime CreatedAt { get; set; }
      }

```
Ademas puede contener otros campos de auditoría solicitados para las tablas que vayamos a crear a partir de las Entidades.

---

Se crea un repositorio genérico, una nueva clase, tipo interfaz, en la carpeta de Interfaces, llamado "IRepository".
Como va a ser genérica, vamos a trabajar con un objeto "T", que puede ser cualquier cosa. Esto es útil para cuando varias entidades requieren las mismas operaciones (por ejemplo tipo CRUD).
Para evitar problemas de seguridad, se agregó la restricción de que T debe pertenecer a la clase EntityBase.

```
  public interface IBaseRepository<T> where T : EntityBase
    {
        
        Task<IEnumerable<T>> GetAll();
        Task<T> GetById(int id);
        Task Insert(T entity);
        public Task<UserModel> GetByEmail(string email);
        Task Update(T entity);
        Task Delete(int id);
        bool EntityExists(int id);
    }

```

Para utilizar esto, creamos una implementación, en la carpeta Repositories, una clase pública llamada BaseRepository:

```

  public class BaseRepository<T> : IBaseRepository<T> where T : EntityBase
    {
        private readonly ApplicationDbContext _context;
        private DbSet<T> _entities;

        public BaseRepository(ApplicationDbContext context)
        {
            _context = context;
            _entities = context.Set<T>();
        }

        public async Task<IEnumerable<T>> GetAll()
        {
            var list = await _entities.Where(x => x.IsDeleted == false).ToListAsync();
            return list;
        }

        public async Task<T> GetById(int id)
        {
            return await _entities.FindAsync(id);
        }

        public async Task<UserModel> GetByEmail(string email)
        {
            IQueryable<UserModel> query = _context.Users.Include(u => u.RoleModel);

            var user = await query.Where(x => x.email.ToUpper() == email.ToUpper()).FirstOrDefaultAsync();

            return user;
        }

        public async Task Insert(T entity)
        {
            entity.CreatedAt = DateTime.Now;
            entity.IsDeleted = false;
            await _entities.AddAsync(entity);
        }

        public async Task Update(T entity)
        {
            _entities.Update(entity);
        }

        public async Task Delete(int id)
        {
            var entity = await _entities.FindAsync(id);
            entity.IsDeleted = true;
            _entities.Update(entity);
        }

        public bool EntityExists(int id)
        {
            return _entities.Any(x => x.Id == id && x.IsDeleted == false);
        }
    }

```

Una vez creada la interfaz, hay que registrarla en el Startup.cs, en la parte ConfigurationServices

```
  services.AddScoped(typeof (IRepository<>), typeof (BaseRepository<>));
```
Si no tuviera un generic, serían varios Repos (uno por cada entidad), se agregan así:

```
  services.AddTransient<IUserService, UserService>();
  services.AddTransient<IUserRepository, UserRepository>();
```

## Unit of Work + Patrón Repositorio

Lo que conseguimos con él es tener un único punto de acceso a nuestro acceso a datos.
Unit of Work contiene todos los repositorios que utiliza nuestra aplicación. En software mas grandes hay varias unidades de trabajo.
Tambien nos sirve para manejar las transacciones respecto a las entidades, porque utilizando UoW, vamos a poder compartir la misma conexión en una instancia de la base de datos entre todos los repositorios de nuestra apliación.

Agregamos una nueva infertaz, que se va a llamar IUnitOfWork

```

  public interface IUnitOfWork : IDisposable
    {
        IBaseRepository<UserModel> UserRepository { get; } //lo ponemos como propiedad para usarlo desde nuestros servicios

        void SaveChanges();
        Task SaveChangesAsync();
    }

```

En nuestros repositorios (folder) vamos a crear la implementación de nuestro UoW.

```
  public class UnitOfWork : IUnitOfWork
    {
        private readonly ApplicationDbContext _context;

        public UnitOfWork(ApplicationDbContext context)
        {
            _context = context; //UoW es lo que va a trabajar con el ConnectionStrings, por eso se lo paso
        }
        private readonly IBaseRepository<UserModel> _userRepository;

        public IBaseRepository<UserModel> UserRepository => _userRepository ?? new BaseRepository<UserModel>(_context);
        
        //Si lo que está en ese repositorio no tiene datos, lo instancia, si los tiene devuelve la información.
        
        public void Dispose()
        {
            if (_context != null)
            {
                _context.Dispose();
            }
        }
        public void SaveChanges()
        {
            _context.SaveChanges();
        }
        public async Task SaveChangesAsync()
        {
            await _context.SaveChangesAsync();
        }
    }

```
La Unit of Work tambien debe registrarse en el Startup

```
  services.AddTransient<IUnitOfWork, UnitOfWork>();
```

Ahora nuestro servicio solo recibe UoW instanciado, ya que el repositorio está ahí (igual que todos los otros)

```
  public class UserService: IUserService
    {

        private readonly IUnitOfWork _unitOfWork;
        private readonly IImagenService _imagenService;

        public UserService(IUnitOfWork unitOfWork, IImagenService imagenService)
        {
            _unitOfWork = unitOfWork;
            _imagenService = imagenService;
        }

        public async Task<bool> DeleteUser(int Id)
        {

            try
            {
                UserInfoDto user = await GetUserById(Id);
                if (!string.IsNullOrEmpty(user.photo))
                {
                    bool result = await _imagenService.Delete(user.photo);
                    if (!result) // if there is an error in AWS service to delete the image
                        return false;
                }
                await _unitOfWork.UserRepository.Delete(Id);
                await _unitOfWork.SaveChangesAsync();
                
            }
            catch (Exception)
            {
                return false;
            }

            return true;
        }

        public bool UserExists(int Id)
        {
            return _unitOfWork.UserRepository.EntityExists(Id);
        }

        public async Task<IEnumerable<UserModel>> GetUsers()
        {
            return await _unitOfWork.UserRepository.GetAll();
        }

        public async Task<UserInfoDto> GetUserById(int Id)
        {
            UserModel user = await _unitOfWork.UserRepository.GetById(Id);
            EntityMapper mapper = new EntityMapper();
            UserInfoDto userInfoDto = mapper.FromUserModelToUserInfoDto(user);

            return userInfoDto;
        }
    }

```
