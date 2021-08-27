# Envío de Emails dentro de la API
El comportamiento de la api es notificar al usuario con un email desde la ONG que se registró correctamente, y darle la bienvenida al sitio.
Tambien se notifica mediante un email cuando se agrega un nuevo contacto. 

Se utiliza el servicio de mensajería SendGrid para implementar esta funcionalidad.

1) Se agregan los datos del email de bienvenida dentro de appsettings.json

```
"JWT": {
    "Secret": "ByYM000OLlMQG6VVVp1OH7Xzyr7gHuw1qvUC5dcGt3SNM"
  },
  "SENDGRID_API_KEY": {
    "Key": "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",
    "FromEmail": "grupo58alkemy@gmail.com",
    "WeolcomeSubject": "Bienvenido",
    "WelcomeMessage": "Usted ha sido registrado exitosamente en nuestra plataforma.",
    "ContactMessage": "Gracias por contactarnos",
    "ContactBodyMessage": "Nuestro equipo se va a comunicar a la brevedad"
  }

```

2) Se crea una plantilla html, que tendrá toda la estructura y estilos del email que vamos a enviar.
3) Se crea un servicio de envío de emails, donde se reemplazan las variables de la plantilla html por el contenido dentro de appsettings.json

```
        public async Task<bool> SendContatcsEmail(string email)
        {
            var apiKey = _configuration["SENDGRID_API_KEY:Key"];
            string html = File.ReadAllText("./Templates/email_template.html");

            html = html.Replace("{mail_title}", _configuration["SENDGRID_API_KEY:ContactMessage"]);
            html = html.Replace("{mail_body}", _configuration["SENDGRID_API_KEY:ContactBodyMessage"]);
            

            var client = new SendGridClient(apiKey);
            var from = new EmailAddress(_configuration["SENDGRID_API_KEY:FromEmail"]);
            var to = new EmailAddress(email);
            string subject = _configuration["SENDGRID_API_KEY:WeolcomeSubject"];
            var msg = MailHelper.CreateSingleEmail(from, to, subject, "", html);
            var response = await client.SendEmailAsync(msg).ConfigureAwait(false);

            return response.IsSuccessStatusCode;
        }

        public async Task<bool> SendRegisterEmail(string toEmail)
        {
            var organization = await _organizationService.GetFirst();

            var apiKey = _configuration["SENDGRID_API_KEY:Key"];
            string html = File.ReadAllText("./Templates/email_template.html");

            html = html.Replace("{mail_title}", _configuration["SENDGRID_API_KEY:WelcomeMessage"]);
            html = html.Replace("{mail_body}", _configuration["SENDGRID_API_KEY:ContactBodyMessage"]);
            html = html.Replace("{mail_contact}", organization.Adress + "<br>" + organization.Phone);

            var client = new SendGridClient(apiKey);
            var from = new EmailAddress(_configuration["SENDGRID_API_KEY:FromEmail"]);
            var to = new EmailAddress(toEmail);
            string subject = _configuration["SENDGRID_API_KEY:WeolcomeSubject"];
            var msg = MailHelper.CreateSingleEmail(from, to, subject, "", html);
            var response = await client.SendEmailAsync(msg).ConfigureAwait(false);

            return response.IsSuccessStatusCode;

        }
```

4) Implementar el servicio dentro de otros servicios que lo requieran

- Se envía un email de al añadir un contacto

```
  public class ContactsService : IContactsService
    {
        private readonly IUnitOfWork _unitOfWork;
        private readonly ISendEmailService _sendEmailService;

        public ContactsService(IUnitOfWork unitOfWork, ISendEmailService sendEmailService)
        {
            _unitOfWork = unitOfWork;
            _sendEmailService = sendEmailService;
        }

        public async Task<IEnumerable<ContactsModel>> GetContacts()
        {
            return await _unitOfWork.ContactsRepository.GetAll();
        }

        public async Task<ContactsModel> Post(ContactsCreateDto contactsCreateDto)
        {
            var mapper = new EntityMapper();
            var contact = mapper.FromContactsCreateDtoToContacts(contactsCreateDto);

            await _unitOfWork.ContactsRepository.Insert(contact);
            await _unitOfWork.SaveChangesAsync();
            bool response = await _sendEmailService.SendContatcsEmail(contact.Email);

            return contact;
        }
    }

```

- Se envía un email al registrarse un nuevo usuario (este servicio se llama en user, pero en realidad es aparte, porque integra el envío de mails una vez se generó el token (JWT))

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

```
