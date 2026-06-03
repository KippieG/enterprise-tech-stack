# Cloud — Azure

## Overzicht

Azure is het cloud platform van Microsoft en de natuurlijke keuze voor bedrijven die al in de Microsoft-stack werken. Het integreert naadloos met .NET, SQL Server, Active Directory en Business Central.

---

## Kernservices voor Enterprise Apps

| Service | Gebruik |
|---------|---------|
| **Azure App Service** | Hostplatform voor .NET API's en Angular apps |
| **Azure Kubernetes Service (AKS)** | Container orchestratie op schaal |
| **Azure SQL Database** | Managed MS SQL Server in de cloud |
| **Azure Key Vault** | Secrets, certificaten en sleutels |
| **Azure Blob Storage** | Bestanden, documenten, backups |
| **Azure Service Bus** | Async messaging tussen services |
| **Azure Active Directory (Entra ID)** | Identity & access management |
| **Azure Monitor + Log Analytics** | Logging, metrics, alerting |
| **Azure Container Registry (ACR)** | Private Docker image registry |
| **Azure DevOps** | CI/CD pipelines, boards, repos |

---

## Azure App Service — .NET API Deployment

```bash
# Via Azure CLI
az webapp create \
  --name myapp-api \
  --resource-group myapp-rg \
  --plan myapp-plan \
  --runtime "DOTNET|8.0"

# Connection string instellen
az webapp config connection-string set \
  --name myapp-api \
  --resource-group myapp-rg \
  --connection-string-type SQLServer \
  --settings DefaultConnection="Server=..."

# Deployment via zip
dotnet publish -c Release -o ./publish
zip -r publish.zip ./publish
az webapp deployment source config-zip \
  --name myapp-api \
  --resource-group myapp-rg \
  --src publish.zip
```

---

## Azure Key Vault integratie met ASP.NET Core

```csharp
// Program.cs — laad secrets automatisch uit Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/"),
    new DefaultAzureCredential()
);

// Gebruik in code via IConfiguration — geen secrets in code of config files
var connectionString = builder.Configuration["DatabaseConnectionString"];
```

```bash
# Secret aanmaken in Key Vault
az keyvault secret set \
  --vault-name myapp-kv \
  --name "DatabaseConnectionString" \
  --value "Server=myserver;Database=mydb;..."
```

---

## Azure Service Bus — Async Messaging

```csharp
// Publisher (vanuit order service)
public class OrderCreatedPublisher
{
    private readonly ServiceBusSender _sender;

    public OrderCreatedPublisher(ServiceBusClient client)
        => _sender = client.CreateSender("orders-topic");

    public async Task PublishAsync(OrderCreatedEvent evt)
    {
        var body = JsonSerializer.Serialize(evt);
        var message = new ServiceBusMessage(body)
        {
            Subject = "OrderCreated",
            CorrelationId = evt.OrderId.ToString()
        };
        await _sender.SendMessageAsync(message);
    }
}

// Consumer (in warehouse service)
public class OrderCreatedConsumer : BackgroundService
{
    private readonly ServiceBusProcessor _processor;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _processor.ProcessMessageAsync += async args =>
        {
            var evt = JsonSerializer.Deserialize<OrderCreatedEvent>(
                args.Message.Body.ToString());
            await ProcessOrderAsync(evt!);
            await args.CompleteMessageAsync(args.Message);
        };

        _processor.ProcessErrorAsync += args =>
        {
            _logger.LogError(args.Exception, "Service Bus error");
            return Task.CompletedTask;
        };

        await _processor.StartProcessingAsync(ct);
    }
}
```

---

## Azure Monitor & Application Insights

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry();

// Custom events en metrics loggen
public class OrderService
{
    private readonly TelemetryClient _telemetry;

    public async Task<int> CreateAsync(CreateOrderCommand cmd)
    {
        using var operation = _telemetry.StartOperation<RequestTelemetry>("CreateOrder");
        try
        {
            var id = await _repo.CreateAsync(cmd);
            _telemetry.TrackEvent("OrderCreated", new Dictionary<string, string>
            {
                ["OrderId"] = id.ToString(),
                ["CustomerId"] = cmd.CustomerId.ToString()
            });
            return id;
        }
        catch (Exception ex)
        {
            operation.Telemetry.Success = false;
            _telemetry.TrackException(ex);
            throw;
        }
    }
}
```

---

## Azure Blob Storage

```csharp
public class DocumentStorageService
{
    private readonly BlobServiceClient _blobService;
    private const string ContainerName = "documents";

    public async Task<string> UploadAsync(Stream fileStream, string fileName)
    {
        var container = _blobService.GetBlobContainerClient(ContainerName);
        await container.CreateIfNotExistsAsync(PublicAccessType.None);

        var blobName = $"{Guid.NewGuid()}/{fileName}";
        var blob = container.GetBlobClient(blobName);
        await blob.UploadAsync(fileStream, overwrite: false);

        return blobName;
    }

    public async Task<Stream> DownloadAsync(string blobName)
    {
        var blob = _blobService
            .GetBlobContainerClient(ContainerName)
            .GetBlobClient(blobName);

        var response = await blob.DownloadAsync();
        return response.Value.Content;
    }
}
```

---

## Azure Entra ID (AAD) — JWT authenticatie

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireOrdersRead", policy =>
        policy.RequireClaim("roles", "Orders.Read"));
});
```

```json
// appsettings.json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<your-tenant-id>",
    "ClientId": "<your-api-client-id>",
    "Audience": "api://<your-api-client-id>"
  }
}
```

---

## Migratiepad: On-Prem → Azure

```
Fase 1: Lift & Shift
  On-prem SQL Server → Azure SQL Database (minimale aanpassingen)
  IIS applicaties → Azure App Service

Fase 2: Cloud-native
  App Service → AKS containers
  File shares → Azure Blob Storage
  On-prem messaging → Azure Service Bus

Fase 3: Optimalisatie
  Caching met Azure Cache for Redis
  Scaling rules & auto-scaling
  Cost optimization via reserved instances
```

---

## Best Practices

- Gebruik **Managed Identity** voor service-to-service authenticatie — geen passwords in config
- Alle secrets in **Key Vault** — nooit in code, appsettings of environment variables
- **Resource tagging**: omgeving, team, kostenplaats
- Gebruik **Private Endpoints** voor SQL en Key Vault — niet publiek blootstellen
- **Azure Policy** voor governance (bijv. verplicht versleuteling, verplichte regio's)
- Monitor kosten via **Azure Cost Management** — stel budgetalarmen in

---

*[← Terug naar overzicht](../../README.md)*
