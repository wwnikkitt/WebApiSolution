# 🎯 ASP.NET Core Web API — Exam Cheat Sheet (In-Memory version)

## 0. The mental model (read this first if you panic)

```
WebApi   →   Services   →   Data   →   Domain
(controller)  (logic)     (DbContext)  (entities)
```

Arrows = "depends on / knows about." Never goes backwards.
- **Domain** = what things ARE (plain classes, no logic).
- **Data** = the storage room (DbContext).
- **Services** = the worker who does the jobs (CRUD, filter).
- **WebApi** = the receptionist (controller) + wiring (Program.cs).

**Dependency Injection in one sentence:** a class asks for what it needs in its
constructor (`AppDbContext`, `IInventoryService`), and `Program.cs` registers, once,
who provides it. That's it.

---

## 1. NuGet packages (In-Memory)

| Project | Package | Why |
|---------|---------|-----|
| **Data** | `Microsoft.EntityFrameworkCore.InMemory` (10.0.x) | DbContext, DbSet, LINQ, `UseInMemoryDatabase` |
| **WebApi** | `Swashbuckle.AspNetCore` | Swagger test page |

Domain & Services need NO packages (they get EF through project references).
`Microsoft.AspNetCore.OpenApi` is already there from the template — don't worry about it.

**Project references (set these or nothing builds):**
- Data → Domain
- Services → Data
- WebApi → Services AND Data

---

## 2. Build order from zero

1. `dotnet new sln -n WebApiSolution` (or VS: new ASP.NET Core Web API, solution name = WebApiSolution)
2. Add 3 class libs: Domain, Data, Services. Add WebApi (Web API template, **Use controllers = ON**).
3. Add all 4 to solution. Set the project references above.
4. Delete every `Class1.cs`. Delete `WeatherForecast.cs` + `WeatherForecastController.cs`.

---

## 3. FILE SKELETONS (fill the blanks)

### Domain/Entities/Supplier.cs  (the "one / parent" side)
```csharp
namespace Domain.Entities;

public class Supplier
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
    public ICollection<Product> Products { get; set; }   // has MANY
}
```

### Domain/Entities/Product.cs  (the "many / child" side)
```csharp
namespace Domain.Entities;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }

    public int SupplierId { get; set; }      // foreign key  ← "belongs to" pair
    public Supplier Supplier { get; set; }    // navigation
}
```

### Data/Context/AppDbContext.cs
```csharp
using Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace Data.Context;

public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Supplier> Suppliers { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)   // lowercase m!
    {
        modelBuilder.Entity<Product>().Property(x => x.Name).HasMaxLength(100);

        modelBuilder.Entity<Product>()
            .HasOne(p => p.Supplier)      // a Product (p) has one Supplier
            .WithMany(s => s.Products)    // that Supplier (s) has many Products
            .HasForeignKey(p => p.SupplierId)
            .OnDelete(DeleteBehavior.NoAction);

        base.OnModelCreating(modelBuilder);
    }
}
```

### Services/Dtos/  (records = simple data bags)
```csharp
namespace Services.Dtos;

public record SupplierDto(int Id, string Name, string Address);

public record ProductDto(int Id, string Name, decimal Price, int Stock, SupplierDto Supplier);

public record CreateProductDto(string Name, decimal Price, int Stock, int SupplierId);
```
⚠️ When VS makes a record, DELETE the `class { }` wrapper it generates. A record is NOT inside a class.

### Services/Abstract/IInventoryService.cs  (the menu — names only)
```csharp
using Services.Dtos;

namespace Services.Abstract;

public interface IInventoryService
{
    List<ProductDto> GetProducts();
    List<ProductDto> GetProductsAbovePrice(decimal price);
    void AddProduct(CreateProductDto product);
    bool UpdateProduct(int id, CreateProductDto product);
    bool DeleteProduct(int id);
    List<ProductDto> SearchByName(string name);     // extra #1
    List<ProductDto> GetBySupplier(int supplierId);  // extra #2
}
```

### Services/Impl/InventoryService.cs  (the worker)
```csharp
using Data.Context;
using Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Services.Abstract;
using Services.Dtos;

namespace Services.Impl;

public class InventoryService : IInventoryService
{
    private readonly AppDbContext _context;
    public InventoryService(AppDbContext context) { _context = context; }   // DI

    private IQueryable<ProductDto> ProductQuery() =>
        _context.Products
            .Include(p => p.Supplier)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Stock,
                new SupplierDto(p.Supplier.Id, p.Supplier.Name, p.Supplier.Address)));

    public List<ProductDto> GetProducts() => ProductQuery().ToList();

    public List<ProductDto> GetProductsAbovePrice(decimal price) =>
        ProductQuery().Where(p => p.Price > price).ToList();

    public void AddProduct(CreateProductDto dto)
    {
        _context.Products.Add(new Product {
            Name = dto.Name, Price = dto.Price, Stock = dto.Stock, SupplierId = dto.SupplierId });
        _context.SaveChanges();
    }

    public bool UpdateProduct(int id, CreateProductDto dto)
    {
        var product = _context.Products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;
        product.Name = dto.Name; product.Price = dto.Price;
        product.Stock = dto.Stock; product.SupplierId = dto.SupplierId;
        _context.SaveChanges();
        return true;
    }

    public bool DeleteProduct(int id)
    {
        var product = _context.Products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;
        _context.Products.Remove(product);
        _context.SaveChanges();
        return true;
    }

    public List<ProductDto> SearchByName(string name) =>
        ProductQuery().Where(p => p.Name.Contains(name)).ToList();

    public List<ProductDto> GetBySupplier(int supplierId) =>
        ProductQuery().Where(p => p.Supplier.Id == supplierId).ToList();
}
```

### WebApi/Controllers/InventoryController.cs  (the receptionist)
```csharp
using Microsoft.AspNetCore.Mvc;
using Services.Abstract;
using Services.Dtos;

namespace WebApi.Controllers;

[ApiController]
[Route("api/inventory")]
public class InventoryController : ControllerBase
{
    private readonly IInventoryService _service;
    public InventoryController(IInventoryService service) { _service = service; }   // DI

    [HttpGet]
    public IActionResult GetProducts() => Ok(_service.GetProducts());

    [HttpGet("above-price/{price}")]
    public IActionResult GetAbovePrice(decimal price) => Ok(_service.GetProductsAbovePrice(price));

    [HttpPost]
    public IActionResult AddProduct(CreateProductDto p) { _service.AddProduct(p); return Ok(); }

    [HttpPut("{id}")]
    public IActionResult UpdateProduct(int id, CreateProductDto p) =>
        _service.UpdateProduct(id, p) ? Ok() : NotFound();

    [HttpDelete("{id}")]
    public IActionResult DeleteProduct(int id) =>
        _service.DeleteProduct(id) ? Ok() : NotFound();

    [HttpGet("search")]
    public IActionResult SearchByName([FromQuery] string name) => Ok(_service.SearchByName(name));

    [HttpGet("by-supplier/{id}")]
    public IActionResult GetBySupplier(int id) => Ok(_service.GetBySupplier(id));
}
```

### WebApi/Program.cs  (the wiring + seed)
```csharp
using Data.Context;
using Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Services.Abstract;
using Services.Impl;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("InventoryDb"));  // ★ the InMemory line
builder.Services.AddScoped<IInventoryService, InventoryService>();                        // ★ menu → worker

var app = builder.Build();

// seed (InMemory starts empty)
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var s1 = new Supplier { Name = "TechSupply", Address = "Tbilisi" };
    var s2 = new Supplier { Name = "FoodCorp", Address = "Batumi" };
    db.Suppliers.AddRange(s1, s2);
    db.Products.AddRange(
        new Product { Name = "Laptop", Price = 1500, Stock = 10, Supplier = s1 },
        new Product { Name = "Mouse",  Price = 25,  Stock = 100, Supplier = s1 },
        new Product { Name = "Coffee", Price = 8,   Stock = 200, Supplier = s2 });
    db.SaveChanges();
}

app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();
app.Run();
```

---

## 4. RENAME CHECKLIST (do it in THIS order)

Use **F2** (Rename) — it updates all references at once. Go in arrow order:

1. **Entities** — rename class + properties (F2 each).
2. **AppDbContext** — `DbSet<X>` names + relationship property names.
3. **DTOs** — record names + fields.
4. **Service** (interface + impl) — method names, query, Add/Update fields.
5. **Controller** — `[Route(...)]`, method names.
6. **Program.cs** — seed data.

Build after each layer; fix red as you go forward.

---

## 5. ADD-A-THIRD-TABLE RECIPE (e.g. add `Category`)

Each "belongs to" = a `XId` + an `X`. Repeat the Supplier pattern:

1. New entity `Category` (copy Supplier's shape: Id, Name, `ICollection<Product> Products`).
2. On Product add: `public int CategoryId { get; set; }` + `public Category Category { get; set; }`.
3. AppDbContext: add `DbSet<Category> Categories`; copy the relationship block, swap names:
   ```csharp
   modelBuilder.Entity<Product>()
       .HasOne(p => p.Category)
       .WithMany(c => c.Products)
       .HasForeignKey(p => p.CategoryId)
       .OnDelete(DeleteBehavior.NoAction);
   ```
4. DTOs: add `CategoryDto`; add it to `ProductDto`; add `int CategoryId` to `CreateProductDto`.
5. Service: add `.Include(p => p.Category)` and `new CategoryDto(...)` in `ProductQuery()`; copy `CategoryId` in Add/Update.
6. Seed a Category and attach it: `new Product { ..., Category = cat1 }`.
7. (If asked) add `GetByCategory(int id)` = copy `GetBySupplier`, swap names.

---

## 6. FILTER / ENDPOINT VARIATIONS (just change the Where)

| They ask for | Code |
|--------------|------|
| above price | `.Where(p => p.Price > price)` |
| below price | `.Where(p => p.Price < price)` |
| low stock (under value) | `.Where(p => p.Stock < value)` |
| search by name | `.Where(p => p.Name.Contains(name))` |
| by supplier | `.Where(p => p.Supplier.Id == id)` |

Each new endpoint = 3 places: interface line, service method, controller method (copy an existing one).

---

## 7. RUN & TEST

- Pick the **http** launch profile (avoids HTTPS cert errors), or run `dotnet dev-certs https --trust` once.
- Swagger opens automatically. Test order: **POST** a product → **GET** to see it → try filters.
- POST body example: `{ "name": "Keyboard", "price": 45, "stock": 30, "supplierId": 1 }` (use a seeded supplierId!).

---

## 8. ERRORS YOU ALREADY HIT → FIXES

| Error / symptom | Cause | Fix |
|-----------------|-------|-----|
| `WeatherForecast could not be found` | leftover template files | delete WeatherForecast.cs + its controller |
| `member names cannot be the same as enclosing type` | record put INSIDE a class | delete the `class { }` wrapper, keep only the record |
| `cannot convert int to string` | entity/DTO field typed wrong or wrong arg order | check property is `string` not `int`; match arg order to record |
| `AddSwaggerGen not found` | Swashbuckle not installed | install `Swashbuckle.AspNetCore` in WebApi |
| `Install greyed out` in NuGet | solution-level manager, no project ticked | tick the WebApi checkbox |
| `object reference required for ModelBuilder.Entity` | wrote `ModelBuilder` (capital) | use `modelBuilder` (lowercase param) |
| `Failed to fetch` in Swagger | untrusted HTTPS dev cert | use http profile OR `dotnet dev-certs https --trust` |
| supplier comes back null | forgot `.Include(p => p.Supplier)` | add the Include in the query |
| no data when GET | InMemory starts empty | seed in Program.cs |

---

## 9. ONE-LINE ANSWERS (if examiner asks "what is X")

- **DTO**: a clean data shape sent to/from the API, separate from the entity.
- **DI**: framework supplies a class's dependencies via its constructor; registered in Program.cs.
- **Interface (IInventoryService)**: a contract listing methods with no bodies; lets you swap implementations.
- **DbContext**: the gateway to the database; `DbSet<T>` = a table.
- **Why InMemory needs +2 endpoints**: exam rule, since it skips migrations.
- **SaveChanges()**: commits staged Add/Update/Remove to the database.
