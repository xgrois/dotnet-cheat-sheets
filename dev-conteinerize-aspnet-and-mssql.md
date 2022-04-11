# How to conteinerize both ASP .NET Web API and MS SQL Server for development
Follow the [dev-conteinerize-aspnet.md](dev-conteinerize-aspnet.md) to be able to run your API with docker-compose.

Once that is working, you add to your API the data layer to interact with you database.
For ilustration purposes, here is the `Program.cs` file with everything you need
```
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();


var server = builder.Configuration["DB_SERVER"];
var port = builder.Configuration["DB_PORT"];
var user = builder.Configuration["DB_USER"];
var password = builder.Configuration["DB_PASS"]; 
var database = builder.Configuration["DB_NAME"];
var connectionString = $"Server={server},{port};Initial Catalog={database};User ID={user};Password={password};";

//var connectionString = @"Server=db;Database=master;User=sa;Password=Your_password123;";

builder.Services.AddSqlServer<AppDbContext>(connectionString);

var app = builder.Build();

// Configure the HTTP request pipeline.
await EnsureDb(app.Services, app.Logger);

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Endpoints
app.MapGet("/books", async (AppDbContext db) => await db.Books.ToListAsync())
    .WithName("GetAllBooks");

app.Run();

// Db stuff
async Task EnsureDb(IServiceProvider services, ILogger logger)
{
    logger.LogInformation("Ensuring database exists and is up to date at connection string '{connectionString}'", connectionString);
 
    using var db = services.CreateScope().ServiceProvider.GetRequiredService<AppDbContext>();

    // If migration files do not exist,
    //  - it will still create the DB but just with a single EFMigrationsHistory Table
    //  - you can always run "dotnet ef migrations add Init" later
    //      - if you did it, MigrateAsync() will update the existing "dummy" DB with your Tables 
    // If migration files exist,
    //  - it will apply migrations to update DB model(s)
    //      - this means the DB is created with all your Tables from scratch
    //      - or do nothing if that migration file was already applied before
    await db.Database.MigrateAsync();
}

public record Book(int Id, string Title);

public class BookEntityTypeConfiguration : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> builder)
    {
        // Constraints
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Title).HasMaxLength(50);

        // Initial data
        builder.HasData(
            new Book(1,"Roma Soy Yo"),
            new Book(2,"No Me Cuentes Cuentos"),
            new Book(3,"El Mentalista"),
            new Book(4,"Nunca"),
            new Book(5,"Matar Al Rey")
            );
    }
}

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> dbContextOptions) : base(dbContextOptions)
    {
    }
    public DbSet<Book> Books => Set<Book>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Instead of defining here all property tables and fields we just read all
        // from each IEntityTypeConfiguration<T> derived class
        // This way is crearer and prevents this method to grow
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}


```
> Note: this API is using EF Core Code First approach. **YOU NEED** to manually create migration files (`dotnet ef migrations add Init`) before running docker-compose commands.


Then, modify docker-compose.yml file to launch both your API and the SQL server in containers.
```csharp
version: "3.9"
services:
    api:
        build: .
        ports:
            - "8000:80"
            - "8001:443"
        environment:
            ASPNETCORE_URLS: "https://+;http://+"
            ASPNETCORE_HTTPS_PORT: "8001"
            ASPNETCORE_ENVIRONMENT: Development
            DB_SERVER: "db"
            DB_PORT: "1433"
            DB_USER: "sa"
            DB_PASS: "Your_password123"
            DB_NAME: "BooksManagementDb"
        volumes:
            - ${APPDATA}\microsoft\UserSecrets\:/root/.microsoft/usersecrets
            - ${USERPROFILE}\.aspnet\https:/root/.aspnet/https/
        depends_on:
            - db
    db:
        image: "mcr.microsoft.com/mssql/server:2019-latest"
        environment:
            SA_PASSWORD: "Your_password123"
            ACCEPT_EULA: "Y"
        ports:
            - "1433:1433"
```

Build and run:

```console
> docker-compose build
> docker-compose up
```

After the application starts, navigate to `http://localhost:8000/books` in your web browser. 
Note both HTTP and HTTPS endpoins are functional. 
If you use the HTTP endpoint in the browser you will be redirected to HTTPS. 
You can try both endpoints with Postman/Insomnia.

You can directly access the SQL Server instance in the container with Microsoft SQL Server Management Studio (MSSMS):

-   Server Type: Database Engine
-   Server Name: _localhost,1433_
-   Authentication: SQL Server Authentication
    -   Login: sa
    -   Password: Your_password123

