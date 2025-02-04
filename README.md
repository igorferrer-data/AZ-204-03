# AZ-204-03
# Criando um Gerenciador de Catálogos com Azure Functions e Banco de Dados

## 1. Introdução

O objetivo é criar uma aplicação baseada em microsserviços para gerenciar catálogos, utilizando:
- Azure Functions para lógica de negócio.
- Redis Cache para escalabilidade.
- Cosmos DB como banco de dados não relacional.
- Storage Account para armazenar vídeos e imagens.
- Um front-end simples criado com Streamlit (Python) para exibir os dados.

## 2. Criando a Infraestrutura na Azure

Provisionamento de recursos necessários:
- Resource Group: FlixDIO.
- API Management (plano de consumo).
- Substituição do SQL por Cosmos DB para maior flexibilidade.
- Criação de containers no Storage Account para armazenar vídeos e imagens.

## 3. Azure Function: Upload de Arquivos no Storage Account

- Desenvolvimento de uma função HTTP Trigger para receber arquivos (vídeos e imagens) via POST e armazená-los no Storage Account.
- Configuração do limite de tamanho do Body da requisição para suportar uploads maiores.

## 4. Azure Function: Salvando Registros no Cosmos DB

- Criação de uma função HTTP Trigger que salva metadados dos vídeos (título, ano, URL do vídeo e thumbnail) no Cosmos DB.
- Configuração do Cosmos DB com chave de partição e container "movies".

## 5. Azure Function: Detalhes de um Filme

Função GET que consulta o Cosmos DB por ID e retorna os detalhes do filme.

## 6. Azure Function: Listagem de Filmes

Função GET que retorna todos os registros salvos no Cosmos DB.

# Scripts

## 1. Provisionamento da Infraestrutura
Utilize o CLI da Azure ou o portal para criar os recursos:

```bash
# Criar Resource Group
az group create --name FlixDIO --location eastus

# Criar Storage Account
az storage account create --name FlixDIOstorage --resource-group FlixDIO --location eastus --sku Standard_LRS

# Criar Containers no Storage
az storage container create --account-name FlixDIOstorage --name videos --public-access blob
az storage container create --account-name FlixDIOstorage --name images --public-access blob

# Criar Cosmos DB
az cosmosdb create --name FlixDIOcosmosdb --resource-group FlixDIO --locations regionName=eastus failoverPriority=0 isZoneRedundant=False

# Criar Container no Cosmos DB
az cosmosdb sql container create \
    --account-name FlixDIOcosmosdb \
    --database-name FlixDB \
    --name movies \
    --partition-key-path "/id" \
    --throughput 400
```

## 2. Função: Upload ao Storage Account

Essa função utiliza um HTTP Trigger para receber arquivos (vídeos ou imagens) via requisição POST e os armazena no Azure Blob Storage.

```csharp
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Azure.Storage.Blobs;

namespace FlixDIO.Functions
{
    public static class PostDataStorage
    {
        [FunctionName("PostDataStorage")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("Processing a file upload request.");

            // Obtém a connection string do Storage Account
            string connectionString = Environment.GetEnvironmentVariable("AzureWebJobsStorage");

            // Obtém o tipo de arquivo (videos ou images) da query string
            string fileType = req.Query["fileType"];
            if (string.IsNullOrEmpty(fileType) || (fileType != "videos" && fileType != "images"))
            {
                return new BadRequestObjectResult("Invalid fileType. Use 'videos' or 'images'.");
            }

            // Lê o arquivo enviado no corpo da requisição
            var formData = await req.ReadFormAsync();
            var file = formData.Files.FirstOrDefault();

            if (file == null)
            {
                return new BadRequestObjectResult("No file was uploaded.");
            }

            // Cria o cliente do Blob Storage para o container especificado
            var blobClient = new BlobContainerClient(connectionString, fileType);
            await blobClient.CreateIfNotExistsAsync();

            // Obtém o Blob Client para o arquivo enviado
            var blob = blobClient.GetBlobClient(file.FileName);

            // Faz o upload do arquivo para o Blob Storage
            using (var stream = file.OpenReadStream())
            {
                await blob.UploadAsync(stream, overwrite: true);
            }

            log.LogInformation($"File '{file.FileName}' uploaded successfully to container '{fileType}'.");

            // Retorna a URL do arquivo armazenado
            return new OkObjectResult(new
            {
                message = "File uploaded successfully.",
                url = blob.Uri.ToString()
            });
        }
    }
}
```

## 3. Função Salvando no Cosmos DB

Essa função utiliza um HTTP Trigger para receber os dados via POST, processá-los e armazená-los no Cosmos DB. O banco é configurado para criar automaticamente o container "movies" caso ele ainda não exista.

```csharp
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace FlixDIO.Functions
{
    public static class PostDatabase
    {
        [FunctionName("PostDatabase")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
            [CosmosDB(
                databaseName: "FlixDB",
                collectionName: "movies",
                ConnectionStringSetting = "CosmosDBConnection",
                CreateIfNotExists = true)] IAsyncCollector<dynamic> moviesOut,
            ILogger log)
        {
            log.LogInformation("Processing a request to save a movie in Cosmos DB.");

            // Lê o corpo da requisição
            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);

            if (data == null || data.title == null || data.year == null || data.videoUrl == null || data.thumbnailUrl == null)
            {
                return new BadRequestObjectResult("Invalid input. Ensure 'title', 'year', 'videoUrl', and 'thumbnailUrl' are provided.");
            }

            // Adiciona um ID único ao registro
            data.id = System.Guid.NewGuid().ToString();

            // Salva o registro no Cosmos DB
            await moviesOut.AddAsync(data);

            log.LogInformation($"Movie '{data.title}' saved successfully with ID: {data.id}");

            // Retorna o registro salvo como resposta
            return new OkObjectResult(data);
        }
    }
}

```
  
## 4. Função: Detalhes de um Filme

Essa função utiliza um HTTP Trigger para buscar os detalhes de um filme no Cosmos DB com base no ID fornecido na rota da requisição.

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace FlixDIO.Functions
{
    public static class GetMovieDetail
    {
        [FunctionName("GetMovieDetail")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = "movies/{id}")] HttpRequest req,
            [CosmosDB(
                databaseName: "FlixDB",
                collectionName: "movies",
                ConnectionStringSetting = "CosmosDBConnection",
                Id = "{id}",
                PartitionKey = "{id}")] dynamic movie,
            ILogger log)
        {
            log.LogInformation("Fetching details for movie with ID: {id}", req.RouteValues["id"]);

            // Verifica se o filme foi encontrado
            if (movie == null)
            {
                log.LogWarning("Movie with ID '{id}' not found.", req.RouteValues["id"]);
                return new NotFoundObjectResult(new { message = "Movie not found." });
            }

            log.LogInformation("Movie details retrieved successfully for ID: {id}", req.RouteValues["id"]);

            // Retorna os detalhes do filme
            return new OkObjectResult(movie);
        }
    }
}
```

## 5. Função: Listagem de Filmes

Essa função utiliza um HTTP Trigger para buscar e retornar todos os filmes armazenados no container movies do Cosmos DB.

```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace FlixDIO.Functions
{
    public static class GetAllMovies
    {
        [FunctionName("GetAllMovies")]
        public static IActionResult Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = "movies")] HttpRequest req,
            [CosmosDB(
                databaseName: "FlixDB",
                collectionName: "movies",
                ConnectionStringSetting = "CosmosDBConnection")] IEnumerable<dynamic> movies,
            ILogger log)
        {
            log.LogInformation("Fetching all movies from the database.");

            // Verifica se há filmes no banco de dados
            if (movies == null)
            {
                log.LogWarning("No movies found in the database.");
                return new NotFoundObjectResult(new { message = "No movies found." });
            }

            log.LogInformation("Movies retrieved successfully.");

            // Retorna a lista de filmes
            return new OkObjectResult(movies);
        }
    }
}
```
