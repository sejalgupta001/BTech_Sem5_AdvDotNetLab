# API Versioning Strategies in ASP.NET Core with Asp.Versioning.Mvc
- **Version 1 (v1)**: Returns products with `Id`, `Name`, and `Price`.
- **Version 2 (v2)**: Adds a `Category` field.

## Prerequisites
**Add required packages**:
   ```bash
   dotnet add package Asp.Versioning.Mvc --version 8.1.0
   dotnet add package Asp.Versioning.Mvc.ApiExplorer --version 8.1.0
   dotnet add package Swashbuckle.AspNetCore
   ```
## Common Setup
### Models
```csharp
// Models/ProductV1.cs
namespace ApiVersioningDemo.Models
{
    public class ProductV1
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
    }
}

// Models/ProductV2.cs
namespace ApiVersioningDemo.Models
{
    public class ProductV2
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public string Category { get; set; }
    }
}
```

### Program.cs (Base Configuration)
```csharp
using Asp.Versioning;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV"; // e.g., "v1", "v2"
    options.SubstituteApiVersionInUrl = true;
});
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "API v1", Version = "v1" });
    c.SwaggerDoc("v2", new OpenApiInfo { Title = "API v2", Version = "v2" });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "API v1");
        c.SwaggerEndpoint("/swagger/v2/swagger.json", "API v2");
    });
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

## 1. URL Versioning
### Explanation
The version number is embedded in the URL path (e.g., `/api/v1/products`) using separate controllers.

### Demo Code
```csharp
// Controllers/V1/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using ApiVersioningDemo.Models;
using Asp.Versioning;

namespace ApiVersioningDemo.Controllers.V1
{
    [ApiController]
    [Route("api/v{version:apiVersion}/[controller]")]
    [ApiVersion("1.0")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get()
        {
            var products = new List<ProductV1>
            {
                new() { Id = 1, Name = "Laptop", Price = 1000 },
                new() { Id = 2, Name = "Phone", Price = 500 }
            };
            return Ok(products);
        }
    }
}

// Controllers/V2/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using ApiVersioningDemo.Models;
using Asp.Versioning;

namespace ApiVersioningDemo.Controllers.V2
{
    [ApiController]
    [Route("api/v{version:apiVersion}/[controller]")]
    [ApiVersion("2.0")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get()
        {
            var products = new List<ProductV2>
            {
                new() { Id = 1, Name = "Laptop", Price = 1000, Category = "Electronics" },
                new() { Id = 2, Name = "Phone", Price = 500, Category = "Electronics" }
            };
            return Ok(products);
        }
    }
}
```

### How to Test
- `GET https://localhost:5001/api/v1/products` → Returns v1 products.
- `GET https://localhost:5001/api/v2/products` → Returns v2 products.
- Open Swagger UI to see versioned endpoints.

### Notes
- **Pros**: Clear, intuitive, and well-supported by `Asp.Versioning`.
- **Cons**: URLs change with versions.

## 2. Query Parameter Versioning
### Explanation
The version is specified as a query parameter (e.g., `?api-version=1.0`) in a single controller.

### Demo Code
```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using ApiVersioningDemo.Models;
using Asp.Versioning;

namespace ApiVersioningDemo.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [ApiVersion("1.0")]
    [ApiVersion("2.0")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        [MapToApiVersion("1.0")]
        public IActionResult GetV1()
        {
            var products = new List<ProductV1>
            {
                new() { Id = 1, Name = "Laptop", Price = 1000 },
                new() { Id = 2, Name = "Phone", Price = 500 }
            };
            return Ok(products);
        }

        [HttpGet]
        [MapToApiVersion("2.0")]
        public IActionResult GetV2()
        {
            var products = new List<ProductV2>
            {
                new() { Id = 1, Name = "Laptop", Price = 1000, Category = "Electronics" },
                new() { Id = 2, Name = "Phone", Price = 500, Category = "Electronics" }
            };
            return Ok(products);
        }
    }
}
```

### Program.cs Update
Add to the `AddApiVersioning` configuration:
```csharp
options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
```

### How to Test
- `GET https://localhost:5001/api/products?api-version=1.0` → Returns v1 products.
- `GET https://localhost:5001/api/products?api-version=2.0` → Returns v2 products.
- `GET https://localhost:5001/api/products` → Defaults to v1.

### Notes
- **Pros**: Clean URLs, single endpoint.
- **Cons**: Less discoverable.

## 3. Header Versioning
### Explanation
The version is specified in an HTTP header (e.g., `X-API-Version`) in a single controller.

### Demo Code
Use the same `ProductsController.cs` as in Query Parameter Versioning.

### Program.cs Update
Add to the `AddApiVersioning` configuration:
```csharp
options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");
```

### How to Test
- `GET https://localhost:5001/api/products` with header `X-API-Version: 1.0` → Returns v1 products.
- `GET https://localhost:5001/api/products` with header `X-API-Version: 2.0` → Returns v2 products.
- `GET https://localhost:5001/api/products` (no header) → Defaults to v1.

### Notes
- **Pros**: Clean URLs, decoupled versioning.
- **Cons**: Requires header knowledge.

## 4. Consumer-Based Versioning
### Explanation
The version is tied to the consumer (via an API key) using a service to store preferences.

### Notes
- **Pros**: Seamless for clients, persistent versioning.
- **Cons**: Requires a database or service for production.
