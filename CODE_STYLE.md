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

### 3.3 Queries sin Parámetros
Algunas queries no requieren parámetros (ej: listar todos los usuarios).

```typescript
// ✅ Query sin parámetros - constructor vacío
import { Query } from '@/Context/Shared/Domain/Queries/Query';

export class GetAllUsersQuery extends Query {
  constructor() {
    super();
  }
}

// ✅ QueryHandler correspondiente
import { QueryHandler } from '@/Context/Shared/Domain/Queries/QueryHandler';
import { Response } from '@/Context/Shared/Domain/Response';

export class GetAllUsersQueryHandler
  implements QueryHandler<GetAllUsersQuery, GetAllUsersResponse[]>
{
  constructor(private readonly userRepository: UserRepository) {}

  async handle(query: GetAllUsersQuery): Promise<GetAllUsersResponse[]> {
    const users = await this.userRepository.findAll();
    return users.map(user => new GetAllUsersResponse(
      user.id,
      user.email,
      user.createdAt,
      user.updatedAt,
    ));
  }

  subscribedTo(): new (...args: any[]) => GetAllUsersQuery {
    return GetAllUsersQuery;
  }
}
```

### 3.4 Errores Comunes en Queries
```typescript
// ❌ NO hacer esto - Query con parámetros pero no los almacena
export class GetUserByIdQuery extends Query {
  constructor(userId: string) { // userId NO se guarda como propiedad
    super();
  }
}

// ✅ CORRECTO - Almacena el parámetro
export class GetUserByIdQuery extends Query {
  constructor(public readonly userId: string) {
    super();
  }
}
```

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

### 7.3 EventBus (Interfaz)
El `EventBus` es la interfaz para publicar eventos del dominio.

```typescript
// src/Context/Shared/Domain/Bus/EventBus.ts
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export interface EventBus {
  publish(events: DomainEvent[]): void;
}
```

### 7.4 DomainEventSubscriber (Suscriptores)
Los suscriptores listen eventos específicos y responden a ellos.

```typescript
// src/Context/Shared/Domain/Events/DomainEventSubscriber.ts
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export interface DomainEventSubscriber<T extends DomainEvent> {
  // Retorna el tipo de evento al que está suscrito
  subscribedTo(): new (...args: any[]) => T;
  // Método que se ejecuta cuando ocurre el evento
  on(event: T): Promise<void>;
}
```

### 7.5 Ejemplo de Suscriptor de Eventos
```typescript
import { DomainEventSubscriber } from '@/Context/Shared/Domain/Events/DomainEventSubscriber';
import { UserRegisteredDomainEvent } from '../Events/UserRegisteredDomainEvent';

export class SendWelcomeEmailSubscriber
  implements DomainEventSubscriber<UserRegisteredDomainEvent>
{
  async on(event: UserRegisteredDomainEvent): Promise<void> {
    // Lógica para enviar email de bienvenida
    console.log(`📧 Enviando email a: ${event.email}`);
  }

  subscribedTo(): new (...args: any[]) => UserRegisteredDomainEvent {
    return UserRegisteredDomainEvent;
  }
}
```

### 7.6 Registro de Suscriptores en EventBus
```typescript
import { EventEmitterEventBus } from '@/Context/Shared/Infrastructure/Bus/EventEmitterEventBus';

const eventBus = new EventEmitterEventBus();
const sendWelcomeEmail = new SendWelcomeEmailSubscriber();

eventBus.addSubscriber(sendWelcomeEmail);
```

---

## 8. Entidades (Domain)

### 8.1 Estructura Base
Las entidades que disparan eventos deben extender `AggregateRoot`.

> **Importante**: En este proyecto **NO se usan getters y setters**. Las propiedades son públicas y directas.

```typescript
import { AggregateRoot } from '@/Context/Shared/Domain/AggregateRoot';
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export class User extends AggregateRoot {
  // Propiedades públicas directas - NO getters/setters
  public readonly id: string;
  public readonly email: string;
  public readonly createdAt: Date;

  private constructor(
    id: string,
    email: string,
    createdAt: Date,
  ) {
    super();
    this.id = id;
    this.email = email;
    this.createdAt = createdAt;
  }

  static create(email: string): User {
    return new User(
      crypto.randomUUID(),
      email,
      new Date(),
    );
    // Para publicar eventos: user.addDomainEvent(new UserCreatedDomainEvent(...))
  }
}
```

### 8.2 Reglas
- ✅ Extender `AggregateRoot` si necesita publicar eventos
- ✅ Usar factory method estático (`create`) para instanciación
- ✅ **NO usar getters/setters** - propiedades públicas directas
- ✅ Usar `crypto.randomUUID()` para IDs
- ✅ Constructor privado (para usar con factory method)

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

### 13.3 Anti-Patterns Críticos (Known Issues del Proyecto)
Estos errores han ocurrido en el proyecto y deben evitarse a toda costa.

```typescript
// ❌ CRÍTICO - Credenciales hardcodeadas en la ruta
router.post('/users', async (req, res) => {
  const command = new RegisterUserCommand(
    'ronald@cqrs.com',          // ❌ NO hardcodear
    'mi_contraseña_super_fuerte' // ❌ NO hardcodear
  );
  await commandBus.dispatch(command);
});

// ✅ CORRECTO - Usar req.body
router.post('/users', async (req, res) => {
  const { email, password } = req.body;
  const command = new RegisterUserCommand(email, password);
  await commandBus.dispatch(command);
});
```

```typescript
// ❌ CRÍTICO - Usar commandBus para queries (error de тип)
router.get('/users/:id', async (req, res) => {
  const commandBus = Container.queryBus(); // ❌ WRONG BUS
  const data = await commandBus.ask(query);
});

// ✅ CORRECTO - Usar queryBus para queries
router.get('/users/:id', async (req, res) => {
  const queryBus = Container.queryBus(); // ✅ CORRECT BUS
  const data = await queryBus.ask(query);
});
```

```typescript
// ❌ CRÍTICO - Entidad no extiende AggregateRoot cuando dispara eventos
export default class User { // ❌ NO extiende AggregateRoot
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly email: string,
  ) {}
}

// ✅ CORRECTO - Extender AggregateRoot si publica eventos
import { AggregateRoot } from '@/Context/Shared/Domain/AggregateRoot';

export class User extends AggregateRoot {
  private constructor(
    private readonly _id: string,
    private readonly _email: string,
  ) {
    super();
  }
  
  static create(email: string): User {
    const user = new User(crypto.randomUUID(), email);
    // Aquí se pueden registrar eventos: user.addDomainEvent(...)
    return user;
  }
}
```

```typescript
// ❌ HIGH - Repository no implementado (pasar interfaz, no objeto vacío)
constructor(private readonly userRepository: object) {} // ❌ NO

// ✅ CORRECTO - Usar la interfaz del Repository
constructor(private readonly userRepository: UserRepository) {}
```

```typescript
// ❌ HIGH - Type safety con 'object' en vez de interfaces
constructor(private readonly config: object) {} // ❌ NO

// ✅ CORRECTO - Usar interfaces específicas
interface AppConfig {
  port: number;
  environment: string;
}

constructor(private readonly config: AppConfig) {}
```


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

## 17. Implementaciones InMemory de los Buses

### 17.1 InMemoryCommandBus
Implementación que registra handlers y los ejecuta por nombre de comando.

```typescript
// src/Context/Shared/Infrastructure/Bus/InMemoryCommandBus.ts
import { CommandBus } from '@/Context/Shared/Domain/Bus/CommandBus';
import { CommandHandler } from '@/Context/Shared/Domain/Commands/CommandHandler';
import { Command } from '@/Context/Shared/Domain/Commands/Command';

export class InMemoryCommandBus implements CommandBus {
  private handlers: Map<string, CommandHandler<Command>> = new Map();

  constructor(handlers: CommandHandler<Command>[]) {
    handlers.forEach((handler) => {
      // Registra el handler por nombre de la clase Command
      this.handlers.set(handler.subscribedTo().name, handler);
    });
  }

  async dispatch(command: Command): Promise<void> {
    const handler = this.handlers.get(command.constructor.name);
    if (!handler) {
      throw new Error(
        `No handler registered for command: ${command.constructor.name}`,
      );
    }
    await handler.handle(command);
  }
}
```

### 17.2 InMemoryQueryBus
Implementación similar al CommandBus pero para queries.

```typescript
// src/Context/Shared/Infrastructure/Bus/InMemoryQueryBus.ts
import { QueryBus } from '@/Context/Shared/Domain/Bus/QueryBus';
import { QueryHandler } from '@/Context/Shared/Domain/Queries/QueryHandler';
import { Query } from '@/Context/Shared/Domain/Queries/Query';
import { Response } from '@/Context/Shared/Domain/Response';

export class InMemoryQueryBus implements QueryBus {
  private handlers: Map<string, QueryHandler<Query, Response>> = new Map();

  constructor(handlers: QueryHandler<Query, Response>[]) {
    handlers.forEach((handler) => {
      this.handlers.set(handler.subscribedTo().name, handler);
    });
  }

  async ask<T extends Response>(query: Query): Promise<T> {
    const handler = this.handlers.get(query.constructor.name);
    if (!handler) {
      throw new Error(
        `No handler registered for query: ${query.constructor.name}`,
      );
    }
    return await handler.handle(query) as T;
  }
}
```

### 17.3 EventEmitterEventBus
Implementación de EventBus usando el módulo `events` de Node.js.

```typescript
// src/Context/Shared/Infrastructure/Bus/EventEmitterEventBus.ts
import { EventEmitter } from 'events';
import { EventBus } from '@/Context/Shared/Domain/Bus/EventBus';
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';
import { DomainEventSubscriber } from '@/Context/Shared/Domain/Events/DomainEventSubscriber';

export class EventEmitterEventBus implements EventBus {
  private emitter: EventEmitter;

  constructor() {
    this.emitter = new EventEmitter();
  }

  publish(events: DomainEvent[]): void {
    for (const event of events) {
      // Emite el evento usando el nombre de la clase
      this.emitter.emit(event.constructor.name, event);
    }
  }

  addSubscriber(subscriber: DomainEventSubscriber<DomainEvent>): void {
    const eventClass = subscriber.subscribedTo();
    this.emitter.on(eventClass.name, (event: DomainEvent) => {
      // Ejecuta el handler de forma asíncrona
      setImmediate(async () => {
        try {
          await subscriber.on(event);
        } catch (error) {
          console.error(`❌ [${event.constructor.name}] Subscriber error:`, error);
        }
      });
    });
  }
}
```

### 17.4 Reglas
- ✅ Los buses in-memory son para desarrollo/testing
- ✅ Los handlers se registran en el constructor
- ✅ Se usa `subscribedTo().name` como clave del Map
- ✅ El `EventEmitterEventBus` soporta múltiples suscriptores por evento

---

## 18. Aplicación CLI

### 18.1 Estructura Base
El proyecto incluye una aplicación CLI que reutiliza los mismos Commands y Queries del backend.

```typescript
// src/Apps/CLI/Index.ts
import { Container } from '@/Apps/Backend/DependencyInjection/Container';
import { RegisterUserCommand } from '@/Context/Users/Application/Commands/RegisterUser/RegisterUserCommand';
import { GetAllUsersQuery } from '@/Context/Users/Application/Queries/GetAllUsers/GetAllUsersQuery';
import { UserResponse } from '@/Context/Users/Application/Queries/GetAllUsers/GetAllUsersQueryHandler';

const args = process.argv.slice(2);
const command = args[0];

async function main() {
  switch (command) {
    case 'create':
      const email = args[1];
      const password = args[2];

      if (!email || !password) {
        console.error('❌ Usage: cli.js create <email> <password>');
        process.exit(1);
      }

      await Container.commandBus().dispatch(
        new RegisterUserCommand(email, password),
      );
      console.log(`✅ Usuario ${email} creado`);
      break;

    case 'list':
      const users = await Container.queryBus().ask<UserResponse[]>(
        new GetAllUsersQuery(),
      );
      console.log('📋 Usuarios:');
      users.forEach((user: UserResponse) => {
        console.log(`  - ${user.email} (id: ${user.id})`);
      });
      break;

    default:
      console.log(`
🔧 CQRS CLI

Usage:
  npm run cli -- create <email> <password>  Create a new user
  npm run cli -- list                       List all users

Examples:
  npm run cli -- create admin@test.com mypassword123
  npm run cli -- list
`);
  }
}

main().catch((error) => {
  console.error('❌ Error:', error.message);
  process.exit(1);
});
```

### 18.2 Reglas
- ✅ Reutilizar el mismo `Container` del backend
- ✅ Usar `commandBus.dispatch()` para Commands
- ✅ Usar `queryBus.ask<T>()` para Queries
- ✅ Validar argumentos antes de ejecutar
- ✅ Usar `process.exit(1)` en errores
- ✅ Mensajes claros con emojis para UX

### 18.3 Extender la CLI
Para agregar nuevos comandos:

```typescript
case 'delete':
  const deleteEmail = args[1];
  if (!deleteEmail) {
    console.error('❌ Usage: cli.js delete <email>');
    process.exit(1);
  }
  await Container.commandBus().dispatch(
    new DeleteUserCommand(deleteEmail),
  );
  console.log(`✅ Usuario ${deleteEmail} eliminado`);
  break;
```
---

## 19. Guía de Extensión del Proyecto

### 19.1 Agregar un nuevo Command

**Paso 1**: Crear la clase Command
```typescript
// src/Context/Users/Application/Commands/DeleteUser/DeleteUserCommand.ts
import { Command } from '@/Context/Shared/Domain/Commands/Command';

export class DeleteUserCommand extends Command {
  constructor(public readonly userId: string) {
    super();
  }
}
```

**Paso 2**: Crear el Handler
```typescript
// src/Context/Users/Application/Commands/DeleteUser/DeleteUserCommandHandler.ts
import { CommandHandler } from '@/Context/Shared/Domain/Commands/CommandHandler';
import { DeleteUserCommand } from './DeleteUserCommand';

export class DeleteUserCommandHandler
  implements CommandHandler<DeleteUserCommand>
{
  constructor(private readonly userRepository: UserRepository) {}

  async handle(command: DeleteUserCommand): Promise<void> {
    await this.userRepository.delete(command.userId);
  }

  subscribedTo(): new (...args: any[]) => DeleteUserCommand {
    return DeleteUserCommand;
  }
}
```

**Paso 3**: Registrar en el Container
```typescript
// src/Apps/Backend/DependencyInjection/Container.ts
const deleteUserHandler = new DeleteUserCommandHandler(userRepository);
this._commandBus = new InMemoryCommandBus([
  registerUserHandler,
  deleteUserHandler, // ← Agregar aquí
]);
```

### 19.2 Agregar una nueva Query

**Paso 1**: Crear Query + Response DTO (en el mismo archivo)
```typescript
// src/Context/Users/Application/Queries/GetUserByEmail/GetUserByEmailQuery.ts
import { Query } from '@/Context/Shared/Domain/Queries/Query';
import { Response } from '@/Context/Shared/Domain/Response';

export class GetUserByEmailResponse implements Response {
  constructor(
    public readonly id: string,
    public readonly email: string,
  ) {}
}

export class GetUserByEmailQuery extends Query {
  constructor(public readonly email: string) {
    super();
  }
}
```

**Paso 2**: Crear el Handler
```typescript
// src/Context/Users/Application/Queries/GetUserByEmail/GetUserByEmailQueryHandler.ts
import { QueryHandler } from '@/Context/Shared/Domain/Queries/QueryHandler';
import { GetUserByEmailQuery } from './GetUserByEmailQuery';
import { GetUserByEmailResponse } from './GetUserByEmailQuery';

export class GetUserByEmailQueryHandler
  implements QueryHandler<GetUserByEmailQuery, GetUserByEmailResponse>
{
  constructor(private readonly userRepository: UserRepository) {}

  async handle(query: GetUserByEmailQuery): Promise<GetUserByEmailResponse> {
    const user = await this.userRepository.findByEmail(query.email);
    if (!user) {
      throw new Error('User not found');
    }
    return new GetUserByEmailResponse(user.id, user.email);
  }

  subscribedTo(): new (...args: any[]) => GetUserByEmailQuery {
    return GetUserByEmailQuery;
  }
}
```

**Paso 3**: Registrar en el Container
```typescript
// src/Apps/Backend/DependencyInjection/Container.ts
const getUserByEmailHandler = new GetUserByEmailQueryHandler(userRepository);
this._queryBus = new InMemoryQueryBus([
  getUserByIdHandler,
  getAllUsersHandler,
  getUserByEmailHandler, // ← Agregar aquí
]);
```

### 19.3 Agregar un nuevo Bounded Context

**Estructura de carpetas:**
```
src/Context/Orders/
├── Application/
│   ├── Commands/
│   │   └── CreateOrder/
│   │       ├── CreateOrderCommand.ts
│   │       └── CreateOrderCommandHandler.ts
│   └── Queries/
│       └── GetOrders/
│           ├── GetOrdersQuery.ts
│           └── GetOrdersQueryHandler.ts
├── Domain/
│   ├── Order.ts
│   ├── OrderRepository.ts      ← Interfaz
│   └── Events/
│       └── OrderCreatedDomainEvent.ts
└── Infrastructure/
    └── Persistence/
        └── InMemoryOrderRepository.ts
```

**Pasos:**
1. Crear carpeta `src/Context/{ContextName}/`
2. Agregar carpeta `Application/` con Commands/ y Queries/
3. Agregar carpeta `Domain/` con entidad, interfaz de repositorio y eventos
4. Implementar el repositorio en `Infrastructure/Persistence/`
5. Registrar handlers en el Container

### 19.4 Agregar un nuevo Domain Event

**Paso 1**: Crear el evento
```typescript
// src/Context/Users/Domain/Events/UserDeletedDomainEvent.ts
import { DomainEvent } from '@/Context/Shared/Domain/Events/DomainEvent';

export class UserDeletedDomainEvent extends DomainEvent {
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

**Paso 2**: Disparar el evento desde la entidad
```typescript
// En la entidad
import { UserDeletedDomainEvent } from './Events/UserDeletedDomainEvent';

static delete(user: User): void {
  user.addDomainEvent(new UserDeletedDomainEvent(user.id, user.email));
  // lógica de eliminación
}
```

**Paso 3**: Crear suscriptor
```typescript
// src/Context/Users/Application/Subscribers/UserDeletedSubscriber.ts
import { DomainEventSubscriber } from '@/Context/Shared/Domain/Events/DomainEventSubscriber';
import { UserDeletedDomainEvent } from '../Domain/Events/UserDeletedDomainEvent';

export class UserDeletedSubscriber
  implements DomainEventSubscriber<UserDeletedDomainEvent>
{
  async on(event: UserDeletedDomainEvent): Promise<void> {
    console.log(`🗑️ Usuario eliminado: ${event.email}`);
  }

  subscribedTo(): new (...args: any[]) => UserDeletedDomainEvent {
    return UserDeletedDomainEvent;
  }
}
```
---

## 20. Validación de Input

### 20.1 Importancia
El proyecto actualmente **no tiene validación de input** ( Known Issue ). Esto es crítico para:
- Prevenir datos inválidos en la base de datos
- Proteger contra ataques de inyección
- Proporcionar feedback claro al usuario

### 20.2 Recomendación: Zod
Se recomienda usar **Zod** para validación por su integración con TypeScript.

```bash
npm install zod
```

### 20.3 Validación de Commands
```typescript
// src/Context/Users/Application/Commands/RegisterUser/RegisterUserCommand.ts
import { Command } from '@/Context/Shared/Domain/Commands/Command';
import { z } from 'zod';

// Schema de validación
const RegisterUserSchema = z.object({
  email: z.string().email('Email inválido'),
  password: z.string().min(8, 'Password debe tener al menos 8 caracteres'),
});

export class RegisterUserCommand extends Command {
  public readonly email: string;
  public readonly password: string;

  constructor(input: unknown) {
    super();
    
    // Validar y tipar el input
    const parsed = RegisterUserSchema.parse(input);
    this.email = parsed.email;
    this.password = parsed.password;
  }
}
```

### 20.4 Validación en Rutas Express
```typescript
// src/Apps/Backend/Routes/Index.ts
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

router.post('/users', async (req, res, next) => {
  try {
    // Validar antes de crear el Command
    const parsed = CreateUserSchema.safeParse(req.body);
    
    if (!parsed.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: parsed.error.flatten(),
      });
    }

    const { email, password } = parsed.data;
    await commandBus.dispatch(new RegisterUserCommand({ email, password }));
    
    return res.status(201).json({ message: 'User created' });
  } catch (error) {
    next(error);
  }
});
```

### 20.5 Reglas de Validación
- ✅ Validar en la ruta antes de crear el Command
- ✅ Commands también pueden autovalidarse con Zod
- ✅ Usar `safeParse()` para evitar excepciones en validación
- ✅ Devolver errores 400 con detalles claros
- ✅ NO confiar en validación del cliente únicamente

### 20.6 Errores de Validación Comunes
```typescript
// ❌ NO hacer esto - Confiar ciegamente en req.body
const { email, password } = req.body;
const command = new RegisterUserCommand(email, password);

// ✅ HACER esto - Validar primero
const parsed = CreateUserSchema.safeParse(req.body);
if (!parsed.success) {
  return res.status(400).json({ error: parsed.error.message });
}
const { email, password } = parsed.data;
```

---

## 21. Resumen de Reglas Clave (Actualizado)

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
| DomainEvents extienen `DomainEvent` | ✅ |
| Repository con interfaz, no implementación en handler | ✅ |
| **Usar queryBus para queries, commandBus para commands** | ✅ |
| **NO hardcodear credenciales en rutas** | ✅ |
| **Validar input con Zod** | ⚠️ Recomendado |
| **CLI reusable con los mismos Commands/Queries** | ✅ |

---

## 22. Checklist de Código Review (Actualizado)

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
- [ ] ¿Se usa `queryBus` para queries y `commandBus` para commands?
- [ ] ¿Hay credenciales hardcodeadas en las rutas?
- [ ] ¿El input está validado con Zod (recomendado)?
