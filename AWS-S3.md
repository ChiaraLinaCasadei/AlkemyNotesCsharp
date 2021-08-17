# Incorporando Amazon Web Services al proyecto

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
