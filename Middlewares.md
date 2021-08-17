# Middlewares
Hay varias formas de hacerlos, el del ejemplo es un Custom Middleware, que se hace como una clase aparte.

```
public class OwnerShipMiddleware
    {
        private readonly RequestDelegate _next;
        public OwnerShipMiddleware(RequestDelegate next)
        {
            _next = next;
        }
        public async Task Invoke(HttpContext context) //En el Invoke es donde se empieza a hacer el middleware
        {
            string path = context.Request.Path; //quiero extraer el Id de la ruta de acceso, por eso el path

            if (path.Contains("users")) //analiza la ruta del archivo, y si contiene el endpoint de Users, se fija si es admin para ver qué es lo que le permite hacer
            {
                string[] splittedPath = path.Split("/");

                var id = context.User.Claims.Where(c => c.Type == ClaimTypes.NameIdentifier).FirstOrDefault()?.Value; //aquí irá el Id del token que le mandamos en la parte de Authorization con Postman

                var rol = context.User.IsInRole("Admin");
                
                // Esto se divide {"https://localhost::.....", "users","id"}
                // Con el único caso donde no hay ID, es si solo tenemos hasta Users, lo cual sería el endpoint de GetAll, y solo el Admin puede obtener este listado.
                //Así que el hecho de que haya llegado a este Middleware, superado el de Authentication, con esta ruta, ya nos dice que es Admin, así qe lo dejamos pasar.
                
                if (splittedPath.Length < 3)
                {
                    await _next.Invoke(context);
                }
                else if (splittedPath[2] == id || rol) //si el id (de lo que quiero alterar) es el mismo del token (osea que el usuario quiere afectarse a sí mismo) Ó el rol es Admin
                {
                    await _next.Invoke(context);
                }
                else
                {
                    context.Response.StatusCode = 403;
                }
            }
            else
            {
                await _next.Invoke(context);
            }
            
        }
    }

```

Se lo invoca en Startup, en la parte de Configuration, detallecito importante, los middleware funcionan como pipeline, entonces el orden en el que se colocan importa.
En este caso, el middleware de Ownership (a menos que seas Admin, solo puedes modificarte a tí mismo), está antes del de Endpoints. Así que antes de dar acceso a los controladores, debemos verificar el rol.

```
  public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseSwagger();
                app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "OngProject v1"));
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            app.UseAuthentication();

            app.UseAuthorization();

            app.UseMiddleware<OwnerShipMiddleware>();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }

```

Los endpoints públicos son los de registro de usuario, login (por obvias razones), y los que sean visuales (por decir así)
