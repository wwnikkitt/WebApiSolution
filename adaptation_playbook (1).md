# 🔧 "What If It Changes?" — Adaptation Playbook

Your template works. This guide is ONLY for: *the exam is different from what I built — now what?*
Find your situation below and follow the exact steps. Every change is small.

**Golden rules before any change:**
- Use **F2** to rename (updates everywhere at once). Never hand-edit names across files.
- Go in **arrow order**: Entities → DbContext → DTOs → Service → Controller → Program.cs.
- **Build after each layer** so errors stay contained instead of piling up.

---

## CHANGE 1 — Different names (Product→Book, Supplier→Author, etc.)

The most common change. Pure renaming, no new logic.

1. Open the **entity** class. Click the class name → **F2** → type new name → Enter. (Renames it everywhere.)
2. F2 each **property** that changed (Price→Pages, Address→Country…).
3. Do the same for the second entity.
4. Build. Fix any stray red (usually the route string and seed data, which F2 doesn't catch).
5. Update the **route** in the controller manually: `[Route("api/inventory")]` → `[Route("api/library")]`.
6. Update **seed data** in Program.cs to use new property names.

Done. No new methods, no new structure.

---

## CHANGE 2 — A third table (e.g. add Category)

A Product now belongs to TWO parents (Supplier AND Category). Copy the Supplier pattern.

1. **New entity** `Category.cs` — copy Supplier's shape:
   ```csharp
   public class Category {
       public int Id { get; set; }
       public string Name { get; set; }
       public ICollection<Product> Products { get; set; }
   }
   ```
2. **On Product**, add the belongs-to pair:
   ```csharp
   public int CategoryId { get; set; }
   public Category Category { get; set; }
   ```
3. **AppDbContext** — add `public DbSet<Category> Categories { get; set; }` and copy the relationship block, swapping names:
   ```csharp
   modelBuilder.Entity<Product>()
       .HasOne(p => p.Category)
       .WithMany(c => c.Products)
       .HasForeignKey(p => p.CategoryId)
       .OnDelete(DeleteBehavior.NoAction);
   ```
4. **DTOs** — add `public record CategoryDto(int Id, string Name);`, add `CategoryDto Category` to `ProductDto`, add `int CategoryId` to `CreateProductDto`.
5. **Service `ProductQuery()`** — add `.Include(p => p.Category)` and `new CategoryDto(p.Category.Id, p.Category.Name)` inside the `new ProductDto(...)`.
6. **Service Add/Update** — add `CategoryId = dto.CategoryId`.
7. **Program.cs seed** — make a category and attach it: `new Product { ..., Category = cat1 }`.
8. (If asked) **GetByCategory** — copy GetBySupplier, swap names.

---

## CHANGE 3 — A new endpoint / filter

Every endpoint touches exactly **3 places**. Copy an existing one and swap.

Example: "get products with stock below a value."

1. **Interface** (`IInventoryService`): `List<ProductDto> GetLowStock(int value);`
2. **Service** (`InventoryService`): copy GetBySupplier, change the Where:
   ```csharp
   public List<ProductDto> GetLowStock(int value) =>
       ProductQuery().Where(p => p.Stock < value).ToList();
   ```
3. **Controller**: copy an existing GET, change route + call:
   ```csharp
   [HttpGet("low-stock/{value}")]
   public IActionResult GetLowStock(int value) => Ok(_service.GetLowStock(value));
   ```

**Filter cheat:** above = `> `, below = `< `, exact = `== `, contains text = `.Contains(name)`.

---

## CHANGE 4 — Remove a CRUD operation (e.g. "no Update")

Just delete it in 3 places:
1. Remove the line from the **interface**.
2. Remove the method from the **service**.
3. Remove the method from the **controller**.
Build. Done.

---

## CHANGE 5 — CRUD on the OTHER table (e.g. add/list Suppliers too)

Same methods you already have, pointed at the other table.

1. **Interface**: add `List<SupplierDto> GetSuppliers();`, `void AddSupplier(...)`, etc.
2. **Service**: copy your Product methods, replace `_context.Products` → `_context.Suppliers`, `ProductDto` → `SupplierDto`.
3. **Controller**: copy your Product endpoints, change routes (`api/suppliers`) and the service calls.

It's the same two recipes (read = Where+reshape, write = change+SaveChanges) aimed at Suppliers.

---

## CHANGE 6 — Different field types or extra fields

E.g. they add `DateTime CreatedAt` or make Price an `int`.

1. Add/change the property on the **entity**.
2. Add it to the **DTO** (and CreateProductDto if the client sends it).
3. Add it to the `new ProductDto(...)` in the service query.
4. Copy it in **Add/Update** (`CreatedAt = dto.CreatedAt`).
5. Set it in **seed** data.

Match the type everywhere (a `string` field stays string in entity + DTO + arguments — mismatches cause the "cannot convert int to string" error).

---

## CHANGE 7 — Different routes or HTTP verbs

Edit only the attributes in the controller; method bodies don't change.
- `[Route("api/inventory")]` → whatever they want.
- `[HttpGet("above-price/{price}")]` → change path or `{param}` name.
- A POST that should be PUT → change `[HttpPost]` to `[HttpPut("{id}")]`.

---

## ABOUT THE NAME "INVENTORY" (renaming is optional)

"Inventory" is NOT a keyword — it's just a label the lecturer chose because the project
tracks stock. You can rename it to match the exam's topic, or leave it. **The grader cares
that the endpoints work, not what you named them.** Even if the exam is about a library and
everything still says "inventory," you get full marks as long as it functions.

It appears in two independent spots:
- **The route**: `[Route("api/inventory")]` in the controller → the URL prefix. Change by hand if you want (`api/library`). F2 will NOT touch text inside `"..."`.
- **The service name**: `InventoryService` / `IInventoryService` → just a chosen name. Rename with **F2** (it auto-updates the controller and the Program.cs registration).

What you MUST match exactly = the endpoint **paths and verbs the task specifies** (e.g. the
lecturer writes `GET /api/inventory`, `PUT /api/inventory/{id}`). Copy those verbatim; the
internal class names are your free choice.

---

## DELETING AN ENDPOINT (the 3 places, in order)

Remove it from all three, then build once:
1. **Controller** — delete the whole method + its `[Http...]` attribute.
2. **Interface** (`IInventoryService`) — delete its one line.
3. **Service** (`InventoryService`) — delete the method body.

⚠️ Don't delete only the interface line — the service will error ("promised but not delivered").
Delete all three together, or controller → interface → service, then build.

---

## THE TWO RECIPES THAT COVER EVERYTHING

If you forget all the above, remember these. Every task is one of them:

**READ (GET):**
```csharp
ProductQuery().Where(<condition>).ToList();   // drop .Where for "get all"
```

**WRITE (POST / PUT / DELETE):**
```csharp
// find or create the entity
// change / add / remove it
_context.SaveChanges();
```

Sort the question into READ or WRITE, copy the matching method you already have, swap the names. That's the exam.

---

## IF SOMETHING BREAKS (quick triage)

- **Red squiggle but you think it's fixed** → Build → **Rebuild Solution** (clears stale errors).
- **"cannot convert int to string"** → a field's type doesn't match between entity/DTO/arguments, or wrong argument order in a `record`.
- **"object reference required"** → you used a Type name where you needed the lowercase variable (e.g. `ModelBuilder` instead of `modelBuilder`).
- **navigation comes back null** (e.g. Supplier empty) → missing `.Include(p => p.Supplier)`.
- **no data on GET** → forgot to seed in Program.cs (In-Memory starts empty).
- **Swagger "Failed to fetch"** → switch to the **http** launch profile.
- **renamed and everything's red** → you renamed by hand and missed spots; undo, use **F2** instead.
