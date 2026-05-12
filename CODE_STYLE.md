# Code Style - Node.js CQRS-DDD

Este documento establece las reglas de estilo y convenciones para el proyecto.

---

## 1. Tipado Fuerte (TypeScript)

### 1.1 Regla General
- **SIEMPRE** usar tipos explícitos. Nunca usar `any`.
- Usar tipos primitivos, interfaces o clases concretas.
- Evitar `object` como tipo.

### 1.2 Tipos Permitidos
```typescript
// ✅ Correcto
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// ❌ Incorrecto - Sin tipado
constructor(private readonly userRepository: object) {}
```

### 1.3 Propiedades en Constructores
```typescript
// ✅ Correcto - Usar public readonly
constructor(
  public readonly id: string,
  public readonly email: string,
) {}

// ❌ Incorrecto - Sin tipo explícito
constructor(id, email) {}
```

---

## 2. Commands

### 2.1 Estructura Base
Todo Command debe extender la clase base `Command`.

```typescript
import { Command } from '@/Context/Shared/Domain/Commands/Command';

export class RegisterUserCommand extends Command {
  constructor(
    public readonly email: string,
    public readonly password: string,
  ) {
    super();
  }
}
```

### 2.2 Reglas
- ✅ Extender `Command`
- ✅ Usar `public readonly` en propiedades del constructor
- ✅ Llamar `super()` en constructor
- ✅ Nombre termina en `Command`

---

## 3. Queries

### 3.1 Estructura Base
Toda Query debe extender la clase base `Query`.

```typescript
import { Query } from '@/Context/Shared/Domain/Queries/Query';

export class GetUserByIdQuery extends Query {
  constructor(public readonly userId: string) {
    super();
  }
}
```

### 3.2 Reglas
- ✅ Extender `Query`
- ✅ Almacenar TODOS los parámetros en propiedades `public readonly`
- ✅ Llamar `super()` en constructor
- ✅ Nombre termina en `Query`

---

## 4. Command Handlers

### 4.1 Estructura Base
Todo CommandHandler debe implementar la interfaz `CommandHandler`.

```typescript
import { CommandHandler } from '@/Context/Shared/Domain/Commands/CommandHandler';
import { RegisterUserCommand } from './RegisterUserCommand';

export class RegisterUserCommandHandler
  implements CommandHandler<RegisterUserCommand>
{
  constructor(private readonly userRepository: UserRepository) {}

  async handle(command: RegisterUserCommand): Promise<void> {
    // Lógica del handler
  }

  subscribedTo(): new (...args: any[]) => RegisterUserCommand {
    return RegisterUserCommand;
  }
}
```

### 4.2 Reglas
- ✅ Implementar `CommandHandler<ConcreteCommand>`
- ✅ Constructor con dependencias tipadas (NO `object`)
- ✅ Método `handle()` debe ser `async` y retornar `Promise<void>`
- ✅ Método `subscribedTo()` retorna constructor de la command

---

## 5. Query Handlers

### 5.1 Estructura Base
Todo QueryHandler debe implementar la interfaz `QueryHandler`.

```typescript
import { QueryHandler } from '@/Context/Shared/Domain/Queries/QueryHandler';
import { Query } from '@/Context/Shared/Domain/Queries/Query';
import { Response } from '@/Context/Shared/Domain/Response';

export class GetUserByIdQueryHandler
  implements QueryHandler<GetUserByIdQuery, GetUserByIdResponse>
{
  constructor(private readonly userRepository: UserRepository) {}

  async handle(query: GetUserByIdQuery): Promise<GetUserByIdResponse> {
    // Lógica del handler
  }

  subscribedTo(): new (...args: any[]) => GetUserByIdQuery {
    return GetUserByIdQuery;
  }
}
```

### 5.2 Reglas
- ✅ Implementar `QueryHandler<ConcreteQuery, ConcreteResponse>`
- ✅ Constructor con dependencias tipadas (NO `object`)
- ✅ Método `handle()` debe ser `async` y retornar `Promise<Response>`
- ✅ Método `subscribedTo()` retorna constructor de la query

---

## 6. DTOs y Responses

### 6.1 Ubicación - REGLA CRÍTICA
**Los DTOs deben estar en el MISMO ARCHIVO que la Query.**

```
src/Context/Users/Application/Queries/GetUserById/
├── GetUserByIdQuery.ts      ← Query + Response DTO
└── GetUserByIdQueryHandler.ts
```

### 6.2 Estructura del Archivo Query
```typescript
import { Query } from '@/Context/Shared/Domain/Queries/Query';
import { Response } from '@/Context/Shared/Domain/Response';

// ✅ DTO en el mismo archivo
export class GetUserByIdResponse implements Response {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly createdAt: Date,
    public readonly updatedAt: Date,
  ) {}
}

// ✅ Query en el mismo archivo
export class GetUserByIdQuery extends Query {
  constructor(public readonly userId: string) {
    super();
  }
}
```

### 6.3 Reglas
- ✅ DTO debe implementar `Response`
- ✅ DTO y Query en el mismo archivo
- ✅ Nombre del DTO: `{QueryName}Response` (ej: `GetUserByIdResponse`)
- ✅ Nombre del DTO para arrays: `{QueryName}Response[]`

---

## 7. Domain Events

### 7.1 Estructura Base
Todo DomainEvent debe extender la clase base `DomainEvent`.

```typescript
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export class UserRegisteredDomainEvent extends DomainEvent {
  constructor(
    aggregateId: string,
    public readonly email: string,
  ) {
    super(aggregateId);
  }

  toPrimitives(): object {
    return {
      userId: this.aggregateId,
      email: this.email,
      occurredOn: this.occurredOn,
    };
  }
}
```

### 7.2 Reglas
- ✅ Extender `DomainEvent`
- ✅ Constructor recibe `aggregateId` y lo pasa a `super()`
- ✅ Implementar método `toPrimitives(): object`
- ✅ Nombre termina en `DomainEvent`

---

## 8. Entidades (Domain)

### 8.1 Estructura Base
Las entidades que disparan eventos deben extender `AggregateRoot`.

```typescript
import { AggregateRoot } from '@/Context/Shared/Domain/AggregateRoot';
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export class User extends AggregateRoot {
  private constructor(
    private readonly _id: string,
    private readonly _email: string,
    private _createdAt: Date,
  ) {
    super();
  }

  static create(email: string): User {
    const user = new User(
      crypto.randomUUID(),
      email,
      new Date(),
    );
    // Aquí se pueden registrar eventos
    return user;
  }

  get id(): string {
    return this._id;
  }

  get email(): string {
    return this._email;
  }

  pullDomainEvents(): DomainEvent[] {
    return super.pullDomainEvents();
  }
}
```

### 8.2 Reglas
- ✅ Extender `AggregateRoot` si necesita publicar eventos
- ✅ Usar factory method estático (`create`) parainstanciación
- ✅ Propiedades privadas con getters públicos
- ✅ Usar `crypto.randomUUID()` para IDs

---

## 9. Repositorios

### 9.1 Interfaz en Domain
```typescript
// src/Context/Users/Domain/UserRepository.ts
export interface UserRepository {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
  findAll(): Promise<User[]>;
}
```

### 9.2 Reglas
- ✅ Interfaz en carpeta Domain
- ✅ Nombre termina en `Repository`
- ✅ Métodos con tipos de retorno explícitos

---

## 10. Dependency Injection (Container)

### 10.1 Estructura
```typescript
export class Container {
  private static _commandBus: CommandBus;
  private static _queryBus: QueryBus;
  private static _eventBus: EventBus;

  static commandBus(): CommandBus {
    if (!this._commandBus) {
      const userRepository = new InMemoryUserRepository();
      const registerUserHandler = new RegisterUserCommandHandler(userRepository);

      this._commandBus = new InMemoryCommandBus([registerUserHandler]);
    }
    return this._commandBus;
  }

  static queryBus(): QueryBus { /* ... */ }
  static eventBus(): EventBus { /* ... */ }
}
```

### 10.2 Reglas
- ✅ Usar singleton estático
- ✅ Pasar dependencias concretas al handler
- ✅ NO pasar `{}` vacío

---

## 11. Rutas (Express)

### 11.1 Estructura
```typescript
export function registerRoutes(app: Express): void {
  const router = Router();

  router.post('/users', async (req, res, next) => {
    try {
      const { email, password } = req.body;

      if (!email || !password) {
        return res.status(400).json({ error: 'Email and password required' });
      }

      const commandBus = Container.commandBus();
      await commandBus.dispatch(new RegisterUserCommand(email, password));

      return res.status(201).json({ message: 'User created' });
    } catch (error) {
      next(error);
    }
  });

  router.get('/users/:id', async (req, res, next) => {
    try {
      const queryBus = Container.queryBus();
      const response = await queryBus.ask<GetUserByIdResponse>(
        new GetUserByIdQuery(req.params.id),
      );

      return res.status(200).json(response);
    } catch (error) {
      next(error);
    }
  });

  app.use(router);
}
```

### 11.2 Reglas
- ✅ Usar `async/await` con try-catch
- ✅ Usar `next(error)` para manejo de errores
- ✅ Validar `req.body` antes de crear comando
- ✅ Usar `queryBus` para queries, `commandBus` para commands
- ✅ Retornar respuesta JSON explícita

---

## 12. Estructura de Archivos

### 12.1 Estructura Obligatoria
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
│
└── Context/
    ├── Shared/                     ← NO usar "shared" en minúsculas
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
    │
    └── {ContextName}/              ← PascalCase (Users, Courses, etc.)
        ├── Application/
        │   ├── Commands/
        │   │   └── {Action}/
        │   │       ├── {Action}Command.ts
        │   │       └── {Action}CommandHandler.ts
        │   └── Queries/
        │       └── {Action}/
        │           ├── {Action}Query.ts      ← Query + Response DTO
        │           └── {Action}QueryHandler.ts
        ├── Domain/
        │   ├── {EntityName}.ts
        │   ├── {EntityName}Repository.ts     ← Interfaz
        │   └── Events/
        │       └── {Action}DomainEvent.ts
        └── Infrastructure/
            └── Persistence/
                └── {Type}{EntityName}Repository.ts
```

### 12.2 Reglas de Nombres
| Elemento | Naming | Ejemplo |
|----------|--------|---------|
| Command | PascalCase + Command | RegisterUserCommand |
| Query | PascalCase + Query | GetUserByIdQuery |
| Command Handler | PascalCase + CommandHandler | RegisterUserCommandHandler |
| Query Handler | PascalCase + QueryHandler | GetUserByIdQueryHandler |
| Response DTO | PascalCase + Response | GetUserByIdResponse |
| Domain Event | PascalCase + DomainEvent | UserRegisteredDomainEvent |
| Entity | PascalCase (singular) | User, Course |
| Repository Interface | PascalCase + Repository | UserRepository |

---

## 13. Errores Comunes a Evitar

### 13.1 NO hacer esto
```typescript
// ❌ Query no almacena parámetro
export class GetUserByIdQuery extends Query {
  constructor(userId: string) { // userId NO se guarda
    super();
  }
}

// ❌ DTO en archivo separado
// GetUserByIdResponse.ts  <- NO

// ❌ Repository como object
constructor(private userRepository: object) {}

// ❌ Handlers hardcodeados
const command = new RegisterUserCommand('hardcoded@email.com', 'password');
```

### 13.2 SIEMPRE hacer esto
```typescript
// ✅ Query almacena parámetro
export class GetUserByIdQuery extends Query {
  constructor(public readonly userId: string) {
    super();
  }
}

// ✅ DTO en mismo archivo
export class GetUserByIdResponse implements Response { }

// ✅ Repository con tipo interfaz
constructor(private readonly userRepository: UserRepository) {}

// ✅ Usar req.body
const { email, password } = req.body;
const command = new RegisterUserCommand(email, password);
```

---

## 14. tsconfig.json

### 14.1 Path Aliases Correctos
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "bundler",
    "outDir": "./build",
    "rootDir": "./src",
    "paths": {
      "@/*": ["./src/*"],
      "@/Apps/*": ["./src/Apps/*"],
      "@/Context/*": ["./src/Context/*"],
      "@/Context/Shared/*": ["./src/Context/Shared/*"]
    },
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

### 14.2 Reglas
- ✅ NO usar `./src/shared` (ruta inexistente)
- ✅ Usar `@/Context/Shared/*` para acceder al Shared kernel

---

## 15. Resumen de Reglas Clave

| Regla | Obligatorio |
|-------|-------------|
| Commands extienden `Command` | ✅ |
| Queries extienden `Query` | ✅ |
| CommandHandlers implementan `CommandHandler` | ✅ |
| QueryHandlers implementan `QueryHandler` | ✅ |
| DTOs implementan `Response` | ✅ |
| DTO en mismo archivo que Query | ✅ |
| Tipos explícitos (no `any` ni `object`) | ✅ |
| Propiedades en constructor con `public readonly` | ✅ |
| Validar `req.body` en rutas | ✅ |
| Try-catch en rutas | ✅ |
| Entidades que disparan eventos extienden `AggregateRoot` | ✅ |
| DomainEvents extienden `DomainEvent` | ✅ |
| Repository con interfaz, no implementación en handler | ✅ |

---

## 16. Checklist de Código Review

Antes de hacer commit, verificar:

- [ ] ¿El Command extiende `Command`?
- [ ] ¿La Query extiende `Query` y guarda sus parámetros?
- [ ] ¿El CommandHandler implementa `CommandHandler`?
- [ ] ¿El QueryHandler implementa `QueryHandler`?
- [ ] ¿El DTO está en el mismo archivo que la Query?
- [ ] ¿El DTO implementa `Response`?
- [ ] ¿Las dependencias del handler tienen tipos (no `object`)?
- [ ] ¿Las rutas validan `req.body`?
- [ ] ¿Las rutas usan try-catch con `next(error)`?
- [ ] ¿Las entidades que disparan eventos extienden `AggregateRoot`?
- [ ] ¿Los DomainEvents extienden `DomainEvent`?