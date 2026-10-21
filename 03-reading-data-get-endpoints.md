# Step 03: Reading Data (GET Endpoints)

[⬅️ Step 02: Model & Context](02-database-model-and-context.md) &bull; **Step 03: Reading Data (GET Endpoints)** &bull; [Next: Step 04 ➡️](04-modifying-data-post-put-delete.md)

---

In this step, you will implement HTTP **GET** endpoints to query and retrieve film data from the SQLite database. You will learn how Minimal APIs automatically inject dependencies, how to execute asynchronous LINQ queries, how to use route constraints, and how to filter data with query string parameters.

---

## 1. Minimal API Route Handlers & Dependency Injection

In ASP.NET Core Minimal APIs, endpoints are mapped using methods on the `WebApplication` instance, such as `app.MapGet(pattern, handler)`.

Notice how cleanly Dependency Injection works:

```csharp
app.MapGet("/api/films", async (FilmContext db) =>
{
    return await db.Films.ToListAsync();
});
```

Because `FilmContext` was registered with `builder.Services.AddSqlite<FilmContext>(...)`, the framework automatically resolves and injects an instance of `FilmContext` into the lambda parameters for every incoming HTTP request.

### `TypedResults` vs `Results`

In modern .NET:
- `Results.Ok(...)` returns `IResult` (loosely typed object).
- `TypedResults.Ok(...)` returns `Ok<T>` (strongly typed).

> [!TIP]
> **Why prefer `TypedResults` in .NET 10?**
> `TypedResults` provides compile-time type safety, enables seamless unit testing without mocking `HttpContext`, and automatically emits accurate OpenAPI response metadata for documentation tools.

---

## 2. Endpoint 1: Get All Films

Let's implement an endpoint that returns all films from `Data/Films.db`.

```csharp
app.MapGet("/api/films", async (FilmContext db) =>
{
    var films = await db.Films.ToListAsync();
    return TypedResults.Ok(films);
});
```

### Why Asynchronous (`async` / `await`)?
Database queries involve input/output (I/O) wait times. Using `await db.Films.ToListAsync()` releases the server thread to handle other incoming requests while SQLite reads the file from disk, ensuring high performance and scalability.

---

## 3. Endpoint 2: Get Film by ID with Route Constraints

To retrieve a specific film by its primary key, we define a route parameter `{id:int}`:

```csharp
app.MapGet("/api/films/{id:int}", async Task<Results<Ok<Film>, NotFound>> (int id, FilmContext db) =>
{
    var film = await db.Films.FindAsync(id);

    return film is not null
        ? TypedResults.Ok(film)
        : TypedResults.NotFound();
});
```

### Key Elements:
1. **Route Constraint `{id:int}`**: The `:int` constraint ensures this endpoint only triggers if the URL segment is a valid integer (e.g. `/api/films/1`). If someone requests `/api/films/foo`, ASP.NET Core immediately rejects it with a `404 Not Found` without querying SQLite.
2. **`FindAsync(id)`**: EF Core looks up the entity by its primary key (`filmID`). It checks local context memory first before issuing a SQL query.
3. **`Results<Ok<Film>, NotFound>`**: A union return type that informs both the C# compiler and OpenAPI document generators that this endpoint returns either `200 OK` (with a `Film`) or `404 Not Found`.

---

## 4. Endpoint 3: Adding Search and Filtering with Query Parameters

In a real-world API, clients frequently want to filter records—for example, searching for a title, filtering by minimum star rating, or filtering by BBFC certificate (`PG`, `12`, `15`, etc.).

Query string parameters are declared as optional parameters in the lambda handler:

```csharp
app.MapGet("/api/films", async (
    string? title,
    int? minStars,
    string? certificate,
    FilmContext db) =>
{
    // Start with IQueryable (query not yet executed in database)
    var query = db.Films.AsQueryable();

    // 1. Filter by title keyword (case-insensitive)
    if (!string.IsNullOrWhiteSpace(title))
    {
        query = query.Where(f => EF.Functions.Like(f.FilmTitle, $"%{title}%"));
    }

    // 2. Filter by minimum stars
    if (minStars.HasValue)
    {
        query = query.Where(f => f.Stars >= minStars.Value);
    }

    // 3. Filter by certificate
    if (!string.IsNullOrWhiteSpace(certificate))
    {
        query = query.Where(f => f.FilmCertificate == certificate);
    }

    // Execute the combined SQL query asynchronously
    var films = await query.ToListAsync();
    return TypedResults.Ok(films);
});
```

### How `IQueryable` Works with SQLite
When you write `query = query.Where(...)`, EF Core **does not** load all films into memory and filter them in C#. Instead, it builds an optimized SQL statement:

```sql
SELECT "f"."filmID", "f"."filmCertificate", "f"."filmDescription", "f"."filmImage", "f"."filmPrice", "f"."filmTitle", "f"."releaseDate", "f"."stars"
FROM "Films" AS "f"
WHERE "f"."filmTitle" LIKE '%Paradiso%' AND "f"."stars" >= 4;
```

The database engine executes the filter, returning only the matching rows over the wire.

---

## 5. Complete `Program.cs` for GET Endpoints

Replace the contents of `Program.cs` with the updated code:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.EntityFrameworkCore;
using FilmApi.Data;
using FilmApi.Models;

var builder = WebApplication.CreateBuilder(args);

// Register SQLite FilmContext
builder.Services.AddSqlite<FilmContext>(
    builder.Configuration.GetConnectionString("DefaultConnection")
);

// Register OpenAPI
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

// GET: All films with optional filtering
app.MapGet("/api/films", async (
    string? title,
    int? minStars,
    string? certificate,
    FilmContext db) =>
{
    var query = db.Films.AsQueryable();

    if (!string.IsNullOrWhiteSpace(title))
    {
        query = query.Where(f => EF.Functions.Like(f.FilmTitle, $"%{title}%"));
    }

    if (minStars.HasValue)
    {
        query = query.Where(f => f.Stars >= minStars.Value);
    }

    if (!string.IsNullOrWhiteSpace(certificate))
    {
        query = query.Where(f => f.FilmCertificate == certificate);
    }

    var results = await query.ToListAsync();
    return TypedResults.Ok(results);
});

// GET: Single film by ID
app.MapGet("/api/films/{id:int}", async Task<Results<Ok<Film>, NotFound>> (int id, FilmContext db) =>
{
    var film = await db.Films.FindAsync(id);

    return film is not null
        ? TypedResults.Ok(film)
        : TypedResults.NotFound();
});

app.Run();
```

---

## 6. Testing Your GET Endpoints

Run the application from your terminal:

```bash
dotnet run
```

### Test 1: Retrieve All Films
Open a separate terminal window and run:

```bash
curl -s http://localhost:5000/api/films | head -n 30
```

You will receive an array of JSON film objects:

```json
[
  {
    "filmId": 1,
    "filmTitle": "Cinema Paradiso",
    "filmCertificate": "15",
    "filmDescription": "A famous film director returns home to a Sicilian village...",
    "filmImage": "cinemaParadiso.jpg",
    "filmPrice": 0.99,
    "stars": 5,
    "releaseDate": "1988-02-23"
  }
]
```

### Test 2: Retrieve a Specific Film by ID
```bash
curl -i http://localhost:5000/api/films/1
```

HTTP Response:
```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"filmId":1,"filmTitle":"Cinema Paradiso",...}
```

### Test 3: Test a Non-Existent ID (404 Not Found)
```bash
curl -i http://localhost:5000/api/films/9999
```

HTTP Response:
```text
HTTP/1.1 404 Not Found
```

### Test 4: Search with Query Parameters
```bash
# Search for films with 'Shawshank' in the title
curl -s "http://localhost:5000/api/films?title=Shawshank"

# Filter 5-star films certified '15'
curl -s "http://localhost:5000/api/films?minStars=5&certificate=15"
```

---

## Summary

In this step, you:
- Built a collection endpoint (`/api/films`) using asynchronous EF Core queries.
- Created an item endpoint (`/api/films/{id}`) using route constraints and `TypedResults`.
- Implemented multi-criteria filtering using query parameters and deferred SQL execution with `IQueryable`.
- Verified HTTP status codes (`200 OK`, `404 Not Found`).

---

[⬅️ Step 02: Model & Context](02-database-model-and-context.md) &bull; **Step 03: Reading Data (GET Endpoints)** &bull; [Next: Step 04 - Modifying Data (POST, PUT, DELETE) ➡️](04-modifying-data-post-put-delete.md)
