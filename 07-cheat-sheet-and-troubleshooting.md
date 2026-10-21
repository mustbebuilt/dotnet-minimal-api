# Step 07: Cheat Sheet and Troubleshooting

[⬅️ Step 06: Testing & OpenAPI](06-testing-with-http-files-and-openapi.md) &bull; **Step 07: Cheat Sheet & Troubleshooting** &bull; [🏠 Overview](README.md)

---

This reference guide summarizes key Minimal API syntax, HTTP return helpers, and solutions to common pitfalls when working with SQLite and Entity Framework Core in .NET 10.

---

## 🚀 Minimal API Quick Reference

### 1. Route Mapping Methods

```csharp
app.MapGet("/path", handler);       // Retrieve data (Read)
app.MapPost("/path", handler);      // Create new resource (Create)
app.MapPut("/path/{id}", handler);  // Replace/Update resource (Update)
app.MapDelete("/path/{id}", handler); // Delete resource (Delete)

// Route Grouping
var group = app.MapGroup("/api/films")
               .WithTags("Films");
```

### 2. Standard HTTP Response Helpers (`TypedResults`)

| Method | HTTP Status | Use Case | Example |
| :--- | :--- | :--- | :--- |
| `TypedResults.Ok(data)` | `200 OK` | Successful retrieval or general response | `return TypedResults.Ok(films);` |
| `TypedResults.Created(uri, data)` | `201 Created` | Successful insertion with `Location` header | `return TypedResults.Created($"/api/films/{id}", film);` |
| `TypedResults.NoContent()` | `204 No Content` | Successful update or deletion with empty body | `return TypedResults.NoContent();` |
| `TypedResults.NotFound()` | `404 Not Found` | Requested ID does not exist | `return TypedResults.NotFound();` |
| `TypedResults.BadRequest(msg)` | `400 Bad Request` | Client sent invalid data or violated schema | `return TypedResults.BadRequest("Invalid price");` |

### 3. Asynchronous EF Core Queries

```csharp
// Load all rows into memory
var all = await db.Films.ToListAsync();

// Lookup by Primary Key
var item = await db.Films.FindAsync(id);

// Filtering (WHERE)
var filtered = await db.Films
    .Where(f => f.Stars >= 4)
    .ToListAsync();

// Case-insensitive SQL LIKE
var search = await db.Films
    .Where(f => EF.Functions.Like(f.FilmTitle, "%matrix%"))
    .ToListAsync();

// Sorting and Paging
var page = await db.Films
    .OrderByDescending(f => f.ReleaseDate)
    .Skip(10)
    .Take(10)
    .ToListAsync();
```

---

## 🛠️ Common SQLite Pitfalls & Troubleshooting

### 1. SQLite Database Locked (`SQLite Error 5: 'database is locked'`)

> [!WARNING]
> SQLite is a file-based database that locks the `.db` file during write operations.

**Cause:**
- You have the database opened in an external desktop tool (such as **DB Browser for SQLite**) with unsaved or pending transactions.
- Multiple threads or processes are attempting to write concurrently without WAL (Write-Ahead Logging) enabled.

**Solution:**
1. In DB Browser for SQLite, click **"Write Changes"** or close the database connection.
2. Ensure you `await db.SaveChangesAsync()` and dispose contexts properly through ASP.NET Core's scoped Dependency Injection lifecycle.

---

### 2. Empty Database Created / Data Not Found

**Symptom:**
When calling `GET /api/films`, you receive an empty array `[]` even though you know data exists in `Data/Films.db`.

**Cause:**
In SQLite, if the database file specified in the connection string does not exist, SQLite **silently creates a new empty database file** at that path. If you launch the application from a different folder (e.g. from inside a subfolder or via a test runner), `Data Source=Data/Films.db` resolves relative to that subfolder, creating a new zero-byte `Films.db`.

**Solution:**
Always execute `dotnet run` or `dotnet watch` from the project root where `Data/Films.db` is located. You can also inspect the current directory in code:
```csharp
Console.WriteLine($"Current Directory: {Environment.CurrentDirectory}");
```

---

### 3. CHECK Constraint Violations (`SQLite Error 19: 'CHECK constraint failed'`)

**Symptom:**
Calling `POST` or `PUT` throws a `Microsoft.Data.Sqlite.SqliteException: SQLite Error 19: 'CHECK constraint failed: Films'`.

**Cause:**
The incoming payload violated one of the table constraints defined in the schema:
1. `filmTitle`: Length must be `<= 70` and non-empty.
2. `filmCertificate`: Length must be `<= 5` and non-empty.
3. `filmImage`: Length must be `<= 50` and non-empty.
4. `filmPrice`: Must be `>= 0`.
5. `releaseDate`: Must match `GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'` (ISO format `YYYY-MM-DD`).

**Solution:**
Always validate inputs in your endpoint handler before calling `await db.SaveChangesAsync()`, returning `TypedResults.BadRequest(...)` when rules are violated.

---

### 4. Enabling CORS for Frontend Clients (React, Vue, Svelte, Angular)

If you build a client-side frontend that calls this API from `http://localhost:3000` or `http://localhost:5173`, the browser will block the requests due to the **Same-Origin Policy**.

To enable Cross-Origin Resource Sharing (CORS):

1. Register CORS in `Program.cs`:
   ```csharp
   builder.Services.AddCors(options =>
   {
       options.AddPolicy("AllowAll", policy =>
       {
           policy.AllowAnyOrigin()
                 .AllowAnyMethod()
                 .AllowAnyHeader();
       });
   });
   ```

2. Enable CORS middleware before endpoint mapping:
   ```csharp
   app.UseCors("AllowAll");
   ```

---

### 5. C# Nullable Reference Warnings (`CS8618`)

In .NET 10, `<Nullable>enable</Nullable>` is enabled by default in `.csproj`. If you declare a string property without initializing it:

```csharp
// Generates compiler warning CS8618
public string FilmTitle { get; set; }
```

**Solution:**
Initialize all non-nullable strings with `= string.Empty;`:

```csharp
public string FilmTitle { get; set; } = string.Empty;
```

---

## 🎓 Congratulations!

You have completed the guide to building a Minimal API with .NET 10, EF Core, and SQLite. You now understand:
- How to structure and configure a Minimal API from scratch.
- How to map existing SQLite databases and handle constraints.
- How to write clean, asynchronous CRUD endpoints with route grouping.
- How to test endpoints with `.http` files and document them with OpenAPI.

---

[🏠 Return to Overview](README.md) &bull; [Step 01](01-project-setup.md) &bull; [Step 02](02-database-model-and-context.md) &bull; [Step 03](03-reading-data-get-endpoints.md) &bull; [Step 04](04-modifying-data-post-put-delete.md) &bull; [Step 05](05-structuring-endpoints-and-route-groups.md) &bull; [Step 06](06-testing-with-http-files-and-openapi.md)
