# Backend — .NET / C# / ASP.NET Core

## Overzicht

.NET is het fundament van enterprise backend development bij Microsoft-stack bedrijven. Met C# schrijf je type-safe, performante en onderhoudbare code. ASP.NET Core is het web framework dat REST API's, gRPC en real-time communicatie mogelijk maakt.

---

## .NET Versies & LTS

| Versie | Type | Support tot |
|--------|------|-------------|
| .NET 8 | LTS | November 2026 |
| .NET 9 | STS | Mei 2026 |
| .NET 10 | LTS | November 2027 |

> **Gebruik altijd een LTS-versie in productie.**

---

## Clean Architecture

De standaard structuur die schaalbaarheid en testbaarheid garandeert:

```
MySolution/
├── src/
│   ├── MyApp.Domain/          # Entities, value objects, domain logic
│   ├── MyApp.Application/     # Use cases, interfaces, DTOs, CQRS
│   ├── MyApp.Infrastructure/  # EF Core, externe services, repositories
│   └── MyApp.API/             # ASP.NET Core controllers, middleware
└── tests/
    ├── MyApp.UnitTests/
    └── MyApp.IntegrationTests/
```

**De gouden regel:** afhankelijkheden wijzen altijd naar binnen (naar Domain). Domain kent niets van Infrastructure.

---

## ASP.NET Core REST API

### Minimal API (modern, .NET 6+)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.MapGet("/api/orders/{id}", async (int id, IOrderService orderService) =>
{
    var order = await orderService.GetByIdAsync(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
})
.WithName("GetOrder")
.WithOpenApi();

app.Run();
```

### Controller-gebaseerde API

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService)
        => _orderService = orderService;

    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id)
    {
        var order = await _orderService.GetByIdAsync(id);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateOrderCommand command)
    {
        var id = await _orderService.CreateAsync(command);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}
```

---

## Dependency Injection

ASP.NET Core heeft ingebouwde DI. Drie lifecycles:

```csharp
// Transient: nieuwe instantie bij elke injectie
builder.Services.AddTransient<IEmailService, SmtpEmailService>();

// Scoped: één instantie per HTTP request
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

// Singleton: één instantie voor de hele applicatie
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();
```

---

## Entity Framework Core

ORM voor MS SQL Server:

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderLine> OrderLines { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.OrderNumber).IsRequired().HasMaxLength(50);
            entity.HasMany(e => e.Lines)
                  .WithOne(l => l.Order)
                  .HasForeignKey(l => l.OrderId);
        });
    }
}

// Repository patroon
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(int id)
        => await _context.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id);
}
```

---

## CQRS met MediatR

Command Query Responsibility Segregation — scheidt lezen van schrijven:

```csharp
// Query
public record GetOrderQuery(int Id) : IRequest<OrderDto?>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderRepository _repo;
    public GetOrderHandler(IOrderRepository repo) => _repo = repo;

    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(request.Id);
        return order is null ? null : new OrderDto(order.Id, order.OrderNumber);
    }
}

// Command
public record CreateOrderCommand(string OrderNumber, int CustomerId) : IRequest<int>;
```

---

## Middleware

```csharp
public class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;

    public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Internal server error" });
        }
    }
}

// Registreer in Program.cs
app.UseMiddleware<ExceptionMiddleware>();
```

---

## Aandachtspunten & Best Practices

- Gebruik `async/await` overal — blokkeer nooit een thread met `.Result` of `.Wait()`
- Valideer inputs met **FluentValidation** of Data Annotations
- Gebruik **Serilog** voor structured logging (JSON output naar Seq/Azure Monitor)
- Schrijf **integration tests** tegen een echte database (testcontainers)
- Versie je API's via URL (`/api/v1/`) of headers
- Gebruik **Health Checks** (`/health`) voor Kubernetes liveness/readiness probes

---

## Nuttige packages

| Package | Gebruik |
|---------|---------|
| `MediatR` | CQRS / in-process messaging |
| `FluentValidation` | Input validatie |
| `Serilog` | Structured logging |
| `AutoMapper` | Object mapping |
| `Polly` | Retry / circuit breaker |
| `Swashbuckle` | Swagger/OpenAPI |
| `xUnit` + `Moq` | Unit testen |
| `Testcontainers` | Integration testen |

---

*[← Terug naar overzicht](../../README.md)*
