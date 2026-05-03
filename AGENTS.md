# AGENTS.md

## Project Overview
Node.js CQRS-DDD template with TypeScript, Express, and in-memory buses.
A learning/starter template demonstrating CQRS pattern with Command/Query separation.

## Commands
- `npm run dev` - Start backend in watch mode
- `npm run cli` - Run CLI application
- `npm run build` - Compile TypeScript with tsc-alias

## Key Paths
- `src/Apps/Backend/` - Express server application
- `src/Apps/CLI/` - CLI application
- `src/Context/Users/` - User bounded context
- `src/Context/Shared/` - Shared domain infrastructure (kernel)

## Architecture

### CQRS Pattern
- **CommandBus**: `src/Context/Shared/Domain/Bus/CommandBus.ts` (interface)
- **QueryBus**: `src/Context/Shared/Domain/Bus/QueryBus.ts` (interface)
- **EventBus**: `src/Context/Shared/Domain/Bus/EventBus.ts` (interface)
- **In-Memory Implementations**: `src/Context/Shared/Infrastructure/Bus/`

### Commands
| Command | Handler | Location |
|---------|---------|---------|
| RegisterUserCommand | RegisterUserCommandHandler | `src/Context/Users/Application/Commands/RegisterUser/` |

### Queries
| Query | Handler | Location |
|-------|---------|---------|
| GetUserByIdQuery | GetUserByIdQueryHandler | `src/Context/Users/Application/Queries/GetUserById/` |
| GetAllUsersQuery | GetAllUsersQueryHandler | `src/Context/Users/Application/Queries/GetAllUsers/` |

### Domain Layer
- **Base Classes**: `src/Context/Shared/Domain/`
  - `Command.ts` - Abstract base for commands
  - `Query.ts` - Abstract base for queries
  - `AggregateRoot.ts` - Base for aggregates with domain events
  - `DomainEvent.ts` - Base for domain events
  - `Response.ts` - Generic response interface
- **User Entity**: `src/Context/Users/Domain/User.ts`

### Dependency Injection
- **Container**: `src/Apps/Backend/DependencyInjection/Container.ts`
- Singleton pattern for CommandBus, QueryBus, and EventBus
- Handlers registered at startup

## File Structure
```
src/
├── Apps/
│   ├── Backend/
│   │   ├── DependencyInjection/
│   │   │   └── Container.ts
│   │   ├── Routes/
│   │   │   └── Index.ts
│   │   ├── Server.ts
│   │   └── Start.ts
│   └── CLI/
│       └── Index.ts
└── Context/
    ├── Shared/
    │   ├── Domain/
    │   │   ├── Bus/
    │   │   │   ├── CommandBus.ts
    │   │   │   ├── QueryBus.ts
    │   │   │   └── EventBus.ts
    │   │   ├── Commands/
    │   │   │   ├── Command.ts
    │   │   │   └── CommandHandler.ts
    │   │   ├── Queries/
    │   │   │   ├── Query.ts
    │   │   │   └── QueryHandler.ts
    │   │   ├── Events/
    │   │   │   ├── DomainEvent.ts
    │   │   │   └── DomainEventSubscriber.ts
    │   │   ├── AggregateRoot.ts
    │   │   └── Response.ts
    │   └── Infrastructure/
    │       └── Bus/
    │           ├── InMemoryCommandBus.ts
    │           ├── InMemoryQueryBus.ts
    │           ├── InMemoryEventBus.ts
    │           └── EventEmitterEventBus.ts
    └── Users/
        ├── Application/
        │   ├── Commands/
        │   │   └── RegisterUser/
        │   │       ├── RegisterUserCommand.ts
        │   │       └── RegisterUserCommandHandler.ts
        │   └── Queries/
        │       ├── GetAllUsers/
        │       │   ├── GetAllUsersQuery.ts
        │       │   └── GetAllUsersQueryHandler.ts
        │       └── GetUserById/
        │           ├── GetUserByIdQuery.ts
        │           └── GetUserByIdQueryHandler.ts
        └── Domain/
            ├── User.ts
            └── Events/
                └── UserRegisteredDomainEvent.ts
```

## API Endpoints

### Users API
| Method | Endpoint | Handler | Status |
|--------|----------|---------|-------|
| POST | /users | RegisterUserCommand | Working (hardcoded values) |
| GET | /users/:id | GetUserByIdQuery | Working |

## Known Issues (v1.0.0)
1. **CRITICAL**: Route hardcodes credentials in `Routes/Index.ts:13-16`
2. **CRITICAL**: No input validation - missing validation middleware
3. **CRITICAL**: GetUserByIdQuery doesn't store userId parameter
4. **HIGH**: User entity doesn't extend AggregateRoot
5. **HIGH**: Repository not implemented - handlers receive empty `{}`
6. **HIGH**: Type safety issues - using `object` instead of interfaces
7. **MEDIUM**: tsconfig.json has incorrect path alias (`@/shared`)

## Extending the Project

### Adding New Commands
1. Create command class extending `Command` in `src/Context/{Context}/Application/Commands/{Name}/`
2. Create handler implementing `CommandHandler` interface
3. Register handler in `Container.ts`

### Adding New Queries
1. Create query class extending `Query`
2. Create handler implementing `QueryHandler` interface
3. Register handler in `Container.ts`

### Adding New Bounded Context
1. Create folder `src/Context/{NewContext}/`
2. Add Application layer with Commands/ and Queries/
3. Add Domain layer with entities and events
4. Register handlers in Container

## Skills Available
- `.agents/skills/nodejs-express-server/` - Express server best practices
- `.agents/skills/nodejs-backend-patterns/` - Backend architecture patterns
- `.agents/skills/typescript-advanced-types/` - TypeScript advanced types
- `.agents/skills/nodejs-best-practices/` - Node.js best practices