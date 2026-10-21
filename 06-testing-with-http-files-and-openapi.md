# Step 06: Testing with .http Files and OpenAPI

[⬅️ Step 05: Route Groups](05-structuring-endpoints-and-route-groups.md) &bull; **Step 06: Testing with .http Files & OpenAPI** &bull; [Next: Step 07 ➡️](07-cheat-sheet-and-troubleshooting.md)

---

Testing your API during development is essential. Instead of leaving Visual Studio Code to use external desktop applications like Postman or Insomnia, modern .NET and VS Code support **`.http` files**—version-controllable, plain-text HTTP test scripts you can run with a single click.

In this step, you will build a comprehensive `films.http` file to test all endpoints and explore .NET 10's built-in **OpenAPI** documentation with **Scalar**.

---

## 1. What is an `.http` File?

An `.http` file allows you to define HTTP requests using plain text directly within your project repository. When opened in VS Code with the **REST Client** extension or **C# Dev Kit**, a clickable **"Send Request"** button appears above each HTTP request.

### Key Syntax Rules:
- Variables are defined with `@variableName = value`.
- Requests are separated by three hash symbols: `###`.
- Comments start with `#` or `//`.

---

## 2. Create `films.http`

Create a file named `films.http` in the root of your project:

```http
@baseUrl = http://localhost:5000

### 1. Get All Films
GET {{baseUrl}}/api/films
Accept: application/json

### 2. Search Films by Title Keyword (e.g. 'Shawshank')
GET {{baseUrl}}/api/films?title=Shawshank
Accept: application/json

### 3. Filter Films by Minimum Stars and Certificate
GET {{baseUrl}}/api/films?minStars=5&certificate=15
Accept: application/json

### 4. Get a Single Film by ID
GET {{baseUrl}}/api/films/1
Accept: application/json

### 5. Get a Non-Existent Film (Should Return 404 Not Found)
GET {{baseUrl}}/api/films/99999
Accept: application/json

### 6. Insert a New Film (POST)
POST {{baseUrl}}/api/films
Content-Type: application/json

{
  "filmTitle": "Interstellar",
  "filmCertificate": "12A",
  "filmDescription": "A team of explorers travel through a wormhole in space in an attempt to ensure humanity's survival.",
  "filmImage": "interstellar.jpg",
  "filmPrice": 4.99,
  "stars": 5,
  "releaseDate": "2014-11-07"
}

### 7. Update the Film Details (PUT)
# Replace 69 with the ID returned from the POST request above
PUT {{baseUrl}}/api/films/69
Content-Type: application/json

{
  "filmTitle": "Interstellar (IMAX Edition)",
  "filmCertificate": "12A",
  "filmDescription": "Special IMAX re-release with expanded space sequences.",
  "filmImage": "interstellar_imax.jpg",
  "filmPrice": 5.99,
  "stars": 5,
  "releaseDate": "2014-11-07"
}

### 8. Attempt an Invalid Insert (Validation Failure - Price < 0)
POST {{baseUrl}}/api/films
Content-Type: application/json

{
  "filmTitle": "Invalid Free Movie",
  "filmCertificate": "PG",
  "filmDescription": "Testing negative price validation.",
  "filmImage": "sample.jpg",
  "filmPrice": -10.00,
  "stars": 4,
  "releaseDate": "2024-01-01"
}

### 9. Delete the Film (DELETE)
DELETE {{baseUrl}}/api/films/69

### 10. Verify Deletion (Should Return 404 Not Found)
GET {{baseUrl}}/api/films/69
Accept: application/json
```

---

## 3. Running Requests in Visual Studio Code

1. Start your API in the terminal:
   ```bash
   dotnet watch
   ```
2. Open `films.http` in VS Code.
3. Click the **"Send Request"** link directly above any request block.
4. A new split-pane window will appear on the right showing the raw HTTP response, headers, timing, and JSON payload!

---

## 4. Interactive API Documentation with Scalar in .NET 10

In .NET 10, ASP.NET Core includes native OpenAPI document generation via `Microsoft.AspNetCore.OpenApi`. When running in the development environment, navigating to:

```text
http://localhost:5000/openapi/v1.json
```

displays the full OpenAPI v3 specification describing all your endpoints, routes, models, and schemas in JSON format.

### Adding an Interactive UI (Scalar)

To view and test your API through a beautiful, modern browser interface, you can add **Scalar**:

1. Install the Scalar package in your terminal:
   ```bash
   dotnet add package Scalar.AspNetCore --version 2.*
   ```

2. Update `Program.cs` to enable the Scalar UI:

```csharp
using FilmApi.Data;
using FilmApi.Endpoints;
using Scalar.AspNetCore; // <-- Add this namespace

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSqlite<FilmContext>(
    builder.Configuration.GetConnectionString("DefaultConnection")
);

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference(); // <-- Adds interactive UI at /scalar/v1
}

app.UseHttpsRedirection();

app.MapGroup("/api/films")
   .MapFilmEndpoints()
   .WithTags("Films");

app.Run();
```

3. Run the application and open your browser to:
   ```text
   http://localhost:5000/scalar/v1
   ```

You will see an interactive API documentation dashboard where you can:
- View all endpoints grouped under **Films**.
- Inspect parameter schemas and response models.
- Click **"Test Request"** to trigger queries directly from the browser!

---

## Summary

In this step, you:
- Created a version-controlled `films.http` test file covering all CRUD and edge cases.
- Tested endpoints directly inside VS Code without needing external GUI tools.
- Discovered how .NET 10 generates OpenAPI specifications automatically from `TypedResults`.
- Configured interactive API documentation using Scalar.

---

[⬅️ Step 05: Route Groups](05-structuring-endpoints-and-route-groups.md) &bull; **Step 06: Testing with .http Files & OpenAPI** &bull; [Next: Step 07 - Cheat Sheet and Troubleshooting ➡️](07-cheat-sheet-and-troubleshooting.md)
