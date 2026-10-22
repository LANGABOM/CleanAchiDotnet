# Clean Architecture: How an Endpoint Should Look

A practical guide for building endpoints the same way across the team. It follows one example, **Create Customer** and **Get Customer**, from the HTTP request all the way to the database and back.

> **Note:** The names here (`MyApp`, `Customer`) are examples. Use your solution's real names, but keep the same structure and responsibilities.
>
> **Sample solution:** Everything in this guide is implemented in the `CleanArchitectureSample` solution, which you can download and run locally. Its README explains how to run it and deploy it to Azure.

---

## 1. The big picture

Every request goes through the same layers. Each layer has **one job**.

```text
HTTP request
   │
   ▼
Controller (Api)
   1. Authentication ([Authorize] + middleware, before the action runs; see section 9)
   2. Log request arrival (debug, safe details only)
   3. Call the application service
   │
   ▼
Application service
   4. Validate        → Validators/CreateCustomerValidator
   5. Business rules  → e.g. "does this email already exist?"
   6. Map in          → Mappers/CustomerMapper (request → entity)
   7. Call Infrastructure through an interface (repository, Kafka publisher)
   8. Map out         → Mappers/CustomerMapper (entity → response)
   9. Return a Result<T>
   │
   ▼
Controller (Api)
  10. Switch on Result.Status → HTTP status code
   │
   ▼
HTTP response
```

### What belongs where

| Layer | Owns | Never does |
| --- | --- | --- |
| **Api (Controller)** | Routes, `[Authorize]`, a debug log, calling the service, turning the `Result` into an HTTP status | Validation, business rules, mapping, database or Kafka calls |
| **Application** | The use case: calls its validator class, its mapper class, and Infrastructure through interfaces; returns a `Result` | Know about HTTP status codes, SQL, DbContext, HttpClient or the Kafka SDK |
| **Domain** | Entities, enums and rules that belong to the entity itself | Depend on any other layer |
| **Infrastructure** | Database access, external APIs, the Kafka producer and its settings; implements Application interfaces | Return HTTP codes or make business decisions |
| **Composition root** (`Program.cs` + each layer's `DependencyInjection.cs`) | Binding appsettings to typed options, registering implementations, choosing Kafka or legacy | Business logic |

### Dependency rule

There are two different pictures here, and mixing them up is the most common source of confusion.

**Project references (who knows about whom at compile time):**

```text
                       ┌──────────────► Domain
Api ───────────► Application              ▲
 │                     ▲                  │
 └─ Program.cs ──► Infrastructure ────────┘
```

- **Domain** references nothing.
- **Application** references only Domain. It defines the **interfaces** it needs (e.g. `ICustomerRepository`).
- **Infrastructure** references Application and Domain, because it **implements** those interfaces and uses the entities.
- **Api** references Application (to call services and use its models). It also references Infrastructure, but only in `Program.cs`, to register the implementations in DI.

Infrastructure isn't "after" Domain. Application never references Infrastructure, which is the whole point.

**Runtime calls (what actually runs during a request):**

```text
Api controller → Application service → ICustomerRepository (Application interface)
                                              │
                                              ▼  resolved by DI
                                       CustomerRepository (Infrastructure)
```

Application only ever calls its own interface. DI hands it the Infrastructure implementation at runtime.

---

## 2. Example requests and responses

This is what the endpoint looks like from the outside. Every response below maps to a branch in the controller's `switch`.

### Create a customer

**Request**

```http
POST /api/customers
Authorization: Bearer <token>
Content-Type: application/json

{
  "firstName": "Thandi",
  "lastName": "Nkosi",
  "email": "thandi@example.com"
}
```

**201 Created**: saved successfully

```http
HTTP/1.1 201 Created
Location: /api/customers/3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90

{
  "id": "3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90",
  "fullName": "Thandi Nkosi",
  "email": "thandi@example.com",
  "createdUtc": "2026-10-21T10:15:00Z"
}
```

**400 Bad Request**: failed validation

```http
HTTP/1.1 400 Bad Request

{
  "errors": [
    "'First Name' must not be empty.",
    "'Email' is not a valid email address."
  ]
}
```

**409 Conflict**: email already exists

```http
HTTP/1.1 409 Conflict

{
  "errors": [ "A customer with this email already exists." ]
}
```

**401 Unauthorized**: no or invalid token. The middleware returns this before the controller runs, so there's no body from our code.

**500 Internal Server Error**: something unexpected (database down, timeout). The global exception handler logs the details; the caller only gets a generic message.

```http
HTTP/1.1 500 Internal Server Error

{
  "title": "An unexpected error occurred.",
  "status": 500
}
```

### Get a customer

```http
GET /api/customers/3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90
Authorization: Bearer <token>
```

| Outcome | Response |
| --- | --- |
| Found | `200 OK` with the customer JSON |
| Not found | `404 Not Found` with `{ "errors": [ "Customer ... was not found." ] }` |

---

## 3. Project structure

One project per layer. Inside Application, each kind of class gets its own folder, so validators, mappers and services are always easy to find.

```text
MyApp.Api/
  Authentication/
    AuthenticationSetup.cs         <- JWT + custom schemes and policies
    ApiKeyAuthenticationOptions.cs <- only if you need a custom header
    ApiKeyAuthenticationHandler.cs <- only if you need a custom header
    CurrentUser.cs                 <- implements ICurrentUser
  Controllers/
    CustomersController.cs
  Program.cs
  appsettings.json                 <- Kafka + Messaging sections

MyApp.Application/
  Common/
    Result.cs
  Interfaces/
    ICustomerService.cs
    ICustomerRepository.cs         <- Infrastructure implements this
    ICustomerEventPublisher.cs     <- Infrastructure implements this (Kafka / legacy)
    ICurrentUser.cs                <- Api implements this (who is calling)
  Models/
    CreateCustomerRequest.cs
    CustomerResponse.cs
  Validators/
    CreateCustomerValidator.cs     <- one validator class per request
  Mappers/
    CustomerMapper.cs              <- one mapper class per feature
  Options/
    MessagingOptions.cs            <- only if Application decides Kafka vs legacy
  Services/
    CustomerService.cs
  DependencyInjection.cs

MyApp.Domain/
  Customers/
    Customer.cs

MyApp.Infrastructure/
  Persistence/
    AppDbContext.cs
    OutboxMessage.cs               <- only with the outbox pattern
    Repositories/
      CustomerRepository.cs
  Messaging/
    KafkaOptions.cs
    KafkaCustomerEventPublisher.cs
    LegacyCustomerEventPublisher.cs
    OutboxPublisherWorker.cs       <- only with the outbox pattern
  DependencyInjection.cs
```

---

## 4. The Result type

Every application service returns a `Result<T>`. The status says **what happened in business terms**; the controller turns it into an HTTP code. This keeps HTTP out of the Application layer.

```csharp
// Application/Common/Result.cs
namespace MyApp.Application.Common;

public enum ResultStatus
{
    Ok,
    Created,
    Invalid,
    NotFound,
    Conflict
}

public sealed class Result<T>
{
    public ResultStatus Status { get; }
    public T? Value { get; }
    public IReadOnlyList<string> Errors { get; }

    private Result(ResultStatus status, T? value, IReadOnlyList<string>? errors)
    {
        Status = status;
        Value = value;
        Errors = errors ?? Array.Empty<string>();
    }

    public static Result<T> Ok(T value) => new(ResultStatus.Ok, value, null);
    public static Result<T> Created(T value) => new(ResultStatus.Created, value, null);
    public static Result<T> Invalid(IReadOnlyList<string> errors) => new(ResultStatus.Invalid, default, errors);
    public static Result<T> NotFound(string error) => new(ResultStatus.NotFound, default, new[] { error });
    public static Result<T> Conflict(string error) => new(ResultStatus.Conflict, default, new[] { error });
}
```

| ResultStatus | HTTP code |
| --- | --- |
| `Ok` | 200 OK |
| `Created` | 201 Created |
| `Invalid` | 400 Bad Request |
| `NotFound` | 404 Not Found |
| `Conflict` | 409 Conflict |
| A status the controller doesn't expect (a coding mistake) | 500 Internal Server Error |
| Unexpected exception (database down, timeout) | 500, returned by the global exception handler |

Only add a new status when a controller needs to answer with a different HTTP code for it (for example `Forbidden`).

There's deliberately **no `Error` status**. Unexpected failures aren't business outcomes: they throw, and the global exception handler logs them once and returns the 500 (see "Errors: Result vs exceptions" in section 6).

---

## 5. Controller (Api layer)

The controller stays thin: **log, call the service, switch on the status.**

Authentication is **not** written inside the action. `[Authorize]` plus the authentication middleware handles it before the action runs, so an unauthenticated call never reaches your code and gets a 401 automatically. Section 9 shows how that's set up, for JWT and for custom headers.

> **Primary constructors:** All classes in this guide use C# 12 primary constructors. The dependencies are declared on the class line and used directly, so there's no field-assignment boilerplate.

```csharp
// Api/Controllers/CustomersController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MyApp.Application.Common;
using MyApp.Application.Interfaces;
using MyApp.Application.Models;

namespace MyApp.Api.Controllers;

[ApiController]
[Route("api/customers")]
[Authorize] // 1. Authentication: handled by middleware before the action runs
public sealed class CustomersController(
    ICustomerService customerService,
    ILogger<CustomersController> logger) : ControllerBase
{
    [HttpPost]
    [ProducesResponseType(typeof(CustomerResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<IActionResult> Create(
        [FromBody] CreateCustomerRequest request,
        CancellationToken cancellationToken)
    {
        // 2. Request-arrival diagnostic. It runs before validation, so it also records
        //    requests that get rejected. Safe details only, never the body or personal data.
        logger.LogDebug("Create customer request received");

        // 3. Call the application service
        var result = await customerService.CreateAsync(request, cancellationToken);

        // 10. Map the business result to an HTTP response
        return result.Status switch
        {
            ResultStatus.Created => CreatedAtAction(nameof(GetById), new { id = result.Value!.Id }, result.Value),
            ResultStatus.Invalid => BadRequest(new { errors = result.Errors }),
            ResultStatus.Conflict => Conflict(new { errors = result.Errors }),
            // A status this action doesn't expect is a coding mistake, not bad input
            _ => Problem(statusCode: StatusCodes.Status500InternalServerError, title: "An unexpected error occurred.")
        };
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(CustomerResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        logger.LogDebug("Get customer {CustomerId} requested", id);

        var result = await customerService.GetByIdAsync(id, cancellationToken);

        return result.Status switch
        {
            ResultStatus.Ok => Ok(result.Value),
            ResultStatus.NotFound => NotFound(new { errors = result.Errors }),
            _ => Problem(statusCode: StatusCodes.Status500InternalServerError, title: "An unexpected error occurred.")
        };
    }
}
```

**Tips**

- Pass the `CancellationToken` all the way down. ASP.NET cancels it when the caller disconnects.
- Never return the exception message to the caller in the 500. It goes to the logs, not the response.
- Once the switch repeats across controllers, move it into one `ToActionResult()` extension method on `Result<T>`. Keep that extension in the **Api** project (e.g. `Api/Extensions/ResultExtensions.cs`), because it uses MVC types. Application must never reference MVC or HTTP result types.
- The request-arrival log runs before validation on purpose: it helps diagnose rejected calls and logs no request data. If your team prefers to log accepted work only, drop it and rely on the log in the Application service after validation.

---

## 6. Application layer

The service runs the use case in a fixed order:

1. Validate (validator class)
2. Check business rules
3. Map the request to an entity (mapper class)
4. Call Infrastructure through its interface
5. Map the entity to a response (mapper class)
6. Return a `Result`

It never touches `DbContext`, HTTP or Kafka directly.

### Models: `Application/Models/`

```csharp
// Application/Models/CreateCustomerRequest.cs
namespace MyApp.Application.Models;

public sealed record CreateCustomerRequest(string FirstName, string LastName, string Email);
```

```csharp
// Application/Models/CustomerResponse.cs
namespace MyApp.Application.Models;

public sealed record CustomerResponse(Guid Id, string FullName, string Email, DateTime CreatedUtc);
```

### Validator: `Application/Validators/`

One class per request. Uses FluentValidation; swap in the solution's own validator if it has one.

```csharp
// Application/Validators/CreateCustomerValidator.cs
using FluentValidation;
using MyApp.Application.Models;

namespace MyApp.Application.Validators;

public sealed class CreateCustomerValidator : AbstractValidator<CreateCustomerRequest>
{
    public CreateCustomerValidator()
    {
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(256);
    }
}
```

### Mapper: `Application/Mappers/`

Its own class, injected into the service. No business logic, no database calls.

```csharp
// Application/Mappers/CustomerMapper.cs
using MyApp.Application.Models;
using MyApp.Domain.Customers;

namespace MyApp.Application.Mappers;

public sealed class CustomerMapper
{
    public Customer ToEntity(CreateCustomerRequest request) =>
        Customer.Create(request.FirstName, request.LastName, request.Email);

    public CustomerResponse ToResponse(Customer customer) =>
        new(customer.Id, $"{customer.FirstName} {customer.LastName}", customer.Email, customer.CreatedUtc);
}
```

### Interfaces: `Application/Interfaces/`

Application owns them; Infrastructure implements the repository and the publisher.

```csharp
// Application/Interfaces/ICustomerService.cs
using MyApp.Application.Common;
using MyApp.Application.Models;

namespace MyApp.Application.Interfaces;

public interface ICustomerService
{
    Task<Result<CustomerResponse>> CreateAsync(CreateCustomerRequest request, CancellationToken cancellationToken);
    Task<Result<CustomerResponse>> GetByIdAsync(Guid id, CancellationToken cancellationToken);
}
```

```csharp
// Application/Interfaces/ICustomerRepository.cs
using MyApp.Domain.Customers;

namespace MyApp.Application.Interfaces;

public interface ICustomerRepository
{
    Task<bool> EmailExistsAsync(string email, CancellationToken cancellationToken);
    Task<Customer?> GetByIdAsync(Guid id, CancellationToken cancellationToken);
    // Returns false when the email already exists (unique index violation).
    Task<bool> TryAddAsync(Customer customer, CancellationToken cancellationToken);
}
```

```csharp
// Application/Interfaces/ICustomerEventPublisher.cs
namespace MyApp.Application.Interfaces;

// Provider-neutral: no Kafka types here. Infrastructure decides how it's sent.
public interface ICustomerEventPublisher
{
    Task PublishCustomerCreatedAsync(Guid customerId, CancellationToken cancellationToken);
}
```

### Service: `Application/Services/`

```csharp
// Application/Services/CustomerService.cs
using FluentValidation;
using Microsoft.Extensions.Logging;
using MyApp.Application.Common;
using MyApp.Application.Interfaces;
using MyApp.Application.Mappers;
using MyApp.Application.Models;

namespace MyApp.Application.Services;

public sealed class CustomerService(
    IValidator<CreateCustomerRequest> validator,
    CustomerMapper mapper,
    ICustomerRepository customerRepository,
    ICustomerEventPublisher eventPublisher,
    ILogger<CustomerService> logger) : ICustomerService
{
    private const string DuplicateEmailMessage = "A customer with this email already exists.";

    public async Task<Result<CustomerResponse>> CreateAsync(
        CreateCustomerRequest request,
        CancellationToken cancellationToken)
    {
        // 4. Validate: the rules live in Validators/CreateCustomerValidator
        var validation = await validator.ValidateAsync(request, cancellationToken);
        if (!validation.IsValid)
        {
            return Result<CustomerResponse>.Invalid(
                validation.Errors.Select(e => e.ErrorMessage).ToList());
        }

        // 5. Business rules that need data (a quick, friendly check for the common case)
        if (await customerRepository.EmailExistsAsync(request.Email, cancellationToken))
        {
            return Result<CustomerResponse>.Conflict(DuplicateEmailMessage);
        }

        // 6. Map request -> entity: Mappers/CustomerMapper
        var customer = mapper.ToEntity(request);

        // 7. Call Infrastructure through its interfaces.
        //    The unique index is the real guarantee: two requests can both pass the check
        //    above, so the repository reports the losing insert as false instead of throwing.
        if (!await customerRepository.TryAddAsync(customer, cancellationToken))
        {
            return Result<CustomerResponse>.Conflict(DuplicateEmailMessage);
        }

        // Not atomic with the save above: see "Saving and publishing" in section 8.
        await eventPublisher.PublishCustomerCreatedAsync(customer.Id, cancellationToken);

        // Log once, safe fields only (an ID, never the email or request body)
        logger.LogInformation("Customer created. CustomerId={CustomerId}", customer.Id);

        // 8 + 9. Map entity -> response and return the result to the controller
        return Result<CustomerResponse>.Created(mapper.ToResponse(customer));
    }

    public async Task<Result<CustomerResponse>> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        var customer = await customerRepository.GetByIdAsync(id, cancellationToken);

        return customer is null
            ? Result<CustomerResponse>.NotFound($"Customer {id} was not found.")
            : Result<CustomerResponse>.Ok(mapper.ToResponse(customer));
    }
}
```

### Domain entity: `MyApp.Domain`

Shown here because the mapper uses it.

```csharp
// Domain/Customers/Customer.cs
namespace MyApp.Domain.Customers;

public sealed class Customer
{
    public Guid Id { get; private set; }
    public string FirstName { get; private set; } = string.Empty;
    public string LastName { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;
    public DateTime CreatedUtc { get; private set; }

    private Customer() { } // for EF Core

    public static Customer Create(string firstName, string lastName, string email) => new()
    {
        Id = Guid.NewGuid(),
        FirstName = firstName.Trim(),
        LastName = lastName.Trim(),
        Email = email.Trim().ToLowerInvariant(),
        CreatedUtc = DateTime.UtcNow
    };
}
```

### Errors: Result vs exceptions

- **Expected outcomes** (invalid, not found, conflict) come back as a `Result`.
- **Unexpected problems** (database down, timeout) are exceptions. Let them bubble up to the global exception handler, which logs them once and returns a 500.
- Don't wrap every method in try/catch. It hides the real error and logs the same failure in several places.

---

## 7. Infrastructure layer

Infrastructure holds the **technical integrations**: the database, Kafka, external APIs and anything else that talks to the outside world. It implements the interfaces the Application layer defined, returns entities or data, never returns HTTP codes, and never makes business decisions.

### Repository (EF Core)

```csharp
// Infrastructure/Persistence/Repositories/CustomerRepository.cs
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using MyApp.Application.Interfaces;
using MyApp.Domain.Customers;

namespace MyApp.Infrastructure.Persistence.Repositories;

internal sealed class CustomerRepository(AppDbContext db) : ICustomerRepository
{
    public Task<bool> EmailExistsAsync(string email, CancellationToken cancellationToken) =>
        db.Customers
            .AsNoTracking()
            .AnyAsync(c => c.Email == email.Trim().ToLowerInvariant(), cancellationToken);

    public Task<Customer?> GetByIdAsync(Guid id, CancellationToken cancellationToken) =>
        db.Customers
            .AsNoTracking()
            .FirstOrDefaultAsync(c => c.Id == id, cancellationToken);

    public async Task<bool> TryAddAsync(Customer customer, CancellationToken cancellationToken)
    {
        db.Customers.Add(customer);

        try
        {
            await db.SaveChangesAsync(cancellationToken);
            return true;
        }
        catch (DbUpdateException ex) when (ex.InnerException is SqlException { Number: 2601 or 2627 })
        {
            // 2601 / 2627 = SQL Server unique index / unique constraint violation.
            // Translate it into a business outcome; every other database error still throws.
            db.Entry(customer).State = EntityState.Detached;
            return false;
        }
    }
}
```

### Same interface, stored procedure version (Dapper)

The shape stays the same, only the body changes.

```csharp
internal sealed class CustomerRepository(IDbConnectionFactory connectionFactory) : ICustomerRepository
{
    public async Task<Customer?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        await using var connection = connectionFactory.Create();

        var command = new CommandDefinition(
            "dbo.Customer_GetById",
            new { CustomerId = id },
            commandType: CommandType.StoredProcedure,
            cancellationToken: cancellationToken);

        return await connection.QuerySingleOrDefaultAsync<Customer>(command);
    }

    // ...other methods
}
```

**Tips**

- Use `AsNoTracking()` on reads. EF Core doesn't need to track entities you won't update, so it's faster and uses less memory.
- The email check isn't a guarantee on its own. Two requests can pass it at the same time, so keep a **unique index on `Email`** in the database as the real safeguard. `TryAddAsync` turns that violation into `false`, which the service returns as a 409 instead of a 500. If the table has more than one unique index, also check the index name in the exception message so you don't report the wrong conflict.
- An external API client follows the same pattern: an interface in Application (e.g. `IPaymentGateway`), the `HttpClient` implementation in Infrastructure.

---

## 8. Kafka: config and switching

Kafka settings and the producer live in **Infrastructure**. Application only ever sees `ICustomerEventPublisher`, so the service code doesn't change when you switch between Kafka and the legacy sender.

### appsettings.json

```json
{
  "Messaging": {
    "UseKafka": true
  },
  "Kafka": {
    "BootstrapServers": "localhost:9092",
    "ClientId": "myapp-api",
    "CustomerCreatedTopic": "customer-created"
  }
}
```

> **Secrets:** Credentials (SASL username/password, certificates) never go in appsettings in source control. Load them from user secrets locally and the approved secret store (e.g. Key Vault) in other environments.

### Typed options

Bind the sections to classes. Never inject raw `IConfiguration` into services.

```csharp
// Infrastructure/Messaging/KafkaOptions.cs
namespace MyApp.Infrastructure.Messaging;

public sealed class KafkaOptions
{
    public const string SectionName = "Kafka";

    public string BootstrapServers { get; init; } = string.Empty;
    public string ClientId { get; init; } = string.Empty;
    public string CustomerCreatedTopic { get; init; } = string.Empty;
}
```

```csharp
// Application/Options/MessagingOptions.cs (only needed for option B below)
namespace MyApp.Application.Options;

public sealed class MessagingOptions
{
    public const string SectionName = "Messaging";

    public bool UseKafka { get; init; }
}
```

### Publishers

Infrastructure implements the Application interface.

```csharp
// Infrastructure/Messaging/KafkaCustomerEventPublisher.cs
using System.Text.Json;
using Confluent.Kafka;
using Microsoft.Extensions.Options;
using MyApp.Application.Interfaces;

namespace MyApp.Infrastructure.Messaging;

internal sealed class KafkaCustomerEventPublisher(
    IProducer<string, string> producer,
    IOptions<KafkaOptions> options) : ICustomerEventPublisher
{
    private readonly KafkaOptions _options = options.Value;

    public async Task PublishCustomerCreatedAsync(Guid customerId, CancellationToken cancellationToken)
    {
        var message = new Message<string, string>
        {
            Key = customerId.ToString(), // same key = same partition = ordered per customer
            Value = JsonSerializer.Serialize(new { CustomerId = customerId, OccurredUtc = DateTime.UtcNow })
        };

        await producer.ProduceAsync(_options.CustomerCreatedTopic, message, cancellationToken);
    }
}
```

```csharp
// Infrastructure/Messaging/LegacyCustomerEventPublisher.cs
using MyApp.Application.Interfaces;

namespace MyApp.Infrastructure.Messaging;

internal sealed class LegacyCustomerEventPublisher : ICustomerEventPublisher
{
    public Task PublishCustomerCreatedAsync(Guid customerId, CancellationToken cancellationToken)
    {
        // Call the existing (pre-Kafka) sender here.
        return Task.CompletedTask;
    }
}
```

### Option A: switch in DI (default)

Use this when the toggle is per environment. The setting decides which implementation gets registered, and `CustomerService` stays exactly as shown above.

```csharp
// Infrastructure/DependencyInjection.cs, called from AddInfrastructure
private static void AddMessaging(IServiceCollection services, IConfiguration configuration)
{
    services.Configure<KafkaOptions>(configuration.GetSection(KafkaOptions.SectionName));

    var useKafka = configuration.GetValue<bool>("Messaging:UseKafka");

    if (useKafka)
    {
        // One producer for the whole app: it's thread-safe and expensive to create.
        services.AddSingleton<IProducer<string, string>>(sp =>
        {
            var kafka = sp.GetRequiredService<IOptions<KafkaOptions>>().Value;

            var producerConfig = new ProducerConfig
            {
                BootstrapServers = kafka.BootstrapServers,
                ClientId = kafka.ClientId,
                Acks = Acks.All,
                EnableIdempotence = true
            };

            return new ProducerBuilder<string, string>(producerConfig).Build();
        });

        services.AddScoped<ICustomerEventPublisher, KafkaCustomerEventPublisher>();
    }
    else
    {
        services.AddScoped<ICustomerEventPublisher, LegacyCustomerEventPublisher>();
    }
}
```

### Option B: switch in Application (only when the team wants the choice in the use case)

`UseKafka` is a global setting, so moving the `if` into the service doesn't make it a per-request decision. It only makes the choice visible in the Application layer. Use option B only if your team specifically wants that; otherwise use option A. If a use case genuinely has to choose per event, base the choice on business data (for example the customer's region), not on a global flag.

For option B, register both publishers under their own interfaces and bind `MessagingOptions`.

```csharp
// Application/Interfaces/IKafkaCustomerEventPublisher.cs
namespace MyApp.Application.Interfaces;

public interface IKafkaCustomerEventPublisher : ICustomerEventPublisher { }
```

```csharp
// Application/Interfaces/ILegacyCustomerEventPublisher.cs
namespace MyApp.Application.Interfaces;

public interface ILegacyCustomerEventPublisher : ICustomerEventPublisher { }
```

The publishers implement the **specific** interface instead of the shared one. Only the class line changes; the method bodies stay as shown above.

```csharp
internal sealed class KafkaCustomerEventPublisher(
    IProducer<string, string> producer,
    IOptions<KafkaOptions> options) : IKafkaCustomerEventPublisher

internal sealed class LegacyCustomerEventPublisher : ILegacyCustomerEventPublisher
```

Register both publishers and bind `MessagingOptions`. This replaces the `if` from option A:

```csharp
// Infrastructure/DependencyInjection.cs: AddMessaging, option B version
private static void AddMessaging(IServiceCollection services, IConfiguration configuration)
{
    services.Configure<KafkaOptions>(configuration.GetSection(KafkaOptions.SectionName));
    services.Configure<MessagingOptions>(configuration.GetSection(MessagingOptions.SectionName));

    services.AddSingleton<IProducer<string, string>>(sp =>
    {
        var kafka = sp.GetRequiredService<IOptions<KafkaOptions>>().Value;

        var producerConfig = new ProducerConfig
        {
            BootstrapServers = kafka.BootstrapServers,
            ClientId = kafka.ClientId,
            Acks = Acks.All,
            EnableIdempotence = true
        };

        return new ProducerBuilder<string, string>(producerConfig).Build();
    });

    services.AddScoped<IKafkaCustomerEventPublisher, KafkaCustomerEventPublisher>();
    services.AddScoped<ILegacyCustomerEventPublisher, LegacyCustomerEventPublisher>();
}
```

```csharp
// Application/Services/CustomerService.cs (option B version)
public sealed class CustomerService(
    IValidator<CreateCustomerRequest> validator,
    CustomerMapper mapper,
    ICustomerRepository customerRepository,
    IKafkaCustomerEventPublisher kafkaPublisher,
    ILegacyCustomerEventPublisher legacyPublisher,
    IOptions<MessagingOptions> messagingOptions,
    ILogger<CustomerService> logger) : ICustomerService
{
    private readonly MessagingOptions _messaging = messagingOptions.Value;

    // CreateAsync is the same as before, except it calls this instead of
    // eventPublisher.PublishCustomerCreatedAsync(...)
    private Task PublishCustomerCreatedAsync(Guid customerId, CancellationToken cancellationToken) =>
        _messaging.UseKafka
            ? kafkaPublisher.PublishCustomerCreatedAsync(customerId, cancellationToken)
            : legacyPublisher.PublishCustomerCreatedAsync(customerId, cancellationToken);
}
```

Option B needs the `Microsoft.Extensions.Options` package in the Application project for `IOptions<T>`.

### Saving and publishing: what can go wrong

The example saves the customer and then publishes the event. Those are two separate operations, so they can't succeed or fail together:

1. The customer is saved.
2. Kafka is down, so the publish throws.
3. The caller gets a **500**, even though the customer **was** created.
4. The caller retries and gets a **409** (email already exists) for a record they think failed.
5. The `CustomerCreated` event is never sent.

So the direct publish in this guide is an **illustrative example, not the production pattern** for guaranteed delivery. Publishing directly is fine when a lost event is acceptable, for example a nice-to-have notification. When the event **must** be delivered, the standard option is the **outbox pattern**:

- Save the customer **and** an outbox row in the same `SaveChangesAsync`, so they commit together or not at all.
- A background worker reads unsent outbox rows, publishes them, and marks them as sent.
- The API no longer publishes directly, so a Kafka outage can't fail the request. With the outbox, remove the `eventPublisher` call (and dependency) from `CustomerService`.

```csharp
// Infrastructure/Persistence/OutboxMessage.cs
namespace MyApp.Infrastructure.Persistence;

internal sealed class OutboxMessage
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string Type { get; init; } = string.Empty;
    public Guid CustomerId { get; init; }
    public DateTime CreatedUtc { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedUtc { get; set; }
}
```

```csharp
// Infrastructure/Persistence/Repositories/CustomerRepository.cs (outbox version of TryAddAsync)
public async Task<bool> TryAddAsync(Customer customer, CancellationToken cancellationToken)
{
    var outboxMessage = new OutboxMessage { Type = "CustomerCreated", CustomerId = customer.Id };

    db.Customers.Add(customer);
    db.OutboxMessages.Add(outboxMessage); // same SaveChangesAsync = same transaction

    try
    {
        await db.SaveChangesAsync(cancellationToken);
        return true;
    }
    catch (DbUpdateException ex) when (ex.InnerException is SqlException { Number: 2601 or 2627 })
    {
        db.Entry(customer).State = EntityState.Detached;
        db.Entry(outboxMessage).State = EntityState.Detached;
        return false;
    }
}
```

```csharp
// Infrastructure/Messaging/OutboxPublisherWorker.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using MyApp.Application.Interfaces;
using MyApp.Infrastructure.Persistence;

namespace MyApp.Infrastructure.Messaging;

internal sealed class OutboxPublisherWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<OutboxPublisherWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await PublishPendingAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                // Unsent rows stay unsent and are retried on the next tick.
                logger.LogError(ex, "Outbox publish failed");
            }
        }
    }

    private async Task PublishPendingAsync(CancellationToken cancellationToken)
    {
        // The worker is a singleton, so it creates a scope for the scoped DbContext and publisher.
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publisher = scope.ServiceProvider.GetRequiredService<ICustomerEventPublisher>();

        var pending = await db.OutboxMessages
            .Where(m => m.ProcessedUtc == null)
            .OrderBy(m => m.CreatedUtc)
            .Take(50)
            .ToListAsync(cancellationToken);

        foreach (var message in pending)
        {
            await publisher.PublishCustomerCreatedAsync(message.CustomerId, cancellationToken);
            message.ProcessedUtc = DateTime.UtcNow;
            await db.SaveChangesAsync(cancellationToken); // mark each one as soon as it's sent
        }
    }
}
```

Register it with `services.AddHostedService<OutboxPublisherWorker>();` and add `DbSet<OutboxMessage> OutboxMessages` to `AppDbContext`.

What juniors should know about the outbox:

- **Delivery is "at least once".** If the publish succeeds but marking the row fails, the event is sent again. Consumers must handle duplicates, for example by ignoring a customer ID they've already processed.
- **Multiple API instances** each run the worker and can pick up the same rows. Either run the worker in one instance only, or lock rows when reading them.
- Real outbox tables usually store `Type` + a JSON `Payload` so one table serves every event. Libraries like MassTransit include a ready-made outbox if the solution already uses one.

**Tips**

- **Default to option A.** It keeps the `if` out of every service and makes the legacy path easy to delete later.
- Use the outbox pattern whenever losing an event isn't acceptable (see "Saving and publishing" above).
- Log the topic and the customer ID on failure, never the message payload.

---

## 9. Authentication: JWT and custom headers

### How it works

Authentication and authorization run as **middleware**, before your controller. The controller only declares what it needs with an attribute.

```text
Request
  │
  ▼
UseAuthentication()   Reads the Authorization header (or your custom header),
  │                   validates it, and sets HttpContext.User
  ▼
UseAuthorization()    Checks the endpoint's [Authorize] / policy against User
  │                     - not authenticated              → 401 Unauthorized
  │                     - authenticated, but not allowed  → 403 Forbidden
  ▼
Controller action     Only runs if both checks passed
```

- **Authentication** = *who are you?* (validate the token or key)
- **Authorization** = *are you allowed?* (`[Authorize]`, roles, policies)

### Where things go

| What | Where |
| --- | --- |
| Scheme setup (JWT, custom header) and policies | `Api/Authentication/AuthenticationSetup.cs`, called from `Program.cs` |
| Custom header checking logic | `Api/Authentication/ApiKeyAuthenticationHandler.cs` |
| Settings (authority, audience, header name) | `Authentication` section in appsettings |
| Secrets (API keys, signing keys) | User secrets locally, the approved secret store (e.g. Key Vault) elsewhere. Never in source control |
| Which endpoints need what | `[Authorize]`, `[Authorize(Policy = "...")]`, `[AllowAnonymous]` on controllers and actions |
| Resource rules ("can this user edit *this* customer?") | Application service, using `ICurrentUser` |

Never check tokens or headers by hand inside a controller action or the Application layer.

### Option 1: JWT bearer (most APIs)

Callers send:

```http
GET /api/customers/3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

You **don't call anything per request** and you **don't write a handler**. The JWT middleware checks the token itself:

- the **signature**, using the identity provider's public signing keys (downloaded once from its `/.well-known/openid-configuration` endpoint and cached),
- the **issuer**, **audience** and **expiry**.

If any check fails, the caller gets a 401 before the controller runs. It's configuration only (package: `Microsoft.AspNetCore.Authentication.JwtBearer`).

ASP.NET reads JWT settings automatically from the `Authentication:Schemes:Bearer` section, so the setup code doesn't need to copy them by hand:

```json
{
  "Authentication": {
    "Schemes": {
      "Bearer": {
        "Authority": "https://login.microsoftonline.com/<tenant-id>/v2.0",
        "ValidAudiences": [ "api://myapp-api" ]
      }
    },
    "ApiKey": {
      "HeaderName": "X-Api-Key"
    }
  }
}
```

**Testing locally without an identity provider:** run `dotnet user-jwts create --role Admin` in the Api project folder. It prints a token that's valid for your local app and writes the matching settings for you. Paste the token into `Authorization: Bearer <token>`.

Two cases need something different:

- **Your own app issues the tokens with a shared secret.** Set `options.TokenValidationParameters` with the signing key (from the secret store) instead of `Authority`.
- **Opaque tokens** (random strings, not JWTs) can't be checked locally. Those need a call to the provider's token introspection endpoint.

### Option 2: a custom header (partners, callbacks, service-to-service)

Use this when a caller sends something like `X-Api-Key: <key>` instead of a JWT. It takes two small classes in `Api/Authentication/`.

**The options** (what the handler needs to know):

```csharp
// Api/Authentication/ApiKeyAuthenticationOptions.cs
using Microsoft.AspNetCore.Authentication;

namespace MyApp.Api.Authentication;

public sealed class ApiKeyAuthenticationOptions : AuthenticationSchemeOptions
{
    public string HeaderName { get; set; } = "X-Api-Key";
    public string ApiKey { get; set; } = string.Empty; // from secrets, never appsettings in git
}
```

**The handler** (where the checking logic lives):

```csharp
// Api/Authentication/ApiKeyAuthenticationHandler.cs
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

namespace MyApp.Api.Authentication;

public sealed class ApiKeyAuthenticationHandler(
    IOptionsMonitor<ApiKeyAuthenticationOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder) : AuthenticationHandler<ApiKeyAuthenticationOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // 1. No header: this scheme doesn't apply -> 401 if the endpoint requires it
        if (!Request.Headers.TryGetValue(Options.HeaderName, out var provided))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        // 2. Misconfiguration guard: never accept anything when no key is configured
        if (string.IsNullOrEmpty(Options.ApiKey))
        {
            return Task.FromResult(AuthenticateResult.Fail("API key is not configured."));
        }

        // 3. Compare in constant time so the key can't be guessed from response timing
        var expected = Encoding.UTF8.GetBytes(Options.ApiKey);
        var actual = Encoding.UTF8.GetBytes(provided.ToString());

        if (!CryptographicOperations.FixedTimeEquals(expected, actual))
        {
            return Task.FromResult(AuthenticateResult.Fail("Invalid API key.")); // -> 401
        }

        // 4. Valid: create the caller's identity, which becomes HttpContext.User
        var identity = new ClaimsIdentity([new Claim(ClaimTypes.Name, "api-key-client")], Scheme.Name);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme.Name);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

The key itself goes in secrets, for example locally:

```text
dotnet user-secrets set "Authentication:ApiKey:ApiKey" "<the key>"
```

### Registering the schemes and policies

Both options are registered in one place. JWT is the default scheme, so a plain `[Authorize]` means "valid JWT". The custom header gets its own **policy**, which names its scheme.

```csharp
// Api/Authentication/AuthenticationSetup.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using MyApp.Application.Interfaces;

namespace MyApp.Api.Authentication;

public static class AuthenticationSetup
{
    public const string ApiKeyScheme = "ApiKey";
    public const string PartnerCallbackPolicy = "PartnerCallback";

    public static IServiceCollection AddApiAuthentication(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme) // default for plain [Authorize]
            .AddJwtBearer() // settings come from Authentication:Schemes:Bearer
            // Only needed if you use a custom header:
            .AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>(ApiKeyScheme, options =>
                configuration.GetSection("Authentication:ApiKey").Bind(options));

        services.AddAuthorization(options =>
        {
            options.AddPolicy(PartnerCallbackPolicy, policy => policy
                .AddAuthenticationSchemes(ApiKeyScheme)
                .RequireAuthenticatedUser());
        });

        // Lets the Application layer ask "who is calling?" without knowing about HTTP
        services.AddHttpContextAccessor();
        services.AddScoped<ICurrentUser, CurrentUser>();

        return services;
    }
}
```

**Different header per client?** Register another scheme with its own name and settings section (e.g. `"PartnerB"` bound to `Authentication:PartnerB`), and a policy that names it.

### Using it on controllers

```csharp
[Authorize]                                                     // any valid JWT (default scheme)
public sealed class CustomersController(...) : ControllerBase

[Authorize(Policy = AuthenticationSetup.PartnerCallbackPolicy)] // must send a valid X-Api-Key
public sealed class PartnerCallbacksController(...) : ControllerBase

[Authorize(Roles = "Admin")]                                    // valid JWT with an Admin role claim
public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)

[AllowAnonymous]                                                // opts one action out, e.g. a public ping
public IActionResult Ping() => Ok();
```

```http
POST /api/partner-callbacks
X-Api-Key: <the key>
Content-Type: application/json
```

| Request | Response |
| --- | --- |
| Valid key / token | Action runs |
| Header missing, wrong key, expired or invalid token | `401 Unauthorized` |
| Valid JWT, but missing the required role or policy | `403 Forbidden` |

### Knowing who's calling in the Application layer

For resource rules ("only the owner can edit this customer"), the service needs the caller's ID, but Application must not touch `HttpContext`. So Application defines an interface and Api implements it:

```csharp
// Application/Interfaces/ICurrentUser.cs
namespace MyApp.Application.Interfaces;

public interface ICurrentUser
{
    string? UserId { get; }
    bool IsInRole(string role);
}
```

```csharp
// Api/Authentication/CurrentUser.cs
using System.Security.Claims;
using MyApp.Application.Interfaces;

namespace MyApp.Api.Authentication;

internal sealed class CurrentUser(IHttpContextAccessor httpContextAccessor) : ICurrentUser
{
    private ClaimsPrincipal? User => httpContextAccessor.HttpContext?.User;

    public string? UserId => User?.FindFirstValue(ClaimTypes.NameIdentifier);

    public bool IsInRole(string role) => User?.IsInRole(role) ?? false;
}
```

Inject `ICurrentUser` into a service like any other dependency. Which claim holds the user ID depends on your identity provider, so confirm it against a real token.

**Auth tips**

- `UseAuthentication()` must come **before** `UseAuthorization()` in `Program.cs`.
- Never log tokens, API keys or the `Authorization` header.
- Don't parse or validate JWTs yourself. The middleware does it correctly, including key rotation.

---

## 10. Wiring it together (DI)

Each layer registers its own services through one extension method; `Program.cs` just calls them.

```csharp
// MyApp.Application/DependencyInjection.cs
using FluentValidation;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Application.Interfaces;
using MyApp.Application.Mappers;
using MyApp.Application.Services;
using MyApp.Application.Validators;

namespace MyApp.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<ICustomerService, CustomerService>();
        services.AddSingleton<CustomerMapper>(); // stateless, safe to share
        services.AddValidatorsFromAssemblyContaining<CreateCustomerValidator>();
        return services;
    }
}
```

```csharp
// MyApp.Infrastructure/DependencyInjection.cs
using Confluent.Kafka;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using MyApp.Application.Interfaces;
using MyApp.Infrastructure.Messaging;
using MyApp.Infrastructure.Persistence;
using MyApp.Infrastructure.Persistence.Repositories;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<ICustomerRepository, CustomerRepository>();

        AddMessaging(services, configuration); // Kafka or legacy: see section 8

        return services;
    }

    // private static void AddMessaging(...) shown in section 8
}
```

```csharp
// MyApp.Api/Program.cs (relevant lines)
// Let the validator classes own the rules. Without this, ASP.NET rejects missing/empty
// non-nullable strings itself with a different 400 format before the validator runs.
builder.Services.AddControllers(options =>
    options.SuppressImplicitRequiredAttributeForNonNullableReferenceTypes = true);
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddProblemDetails();
builder.Services.AddApiAuthentication(builder.Configuration); // section 9: JWT, custom header, policies

var app = builder.Build();

app.UseExceptionHandler(); // global handler: logs unexpected exceptions, returns 500
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

- Use `Scoped` for services and repositories so they share the request's `DbContext`.
- Use `Singleton` for stateless classes (mappers) and the Kafka producer.
- Order matters: authentication before authorization, both before the controllers.

---

## 11. Rules of thumb

**Not sure where code goes?** Ask what it knows about: HTTP belongs in the controller, business rules in Application, SQL, external APIs and Kafka in Infrastructure.

| Layer | Do | Common mistake |
| --- | --- | --- |
| Controller | Log safe details, call one service method, switch on `Result.Status` | Validation or business `if`s in the action; calling a repository directly |
| Controller | Rely on `[Authorize]` and middleware for authentication | Checking tokens or claims by hand in every action |
| Application | Validator class → business rules → mapper class → interface → mapper class → `Result` | Returning `IActionResult` or HTTP codes from a service |
| Application | Check ownership/permissions on the resource ("can this user edit this customer?") | Assuming `[Authorize]` covers resource-level access |
| Application | Return a `Result` for expected outcomes | Wrapping everything in try/catch and swallowing the exception |
| Infrastructure | Technical integrations only (database, Kafka, external APIs), `AsNoTracking()` on reads, pass the `CancellationToken` | Business decisions or HTTP status codes in a repository or client |
| All layers | Pass `CancellationToken` through every async call | Dropping it halfway, so a cancelled request keeps hitting the database |
| All layers | Log IDs, operation names and errors | Logging request bodies, emails, phone numbers or tokens |

---

## 12. Checklist: adding a new endpoint

1. Request and response records in `Application/Models/`.
2. Validator class in `Application/Validators/`.
3. Mapper class in `Application/Mappers/` (request → entity, entity → response).
4. Interface in `Application/Interfaces/`, implementation in Infrastructure (repository, API client or Kafka publisher).
5. Service method in `Application/Services/` returning `Result<T>`.
6. Register new classes in each layer's `DependencyInjection.cs`.
7. Controller action: `[Authorize]` or a named policy (section 9), log, call, switch on status, `[ProducesResponseType]` per code including 401 and 500.
8. Add example requests for each response (success, 400, 404, 409) to the API collection (Postman / `.http` file).
9. Unit tests for the service (one per `Result` status) and an integration test for the endpoint.
