# Step 02: Database Model and DbContext

[⬅️ Step 01: Project Setup](01-project-setup.md) &bull; **Step 02: Database Model and DbContext** &bull; [Next: Step 03 ➡️](03-reading-data-get-endpoints.md)

---

In this step, you will create a strongly-typed C# entity model that maps directly to the existing `Films` SQLite table, configure an Entity Framework Core `DbContext`, set up the connection string in `appsettings.json`, and register the context with ASP.NET Core's Dependency Injection (DI) container.

---

## 1. Analyzing the SQLite Database Schema

Our SQLite database is located at `Data/Films.db`. The table was defined using:

```sql
CREATE TABLE Films (
    filmID INTEGER PRIMARY KEY AUTOINCREMENT,
    filmTitle TEXT NOT NULL CHECK(length(filmTitle) <= 70),
    filmCertificate TEXT NOT NULL CHECK(length(filmCertificate) <= 5),
    filmDescription TEXT NOT NULL,
    filmImage TEXT NOT NULL CHECK(length(filmImage) <= 50),
    filmPrice NUMERIC NOT NULL CHECK(filmPrice >= 0),
    stars INTEGER NOT NULL,
    releaseDate TEXT NOT NULL CHECK(releaseDate GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]')
);
CREATE INDEX idx_films_title ON Films(filmTitle);
CREATE INDEX idx_films_release_date ON Films(releaseDate);
```

### Mapping Rules Between SQLite and C#

| SQLite Column | Constraint / Description | C# Property & Type | Data Annotation |
| :--- | :--- | :--- | :--- |
| `filmID` | `INTEGER PRIMARY KEY AUTOINCREMENT` | `int FilmId` | `[Key]`, `[Column("filmID")]` |
| `filmTitle` | `TEXT NOT NULL <= 70 chars` | `string FilmTitle` | `[Required]`, `[MaxLength(70)]`, `[Column("filmTitle")]` |
| `filmCertificate`| `TEXT NOT NULL <= 5 chars` | `string FilmCertificate` | `[Required]`, `[MaxLength(5)]`, `[Column("filmCertificate")]` |
| `filmDescription`| `TEXT NOT NULL` | `string FilmDescription` | `[Required]`, `[Column("filmDescription")]` |
| `filmImage` | `TEXT NOT NULL <= 50 chars` | `string FilmImage` | `[Required]`, `[MaxLength(50)]`, `[Column("filmImage")]` |
| `filmPrice` | `NUMERIC NOT NULL >= 0` | `decimal FilmPrice` | `[Required]`, `[Range(0, double.MaxValue)]`, `[Column("filmPrice")]` |
| `stars` | `INTEGER NOT NULL (1 to 5)` | `int Stars` | `[Required]`, `[Range(1, 5)]`, `[Column("stars")]` |
| `releaseDate` | `TEXT NOT NULL GLOB 'YYYY-MM-DD'` | `DateOnly ReleaseDate` | `[Required]`, `[Column("releaseDate")]` |

> [!TIP]
> **Why use Data Annotations?**
> C# naming conventions use **PascalCase** (`FilmId`, `FilmTitle`), whereas the existing database table uses **camelCase** (`filmID`, `filmTitle`). By applying `[Column("filmTitle")]`, EF Core maps the property seamlessly without altering the existing database schema.
> In .NET 10, `DateOnly` automatically serializes to and from SQLite's `'yyyy-MM-dd'` string format, satisfying the SQLite `GLOB` CHECK constraint!

---

## 2. Create the `Film` Model

1. Create a `Models` folder in your project root:
   ```bash
   mkdir -p Models
   ```
2. Create `Models/Film.cs` with the following code:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace FilmApi.Models;

[Table("Films")]
public class Film
{
    [Key]
    [Column("filmID")]
    public int FilmId { get; set; }

    [Required]
    [MaxLength(70)]
    [Column("filmTitle")]
    public string FilmTitle { get; set; } = string.Empty;

    [Required]
    [MaxLength(5)]
    [Column("filmCertificate")]
    public string FilmCertificate { get; set; } = string.Empty;

    [Required]
    [Column("filmDescription")]
    public string FilmDescription { get; set; } = string.Empty;

    [Required]
    [MaxLength(50)]
    [Column("filmImage")]
    public string FilmImage { get; set; } = string.Empty;

    [Required]
    [Range(0, double.MaxValue, ErrorMessage = "Price must be non-negative")]
    [Column("filmPrice")]
    public decimal FilmPrice { get; set; }

    [Required]
    [Range(1, 5, ErrorMessage = "Star rating must be between 1 and 5")]
    [Column("stars")]
    public int Stars { get; set; }

    [Required]
    [Column("releaseDate")]
    public DateOnly ReleaseDate { get; set; }
}
```

### Code Explanation:
- `[Table("Films")]`: Directs EF Core to map this entity to the `Films` table rather than the default pluralized class name.
- `= string.Empty;`: Initializes non-nullable string properties to avoid compiler warnings under `<Nullable>enable</Nullable>`.
- `DateOnly`: Represents a pure calendar date without time or timezone offset, matching SQLite's `YYYY-MM-DD` text structure.

---

## 3. Create the Database Context (`FilmContext`)

An EF Core **DbContext** represents a session with the database, allowing you to query and save instances of your entities.

1. Create a `Data` folder or use the existing one. Since `Data/Films.db` is already present, place `FilmContext.cs` inside `Data/`:

```csharp
// Data/FilmContext.cs
using Microsoft.EntityFrameworkCore;
using FilmApi.Models;

namespace FilmApi.Data;

public class FilmContext : DbContext
{
    public FilmContext(DbContextOptions<FilmContext> options) : base(options)
    {
    }

    public DbSet<Film> Films => Set<Film>();
}
```

### Code Explanation:
- `DbContextOptions<FilmContext>`: Receives configuration settings (such as the database provider and connection string) from ASP.NET Core's Dependency Injection system.
- `DbSet<Film> Films => Set<Film>();`: Exposes the `Films` table as an EF Core queryable collection. The expression `Set<Film>()` is the recommended modern C# approach to avoid null warnings.

---

## 4. Configure the Connection String in `appsettings.json`

Open `appsettings.json` and add the `ConnectionStrings` section:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=Data/Films.db"
  }
}
```

> [!IMPORTANT]
> In SQLite, `Data Source=Data/Films.db` specifies a relative path from the working directory where the process is executed. If you run `dotnet run` from the project root, it resolves directly to `Data/Films.db`.

---

## 5. Register `FilmContext` in `Program.cs`

Open `Program.cs`. We will register the SQLite `FilmContext` with the Dependency Injection container.

In .NET 10, ASP.NET Core provides a clean helper method `AddSqlite<TContext>`:

```csharp
using Microsoft.EntityFrameworkCore;
using FilmApi.Data;

var builder = WebApplication.CreateBuilder(args);

// Register SQLite FilmContext with DI
builder.Services.AddSqlite<FilmContext>(
    builder.Configuration.GetConnectionString("DefaultConnection")
);

// Register OpenAPI services
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

// Placeholder route for testing
app.MapGet("/", () => "Film Minimal API is running!");

app.Run();
```

---

## 6. Verify with `dotnet build`

In your terminal, build the project to verify that there are no syntax or type errors:

```bash
dotnet build
```

Expected output:

```text
Build succeeded.
    0 Warning(s)
    0 Error(s)
```

---

## Summary

You have now:
- Mapped the SQLite `Films` table to the `Film` C# model with precise column and validation attributes.
- Created `FilmContext` inheriting from `DbContext`.
- Configured the SQLite connection string in `appsettings.json`.
- Registered `FilmContext` in ASP.NET Core's Dependency Injection container using `AddSqlite<FilmContext>`.

---

[⬅️ Step 01: Project Setup](01-project-setup.md) &bull; **Step 02: Database Model and DbContext** &bull; [Next: Step 03 - Reading Data (GET Endpoints) ➡️](03-reading-data-get-endpoints.md)
