# Step 01: Project Setup and CLI

[🏠 Overview](README.md) &bull; **Step 01: Project Setup** &bull; [Next: Step 02 ➡️](02-database-model-and-context.md)

---

In this step, you will create a new ASP.NET Core Minimal API project targeting **.NET 10** using the .NET CLI within Visual Studio Code, add the necessary Entity Framework Core SQLite packages, and verify that the local web server runs.

---

## 1. Open the Project in Visual Studio Code

1. Launch **Visual Studio Code**.
2. Open the repository root directory (`dotnet-minimal-api`).
3. You should see the `Data/` folder containing `Films.db`.

To open the integrated terminal in VS Code:
- Press <kbd>Ctrl</kbd> + <kbd>`</kbd> (or <kbd>Cmd</kbd> + <kbd>`</kbd> on macOS).
- Alternatively, select **Terminal** &rarr; **New Terminal** from the top menu.

Confirm you have the .NET 10 SDK installed by running:

```bash
dotnet --version
```

You should see an output beginning with `10.` (e.g. `10.0.301`).

---

## 2. Initialize the Minimal API Project

From the project root directory, run the following command to scaffold a new ASP.NET Core Web API:

```bash
dotnet new webapi
```

> [!NOTE]
> In .NET 10, the `webapi` template uses **Minimal APIs** and **top-level statements** by default. It also pre-configures OpenAPI documentation support.

### Understanding the Generated Files

Once the command completes, notice the newly created files alongside your `Data/` directory:

```text
dotnet-minimal-api/
├── Data/
│   ├── Films.db              <-- Existing SQLite database
│   └── Films.sqbpro          <-- DB Browser project file
├── Properties/
│   └── launchSettings.json   <-- Local development URLs and port bindings
├── appsettings.json          <-- Application configuration
├── appsettings.Development.json
├── Program.cs                <-- Application entry point and Minimal API endpoints
└── dotnet-minimal-api.csproj <-- Project manifest and NuGet package references
```

Let's inspect `dotnet-minimal-api.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.*" />
  </ItemGroup>

</Project>
```

Key settings in this file:
- `<TargetFramework>net10.0</TargetFramework>`: Configures the project to run on .NET 10.
- `<Nullable>enable</Nullable>`: Enables C# nullable reference types to detect potential null reference exceptions at compile time.
- `<ImplicitUsings>enable</ImplicitUsings>`: Automatically imports common namespaces (like `System`, `System.Collections.Generic`, `Microsoft.AspNetCore.Builder`, etc.) so you do not need repetitive `using` statements at the top of each file.

---

## 3. Install the Entity Framework Core SQLite Package

To interact with our SQLite database, we need the official Microsoft SQLite provider for Entity Framework Core.

Run the following command in your terminal:

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

This package pulls in:
1. `Microsoft.EntityFrameworkCore`: The core ORM engine.
2. `Microsoft.Data.Sqlite`: The high-performance low-level SQLite driver.
3. EF Core relational query translation providers for SQLite.

You can verify that the package was added by checking `dotnet-minimal-api.csproj`. It will now include:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.*" />
</ItemGroup>
```

---

## 4. Run and Test the Starter Application

The scaffolded template includes a default `/weatherforecast` sample endpoint in `Program.cs`. Let's test running the project.

In your terminal, execute:

```bash
dotnet run
```

You will see output similar to:

```text
Building...
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
```

> [!TIP]
> The exact port number may vary based on `Properties/launchSettings.json` (e.g. `http://localhost:5031` or `http://localhost:5000`).

Open your browser or run `curl` in a separate terminal:

```bash
curl http://localhost:5000/weatherforecast
```

You will receive a JSON response containing generated forecast items:

```json
[
  {"date":"2026-10-22","temperatureC":12,"summary":"Cool","temperatureF":53},
  {"date":"2026-10-23","temperatureC":24,"summary":"Warm","temperatureF":75}
]
```

Press <kbd>Ctrl</kbd> + <kbd>C</kbd> in your terminal to stop the web server.

---

## 5. Development with `dotnet watch` (Hot Reload)

Instead of manually stopping and re-running `dotnet run` every time you modify C# code, you can use `dotnet watch`:

```bash
dotnet watch
```

`dotnet watch` monitors source files for edits and automatically compiles and hot-reloads changes into the running application without restarting the entire process.

---

## Summary

You have successfully:
- Initialized a .NET 10 Minimal API project in VS Code.
- Inspected the project configuration file (`.csproj`).
- Installed the `Microsoft.EntityFrameworkCore.Sqlite` NuGet package.
- Verified that the ASP.NET Core server runs and responds to HTTP requests.

---

[🏠 Overview](README.md) &bull; **Step 01: Project Setup** &bull; [Next: Step 02 - Database Model and DbContext ➡️](02-database-model-and-context.md)
