# Authorization
Se agrega como servicio, pero con su propia sección, para distinguirlo de los servicios de los modelos.
Hay endpoints a los que solo puede acceder alguien que se encuentra registrado en la base de datos de usuarios de la página, para eso requiero de una autorización, para mas seguridad.

JWT: Cada usuario tiene un Token asignado, que es único y se genera cada vez que el usuario se loggea. Ese token está asociado a un rol de usuario.
SendGrid: En esta api en particular, tambien utilizo un servicio de envío de emails, para el caso de envío de un mail de bienvenida cuando alguien se registra como usuario

1) Agregamos el NuGet: microsoft.aspnetcore.authentication.jwtbearer

2) Utilizamos instancias de JWT para generación y manejo de tokens.

```
  public class AuthService: IAuthService
    {

        private readonly IUnitOfWork _unitOfWork;
        private readonly IConfiguration _configuration;
        private readonly IImagenService _imagenService;
        private readonly ISendEmailService _sendEmailService;

        public AuthService(IUnitOfWork unitOfWork, IConfiguration configuration, IImagenService imagenService, ISendEmailService sendEmailService)
        {
            this._unitOfWork = unitOfWork;
            this._configuration = configuration;
            this._imagenService = imagenService;
            _sendEmailService = sendEmailService;

        }

        public async Task<UserDto> register( RegisterDTO register)
        {
            var userExists = await _unitOfWork.UserRepository.GetByEmail(register.email);

            if (userExists != null)
            {
                throw new Exception("User already exists!");
            }
            else {
                try
                {
                    register.password = UserModel.ComputeSha256Hash(register.password);

                    var mapper = new EntityMapper();
                    var user = mapper.FromRegisterDtoToUser(register);
                    user.RoleModel = await _unitOfWork.RoleRepository.GetById(2);

                   
                    await _imagenService.Save(user.photo, register.photo);

                    await _unitOfWork.UserRepository.Insert(user);
                    await _unitOfWork.SaveChangesAsync();

                    //Send Registration Email
                    bool response = await _sendEmailService.SendRegisterEmail(user.email);

                    if (user != null)
                    {
                        var token = GetToken(user);
                        var map = new EntityMapper();
                        return map.FromUserToUserDto(user, token);
                    }

                    return null;
                }
                catch (Exception e)
                {
                    throw new Exception(e.Message);
                }
            }
             
           
        }

        public async Task<UserDto> login(LoginDTO login)
        {
            var user = await _unitOfWork.UserRepository.GetByEmail(login.email);  

            if(user != null && (user.password == UserModel.ComputeSha256Hash(login.password)))
            {
                var token = GetToken(user);

                var mapper = new EntityMapper();
                return mapper.FromUserToUserDto(user, token);
            }

            return null;
        }
        
        /*A partir de los datos del usuario, se genera un token unico personalizado, que no solo sirve para identificar al usuario, sino tambien el rol del mismo
        eso permite que, al entrar a un endpoint que requiere que rol sea admin, no permita el acceso a otro usuario que no tenga dicho rol*/
        
        public string GetToken(UserModel user)
        {
            var tokenHandler = new JwtSecurityTokenHandler();

            var key = Encoding.ASCII.GetBytes(_configuration["JWT:Secret"]);

            var authClaims = new List<Claim>
                {
                    new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                    new Claim(ClaimTypes.Email, user.email),
                    new Claim(ClaimTypes.Role, user.RoleModel?.Name),

                };

            var authSigningKey = new SymmetricSecurityKey(key);
            
            /*tambien se le agrega la expiracion del token y se encripta con la clave secreta que tenemos en la configuracion.  */

            var token = new JwtSecurityToken(
                expires: DateTime.Now.AddHours(1),
                claims: authClaims,
                signingCredentials: new SigningCredentials(authSigningKey, SecurityAlgorithms.HmacSha256) //Se encripta con sha256
                );

            return tokenHandler.WriteToken(token);
        }

        public int GetUserId(string token)
        {
            var handler = new JwtSecurityTokenHandler();

            var stringSplit = token.Split(' ');

            var Token = handler.ReadJwtToken(stringSplit[1]);

            var claims = Token.Claims.Where(c => c.Type == ClaimTypes.NameIdentifier).FirstOrDefault();

            var id = int.Parse(claims.Value);

            return (int)id;
        }

    }

```

3) Para aplicarlo a la API, hay que llamarlo en Startup.cs, en la sección ConfigureServices

```
  
    var key = Encoding.ASCII.GetBytes(Configuration["JWT:Secret"]);

            services.AddAuthentication(x =>
            {
                x.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                x.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;

            }).AddJwtBearer(x =>
            {
                x.RequireHttpsMetadata = false;
                x.SaveToken = true;
                x.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(key),
                    ValidateIssuer = false,
                    ValidateAudience = false
                };
            });

```
En appsettings.json es donde se agrega la clave secreta

```
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=baseOng;Trusted_Connection=True;"


  },
  "JWT": {
    "Secret": "acá va la clave secreta"
  },
```

Para testing, hay que completar la seccion Authentication de Postman, pegar el token obtenido como respuesta tras hacer un POST en el endpoint login. El tipo de autenticacion es Bearer Token.
