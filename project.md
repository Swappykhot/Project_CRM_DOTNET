Project overview

Name: Email-to-Case (Local) — ASP.NET Core demo
Main idea: Accept incoming email-like JSON via a REST API, queue it, process it in a background worker that creates Case records in a local SQLite DB. The design demonstrates decoupling (API vs processing), background processing, persistence (EF Core), and logging (Serilog).

High-level architecture (text diagram)
[Email Client or Gateway]
        |
  HTTP POST /api/email/receive
        |
   [EmailController] ----> EmailQueue (in-memory concurrent queue)
        |
   [EmailProcessorService] (BackgroundService)  <-- runs continuously
        |
   [CrmDbContext] (EF Core -> SQLite crm.db)
        |
   Cases table persisted


Additionally:

CasesController exposes GET /api/cases, GET /api/cases/{id}, DELETE /api/cases/{id} for read/delete.

Serilog logs to console and logs/emailtocase_.log.

Runtime flow (step-by-step, what happens end-to-end)

Client sends email JSON
HTTP POST http://localhost:5000/api/email/receive (or https://localhost:5001) with body:

{ "from":"customer@example.com", "subject":"Payment not reflecting", "body":"I paid 2000 but CRM shows pending." }


EmailController.Receive runs

Validates basic fields (non-null).

Calls EmailQueue.Enqueue(email) to push the Email DTO into an in-memory ConcurrentQueue.

Returns 200 OK with { message: "Email queued for processing" }.

This keeps the HTTP request fast (non-blocking).

Email sits in EmailQueue until the background service picks it up.

EmailProcessorService.ExecuteAsync loop

Background service continuously checks EmailQueue.TryDequeue.

If an item is dequeued, it opens a scoped CrmDbContext (via IServiceScopeFactory) and processes the email:

Performs a basic duplicate check: looks for any Case with the same CustomerEmail and Title created within last 5 minutes.

If duplicate → log and skip.

If not duplicate → create a new Case entity (Title, Description, CustomerEmail, Status, CreatedAt) and call db.SaveChangesAsync().

Logs success/failure via Serilog/ILogger.

If queue empty → await Task.Delay(1000) and re-check.

Case persisted in crm.db (SQLite file). You can query Cases table, call GET /api/cases or inspect with a SQLite viewer.

Read & Cleanup APIs

GET /api/cases returns all cases ordered by CreatedAt desc.

GET /api/cases/{id} returns specific case or 404.

DELETE /api/cases/{id} removes a case and returns 204 No Content (or 404 if absent).

File-by-file explanation (what each file does and important lines)

Project root files: EmailToCase.csproj, Program.cs, appsettings.json, README.md, Dockerfile, .gitignore.

EmailToCase.csproj

Standard .NET SDK web project file.

Includes package references: Microsoft.EntityFrameworkCore.Sqlite, Microsoft.EntityFrameworkCore.Design, and Serilog packages.

Explains runtime target .NET 7.

Why important: dependency manifest; controls which NuGet packages are available.

Program.cs

Key responsibilities:

Configure logging with Serilog:

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("logs/emailtocase_.log", rollingInterval: RollingInterval.Day)
    .CreateLogger();
builder.Host.UseSerilog();


Register services:

builder.Services.AddDbContext<CrmDbContext>(...) — configures EF Core with SQLite.

builder.Services.AddSingleton<EmailQueue>(); — single instance queue used by both controller and hosted service.

builder.Services.AddHostedService<EmailProcessorService>(); — registers the background worker.

builder.Services.AddControllers(), AddSwaggerGen(), AddCors(...).

Ensure DB exists and seed one case on startup:

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<CrmDbContext>();
    db.Database.EnsureCreated();
    if (!db.Cases.Any()) { /* add seed case */ }
}


Map controllers and run the host:

app.MapControllers();
app.Run();


Why important: shows DI, lifetime management, environment-aware behavior (Swagger only in dev), and where the app starts.

Controllers/EmailController.cs

Key endpoints:

POST api/email/receive — receives an EmailDto and enqueues it:

_queue.Enqueue(email);
_logger.LogInformation("Email queued from {from} subject {subject}", email.From, email.Subject);
return Ok(new { message = "Email queued for processing" });


GET api/email/health — basic health check returning status ok.

Why important: demonstrates small, fast controllers and pushes heavy work to background processing.

Controllers/CasesController.cs

Endpoints:

GET api/cases — returns all cases:

var list = await _db.Cases.OrderByDescending(c => c.CreatedAt).ToListAsync();
return Ok(list);


GET api/cases/{id} — returns one or NotFound().

DELETE api/cases/{id} — removes the case and saves changes.

Why important: show read/delete endpoints useful for UI and for the demo to verify results.

Services/EmailQueue.cs

Implementation:

private readonly ConcurrentQueue<EmailDto> _queue = new();
public void Enqueue(EmailDto email) => _queue.Enqueue(email);
public bool TryDequeue(out EmailDto? email) => _queue.TryDequeue(out email);


Why important: simple thread-safe in-memory queue. Good to explain decoupling: producer (controller) vs consumer (background worker).

Services/EmailProcessorService.cs

Implements BackgroundService:

Runs an infinite loop until cancellation requested.

For each dequeued email:

Creates DI scope:

using var scope = _scopeFactory.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<CrmDbContext>();


Checks duplicates (within 5 mins) via LINQ:

var exists = db.Cases.Any(c => c.CustomerEmail == email.From && c.Title == email.Subject && (DateTime.UtcNow - c.CreatedAt).TotalMinutes < 5);


If not duplicate, create new Case entity, db.Cases.Add(@case); await db.SaveChangesAsync(stoppingToken);

Handles exceptions and logs errors; if queue empty, await Task.Delay(1000, stoppingToken);.

Why important:

Shows how to implement background processing in ASP.NET Core.

Shows scope creation so DB context doesn't live forever (avoids memory leaks).

Demonstrates simple deduplication and error handling.

Data/CrmDbContext.cs

EF Core DbContext:

DbSet<Case> Cases

OnModelCreating config for Case (Title required and max length 500).

Why important: the EF Core mapping and DB context used throughout.

Models/Case.cs

Entity fields:

public int Id { get; set; }
public string Title { get; set; } = string.Empty;
public string Description { get; set; } = string.Empty;
public string CustomerEmail { get; set; } = string.Empty;
public string Status { get; set; } = "Open";
public DateTime CreatedAt { get; set; } = DateTime.UtcNow;


Why important: simple normalized Case model you can expand (priority, assignedTo, SLA fields).

DTOs/EmailDto.cs

Input DTO for incoming email:

public string From { get; set; } = string.Empty;
public string Subject { get; set; } = string.Empty;
public string Body { get; set; } = string.Empty;


Why important: DTO keeps API surface stable and separate from DB model.

appsettings.json

Contains:

"ConnectionStrings": { "DefaultConnection": "Data Source=crm.db" }


Why important: shows DB used is SQLite file crm.db. For production you'd change to SQL Server/Postgres and use secrets.

README.md

Contains run instructions, endpoints, explanation, interview notes. Use it when demonstrating.

Dockerfile

Simple publish container image for running in Docker.

Database & storage

DB: crm.db (SQLite). Located in the app working directory after run.

Table: Cases with fields from Case model.

To inspect: use DB Browser for SQLite or VS Code’s SQLite extension.

Logging

Configured via Serilog:

Console output (visible in terminal).

Rolling file: logs/emailtocase_.log (daily).

Logs will include info on enqueued emails, created cases, duplicates, and processing errors.

Example HTTP calls & responses

Post an email
Request:

POST /api/email/receive
Content-Type: application/json
{
  "from":"customer@example.com",
  "subject":"Payment not reflecting",
  "body":"I paid 2000 but CRM shows pending."
}


Response: 200 OK

{ "message":"Email queued for processing" }


List cases

GET /api/cases


Response: 200 OK — array of Cases (each has Id, Title, Description, CustomerEmail, Status, CreatedAt)

Get single case

GET /api/cases/1


Response: 200 OK (Case JSON) or 404 Not Found if it doesn’t exist.

Delete

DELETE /api/cases/1


Response: 204 No Content if deleted, 404 if not found.

Concurrency, safety, and limitations

In-memory queue: Non-durable. If the application process dies, queued items are lost. Good for demos; not for production.

Duplicate detection: Basic (Title+Email within 5 minutes). Explain that production requires stronger idempotency keys or queue built-in dedup.

DB concurrency: SQLite supports concurrent reads but limited write concurrency. For production use SQL Server/Postgres.

Background worker scale: You can run multiple instances; with an in-memory queue only one instance has the queue. For multi-instance scaling use durable queues (Azure Service Bus, RabbitMQ, etc.).

Error handling: The background worker catches and logs exceptions and retries after short delays. For production implement poison message handling and retry policies.

Troubleshooting (common issues & fixes)

Swagger doesn’t open / app shows https only: go to https://localhost:5001/swagger or use http://localhost:5000/swagger depending on console message.

Database file not created: Ensure the app has write access to the folder. db.Database.EnsureCreated() in Program.cs should create crm.db.

No cases after POST:

Check logs (Console or logs/emailtocase_.log) for Created case ....

Background worker may be sleeping or crashed — check console logs and exception traces.

Port conflicts: Use ASPNETCORE_URLS environment variable or change launch settings.

SSL certificate issues: For local dev, use HTTP or trust the dev certificate (dotnet dev-certs https --trust).

How to run & demo (exact commands for VS Code)

Extract EmailToCase_Upgraded.zip.

Open folder in VS Code (you already use C# extension).

Terminal:

dotnet restore
dotnet run


or press F5 to debug (ensure required assets created).

Open https://localhost:5001/swagger (or http://localhost:5000/swagger).

POST /api/email/receive with the sample JSON. Response should be quick.

Wait ~1–2 seconds and then GET /api/cases to see created case.

View logs in terminal or open logs/emailtocase_*.log.

You can also run the included script (Linux/Mac or WSL) ./curl_command.sh which posts a sample email and lists cases.

How to explain this in an interview — short script

One-liner: “I built an Email-to-Case pipeline using ASP.NET Core where incoming emails are accepted by a REST API, queued, and processed by a background worker that creates Cases in a DB.”

Why this design: “It decouples ingestion from processing — the API stays responsive and the worker can be scaled independently. Using a queue makes the system more resilient to spikes and transient downstream issues.”

Tech choices:

ASP.NET Core for web API and hosted services.

EF Core + SQLite for quick local persistence (production would use SQL Server).

In-memory queue for demo; production would use Azure Service Bus.

Serilog for structured logging.

What you’d change for production:

Replace in-memory queue with Azure Service Bus (durable, retries, dead-letter).

Use SQL Server (or managed DB), add migrations.

Add authentication (JWT/Azure AD), add monitoring and health checks, add CI/CD.

Potential pitfalls & mitigations:

Message loss → durable queue/outbox.

Duplicates → idempotency keys and database constraints.

Scaling → backplane, distributed queues, stateless workers.

Extension options (I can add any of these)

Replace the EmailQueue with an Azure Service Bus implementation (I’ll add code for the sender in controller and receiver as a ServiceBusProcessor background worker; you’d need a connection string).

Add JWT authentication and secure controllers/hub.

Add EF Core Migrations and provide dotnet ef commands + generated migration files.

Add a simple React admin UI with /cases listing and form to post email JSON.

Add Docker Compose to run the API plus a lightweight UI or inspector.

Files location quick map
/EmailToCase_Upgraded/
  EmailToCase.csproj
  Program.cs
  appsettings.json
  README.md
  Dockerfile
  Controllers/
    EmailController.cs
    CasesController.cs
  Services/
    EmailQueue.cs
    EmailProcessorService.cs
  Data/
    CrmDbContext.cs
  Models/
    Case.cs
  DTOs/
    EmailDto.cs
  logs/ (created at runtime)
  crm.db (created at runtime)

Short checklist to show in interview/demo

Show Program.cs to explain DI, hosted service registration, and DB setup.

Open EmailController.cs and point out enqueue call.

Open EmailProcessorService.cs and point out dequeue → DB save.

Run dotnet run, post sample email in Swagger, show console logs, then GET /api/cases.

Explain production improvements (durable queue, auth, migrations, metrics).

If you want, I’ll now:

Add Azure Service Bus variant (I’ll make the controller push to Service Bus and update hosted service to use ServiceBusProcessor instead of in-memory queue) — I’ll include code and README notes (you will need a Service Bus connection string).

Add a small React UI that lists cases and has a form to POST emails (complete with source files and new zip).

Add EF Core Migrations and include migration files ready to run.
