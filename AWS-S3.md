# Incorporando Amazon Web Services al proyecto (para alojar imagenes)

1) Instalar paquetes NuGet:
  - AWSSDK.S3 (S3 es el protocolo que usa para subir los archivos)
  - AWSSDK.Extensions.NETCore.Setup (instala el cliente para poder subir los archivos)
2) Crear clase para Amazon Response
```
  public class AwsManagerResponse
    {
        public string Message { get; set; }
        public int? Code { get; set; }
        public string Errors { get; set; }
        public string NameImage { get; set; }
        public string Url { get; set; }
    }
```
3) Implementar interfaz, empleando métodos provistos por Amazon en los paquetes NuGet
```
  public class S3AwsHelper
    {
        private readonly IAmazonS3 _amazonS3; //Implementar Interfaz
        public S3AwsHelper()
        {
            var chain = new CredentialProfileStoreChain("app_data\\credentials.ini"); //archivo dentro de carpeta app-data, proporcionado por Alkemy
            AWSCredentials awsCredentials;
            RegionEndpoint sAEast1 = RegionEndpoint.SAEast1;
            if (chain.TryGetAWSCredentials("default", out awsCredentials)) //crea el cliente, empleando las credenciales
            {
                _amazonS3 = new AmazonS3Client(awsCredentials.GetCredentials().AccessKey, awsCredentials.GetCredentials().SecretKey, sAEast1);
            }
        }
        //metodo que recibe el nombre del archivo (key es el nombre por el que amazon lo interpreta), y el archivo en sí
        
        public async Task<AwsManagerResponse> AwsUploadFile(string key, IFormFile file) 
        {
            try
            {
                var putRequest = new PutObjectRequest() //creo una instancia de PutRequest porque es lo que recibe el método de la linea 34, que ya viene en el NuGet, no hay que crearlo
                {
                    BucketName = "alkemy-ong", //nombre de la partición de Amazon donde están alojados mis archivos
                    Key = key, //nombre del archivo
                    InputStream = file.OpenReadStream(),
                    ContentType = file.ContentType //formato del archivo (.avi, .gif, .jpg, etc...)
                };
                var result = await _amazonS3.PutObjectAsync(putRequest);

                var response = new AwsManagerResponse //es una clase previamente creada
                {
                    Message = "File upload successfully",
                    Code = (int)result.HttpStatusCode,
                    NameImage = key,
                    Url = $"https://alkemy-ong.s3.amazonaws.com/{key}" //copio esto, lo pego en el navegador y me lleva a la imagen. Si lo pongo como src en html me carga la imagen desde amazon
                };
                return response;
            }
            catch (AmazonS3Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Error encountered when writing an object",
                    Code = (int)e.StatusCode,
                    Errors = e.Message
                };
            }
            catch (Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Unknown encountered on server when writing an object",
                    Code = 500,
                    Errors = e.Message
                };
            }
        }

        public async Task<AwsManagerResponse> AwsFileDelete(string key)
        {
            try
            {
                DeleteObjectRequest request = new DeleteObjectRequest
                {
                    BucketName = "alkemy-ong",
                    Key = key
                };

                var result = await _amazonS3.DeleteObjectAsync(request);

                var response = new AwsManagerResponse
                {
                    Message = "File deleted successfully",
                    Code = (int)result.HttpStatusCode,
                    NameImage = key
                };

                return response;

            }
            catch (AmazonS3Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Error encountered when deleting an object",
                    Code = (int)e.StatusCode,
                    Errors = e.Message
                };
            }
            catch (Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Unknown encountered on server when deleting an object",
                    Code = 500,
                    Errors = e.Message
                };
            }


        }

        public async Task<AwsManagerResponse> AwsGetFileUrl(string key)
        {
            try
            {
                var request = new GetObjectRequest()
                {
                    BucketName = "alkemy-ong",
                    Key = key
                };
                using GetObjectResponse response = await _amazonS3.GetObjectAsync(request);
                var result = new AwsManagerResponse
                {
                    Message = "File encountered successfully",
                    Code = 200,
                    NameImage = response.Key,
                    Url = $"https://alkemy-ong.s3.amazonaws.com/{response.Key}"
                };
                return result;
            }
            catch (AmazonS3Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Error encountered when writing an object",
                    Code = (int)e.StatusCode,
                    Errors = e.Message
                };
            }
            catch (Exception e)
            {
                return new AwsManagerResponse
                {
                    Message = "Unknown encountered on server when writing an object",
                    Code = 500,
                    Errors = e.Message
                };
            }
        }
    }
```

4) Ahora que tenemos el Helper creado, podemos utilizarlo en un servicio exclusivo para las imagenes, con su correspondiente interfaz. 
   Este servicio se encarga de guardar una imagen (recibiendo el nombre del archivo, y el archivo en sí). Si la imagen existe (el usuario la proporcionó), entonces la valida (debe ser formato aceptado), si no pasa la validación arroja una excepcion. Si no existe, arroja una excepción diferente. Los errores que pueden darse durante la subida de imagen, es que la imagen no exista, o que el formato sea inválido.
   Para borrar la imagen, primero tiene que existir, y eso es lo que valida el método, luego recurre a metodos ya incluidos en la instancia del helper de amazon, debe recibir el nombre del archivo.

```
  public class ImageService : IImagenService
    {
        private S3AwsHelper _s3AwsHelper;

        public ImageService()
        {
            _s3AwsHelper = new S3AwsHelper();
        }

        public async Task<String> Save(string fileName, IFormFile image)
        {
            AwsManagerResponse responseAws;
            if (image != null)
            {
                if (ValidateFiles.ValidateImage(image))
                {
                    responseAws = await _s3AwsHelper.AwsUploadFile(fileName, image);
                    if (!String.IsNullOrEmpty(responseAws.Errors))
                    {
                        throw new Exception("Error en al guardar imagen. Detalle:" + responseAws.Errors);
                    }

                    return responseAws.Url;
                }
                else
                    throw new Exception("Extensión de imagen no válida. Debe ser jpg, png o jpeg.");
            }
            else
                throw new Exception("Error, no existe imagen.");
        }

        public async Task<bool> Delete(string name)
        {
            if (string.IsNullOrEmpty(name))
            {
                return false;
            }
            else
            {
                string nameImage = await _s3AwsHelper.GetKeyFromUrl(name);
                AwsManagerResponse responseAws = await _s3AwsHelper.AwsFileDelete(nameImage);
                if (!String.IsNullOrEmpty(responseAws.Errors))
                    return false;

                return true;
            }
            

        }
       
    }

```

  5) ¿Cómo validamos las imagenes en el servicio de imagenes?
  INCLUIR EXPLICACIÓN

```
  public class ValidateFiles
    {
        private static readonly Dictionary<string, List<byte[]>> _fileSignature =
        new Dictionary<string, List<byte[]>>
        {
            { ".jpeg", new List<byte[]>
                {
                    new byte[] { 0xFF, 0xD8, 0xFF, 0xE0 },
                    new byte[] { 0xFF, 0xD8, 0xFF, 0xE2 },
                    new byte[] { 0xFF, 0xD8, 0xFF, 0xE3 },
                }
            },
            { ".png", new List<byte[]>
                {
                    new byte[] { 0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A },
                }
            },
        };

        public static string GetImageExtensionFromFile(byte[] file)
        {
            MemoryStream stream = new MemoryStream(file);
            using (var reader = new BinaryReader(stream))
            {
                Dictionary<string, List<byte[]>>.KeyCollection keyColl = _fileSignature.Keys;
                foreach (var ext in keyColl)
                {
                    var signatures = _fileSignature[ext];
                    var headerBytes = reader.ReadBytes(signatures.Max(m => m.Length));
                    bool verification = signatures.Any(signature => headerBytes.Take(signature.Length).SequenceEqual(signature));
                    if (verification)
                    {
                        reader.Close();
                        stream.Close();
                        return ext;
                    }
                    stream.Seek(0, SeekOrigin.Begin);
                }
            }
            stream.Close();
            throw new InvalidDataException();
        }

        public static bool ValidateImage(IFormFile image)
        {
            var postedFileExtension = Path.GetExtension(image.FileName);
            if (!string.Equals(postedFileExtension, ".jpg", StringComparison.OrdinalIgnoreCase)
                && !string.Equals(postedFileExtension, ".png", StringComparison.OrdinalIgnoreCase)
                && !string.Equals(postedFileExtension, ".jpeg", StringComparison.OrdinalIgnoreCase))
            {
                return false;
            }
            return true;
        }
    }

```
