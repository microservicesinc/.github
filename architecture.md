Act as a senior software architect and adaptive AI development collaborator. We are transitioning a highly technical, multi-turn architectural session to this fresh thread because the previous context window became saturated. Below is the precise state of our multi-project .NET 10 and Python/Flask distributed microservices ecosystem using AWS CDK and an event-driven Saga pattern. Absorb this context completely; do not write code yet, simply confirm understanding of the current system design state.

### 1. REPOSITORY CORE ARCHITECTURE & SYSTEM BOUNDARIES
We are building a highly decoupled, polyrepo microservices application simulating an event-driven e-commerce transaction loop. The current layout consists of three independent workspaces:
1. `org-infrastructure-repo`: A centralized infrastructure management tree containing shared local development runtimes and global deployments.
2. `orders-service`: A .NET 10 microservice strictly adhering to Clean Architecture and Domain-Driven Design (DDD) principles, using Minimal APIs and CQRS.
3. `inventory-service`: A Python 3.11+ Flask microservice utilizing AWS Lambda for serverless background tasks.

### 2. CORE WORKSPACE DIRECTORY STRUCTURES
The synchronized physical directory state across our repositories looks exactly like this:

[A] org-infrastructure-repo/
├── docker/
│   └── local-dev/
│       ├── docker-compose.yml          # Exposes Centralized LocalStack on port 4566
│       └── localstack-data/            # Persistent local cloud volume (ignored in git)
└── terraform/                          # Handles global ECR and GitHub modules

[B] orders-service/
├── OrdersService.slnx                  # Modern .NET solution format map
├── cdk/                                # .NET 10 AWS CDK Infrastructure Project
│   ├── src/Cdk/Program.cs             # Synthesizes infra stacks using custom Main entrypoint
│   └── src/Cdk/CdkStack.cs             # Provisions the Single-Table Design 'Orders' DynamoDB table
└── src/
    ├── Orders.Domain/                  # Pure Core Domain Layer (Zero external dependencies)
    │   ├── Core/Domain/Order.cs        # Domain Entity enforcing invariant business rules
    │   └── Infrastructure/Data/IOrderRepository.cs # Data access boundary contract interface
    ├── Orders.Infrastructure/          # Low-Level Data Access Technical Engine
    │   └── Data/
    │       ├── DynamoOrderRepository.cs # Concrete implementation utilizing Amazon.DynamoDBv2
    │       └── OrderDynamoDbModel.cs   # Maps domain models to Single-Table attributes (PK/SK)
    ├── Orders.Application/             # Use Case Orchestration Layer
    │   ├── Commands/
    │   │   ├── CreateOrderCommand.cs   # CQRS Request DTO capturing user intent
    │   │   └── CreateOrderCommandHandler.cs # Orchestrates Domain instantiation & SQS dispatch
    │   └── Queries/
    │       ├── GetOrderByIdQuery.cs
    │       └── GetOrderByIdQueryHandler.cs
    └── Orders.Api/                     # Presentation Layer Execution Web Server
        ├── Configuration/
        │   ├── DynamoDbSetup.cs       # Registers IAmazonDynamoDB & IOrderRepository mapping
        │   └── SqsSetup.cs            # Registers IAmazonSQS client referencing LocalStack
        ├── Data/OrderSeeder.cs         # Startup database seeder injecting trace-linked data
        ├── Endpoints/OrderEndpoints.cs # Minimal API HTTP mapping layer (GET, GET by ID, POST)
        ├── appsettings.Development.json # Routes AWS SDK ServiceURL demands to http://localhost:4566
        └── Program.cs                  # Bootstrapping composition root (NSwag OpenAPI setup)


### 3. THE EVENT-DRIVEN TRANSACTION LIFECYCLE (SAGA CHOREOGRAPHY)
We designed and implemented a production-grade asynchronous transaction loop to enforce eventual consistency between boundaries:
1. Intake: Client fires an HTTP POST payload (`itemId`, `traceId`) to `Orders.Api` (`POST /api/orders`).
2. Order Persistence: The `CreateOrderCommandHandler` generates a tracking identifier via `$"ORD_{Guid.NewGuid().ToString("N").ToUpper()[..12]}"`, instantiates the pure `Order` entity, commits its state to DynamoDB as `PENDING`, and returns an HTTP 201 Created code.
3. Message Brokerage: The Handler converts the payload to JSON and dispatches a stock deduction event straight to an AWS SQS queue named `StockUpdateQueue` hosted inside LocalStack on port 4566.
4. Downstream Processing: An Inventory Service Lambda function is triggered by `StockUpdateQueue`. It executes validation, decreases stock inside the `InventoryOrdersTable` (DynamoDB), and passes a completion response back.
5. Live Status Tracking: Clients use an HTTP GET route (`api/orders/{id}`) orchestrated by `GetOrderByIdQueryHandler` to poll the live database records and track the transition from `PENDING` to `APPROVED` or `REJECTED`.

### 4. TECHNICAL DEBT LOGGED FOR IN-DEPTH FUTURE REVIEWS
We formalized architectural tracking guidelines regarding deferred features:
* File: `docs/architecture-notes/tech-debt-item-validation.md`
* Topic: Decoupling Cross-Service Validation Contracts.
* Scope: Temporarily logging the validation loop as an async technical debt item. The API accepts any valid string configuration, allowing the serverless Saga pattern choreography framework to catch catalog discrepancies asynchronously downstream via a separate error-reconciliation SQS queue (`OrderValidationFailQueue`) and a dedicated .NET 10 Lambda update function.

### 5. RESOLVED CRITICAL BUGS & DEVELOPMENT BLOCKERS
We systematically worked through and patched several framework issues:
* System.IO.IOException (inotify user instances limit reached): Resolved by dynamically scaling Linux kernel watch settings via `sudo sysctl fs.inotify.max_user_instances=512` and making it permanent inside `/etc/sysctl.conf`.
* Framework Upgrades & Deprecations: Upgraded the CDK workspace project target directly from `.NET 8` to `.NET 10` (`net10.0`). Replaced the deprecated `builder.Services.AddSwaggerGen()` and `.WithOpenApi()` chains with `NSwag.AspNetCore` components (`builder.Services.AddOpenApiDocument()`, `app.UseOpenApi()`, `app.UseSwaggerUi()`) to ensure .NET 10 compatibility.
* Dependency Pollution Core Fix: Cleared compilation errors inside `Orders.Application` by installing the missing `AWSSDK.SQS` NuGet package directly into its specific `.csproj` dependencies block.
* LocalStack Centralization Pathing Fixes: Moved LocalStack to the shared infrastructure repository. Patched the missing volume configurations in `docker-compose.yml` by explicitly mounting `/var/run/docker.sock:/var/run/docker.sock` and setting `LAMBDA_DOCKER_NETWORK=dev_cloud_network` to allow child Lambda containers to boot and network properly. Fixed DynamoDB Composite Key lookup failures by updating the CLI parameters to pass explicit `PK` (with prefix `"ITEM#"`) and `SK` (with value `"METADATA"`) properties together.

### 6. CURRENT STATE AND IMMEDIATE SYSTEM GOALS
The entire system compiles cleanly with zero errors, and all continuous code-watching daemons (`dotnet watch run`) execute flawlessly. LocalStack is configured as a global local utility cloud engine, while individual CDK codebases live inside their respective decoupled repositories.

We are ready to start our next working session. Acknowledge this current architecture state concisely, match our previous tone, and wait for my next engineering instruction.
