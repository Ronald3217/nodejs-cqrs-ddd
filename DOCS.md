# CQRS: Aprendiendo CQRS en Node.js (CQRS, Hexagonal y DDD)

## Diagrama para Usuarios
```bash
src/
└── Context/
    └── Users/
        ├── Application/
        │   ├── Commands/
        │   │   └── RegisterUser/
        │   │       ├── RegisterUserCommand.ts        <-- DTO: email, password
        │   │       └── RegisterUserCommandHandler.ts <-- Lógica de orquestación
        │   └── Queries/
        │       └── GetUserById/
        │           ├── GetUserByIdQuery.ts           <-- Query con filtro id
        │           └── GetUserByIdQueryHandler.ts    <-- Lógica de lectura
        ├── Domain/
        │   ├── User.ts             <-- Entidad (id: string, email: string, etc.)
        │   ├── UserRepository.ts   <-- Interfaz (Puerto)
        │   └── Events/
        │       └── UserRegisteredDomainEvent.ts
        └── Infrastructure/
            └── Persistence/
                └── MongoUserRepository.ts (Implementación del repositorio)
```

## Diagrama de Apps (Entry Points)
```bash
src/
└── Apps/
    ├── Backend/                        <-- Servidor HTTP (API REST)
    │   ├── DependencyInjection/
    │   │   └── Container.ts            <-- Composición de handlers y repos
    │   ├── Routes/
    │   │   └── Index.ts                <-- Rutas Express
    │   ├── Server.ts                   <-- Configuración del servidor
    │   └── Start.ts                    <-- Punto de entrada
    │
    └── CLI/                            <-- Interfaz de línea de comandos
        └── Index.ts                    <-- Punto de entrada CLI
```

### ¿Por qué el Container está en Backend?

El **Container** contiene la **composición específica** de cada aplicación:

```typescript
// En Container se configuran:
// - Los handlers concretos (RegisterUserCommandHandler)
// - Los repositorios (MongoUserRepository, MySqlUserRepository)
// - Las implementaciones de buses (InMemory, EventEmitter, RabbitMQ)
```

**Beneficios de no separar la inicialización:**

1. **Simplicidad**: Un solo lugar donde defines qué necesita tu app
2. **Consistencia**: La API y CLI usan la misma composición
3. **Mantenimiento**: Cuando agregues un handler, solo lo registras una vez
4. **Flexibilidad**: Cada app (Backend/CLI/Worker) puede tener su propio Container si necesita diferentes configuraciones

**Si tienes múltiples apps con diferentes necesidades:**
```typescript
// src/Apps/Backend/Container.ts   → API REST (HTTP handlers)
// src/Apps/CLI/Container.ts       → CLI (solo algunos commands)
// src/Apps/Worker/Container.ts   → Jobs (eventos, no HTTP)
```

### Ejemplo de uso del CLI
```bash
# Crear usuario
npm run cli -- create admin@test.com mypassword123

# Listar usuarios
npm run cli -- list
```

## Diagrama para Cursos
```bash
src/
└── Context/
    └── Courses/
        ├── Application/
        │   ├── Commands/
        │   │   └── CreateCourse/
        │   │       ├── CreateCourseCommand.ts        <-- DTO de entrada
        │   │       └── CreateCourseCommandHandler.ts <-- Lógica del "Caso de Uso"
        │   └── Queries/
        │       └── GetCourseById/
        │           ├── GetCourseByIdQuery.ts
        │           └── GetCourseByIdQueryHandler.ts
        ├── Domain/
        │   ├── Course.ts           <-- Entidad (usando UUID string plano)
        │   ├── CourseRepository.ts <-- Interfaz del puerto
        │   └── Events/             <-- Domain Events
        └── Infrastructure/
            └── Persistence/
                └── TypeOrmCourseRepository.ts
```

## Shared context Diagram
```bash
src/
└── Context/
    └── Shared/
        ├── Domain/
        │   ├── Commands/
        │   │   ├── Command.ts          <-- Clase base que extienden todos los Commands
        │   │   └── CommandHandler.ts   <-- Interfaz para handlers de comandos
        │   ├── Queries/
        │   │   ├── Query.ts            <-- Interfaz base para todas las Queries
        │   │   └── QueryHandler.ts     <-- Interfaz para handlers de queries
        │   ├── Bus/
        │   │   ├── CommandBus.ts        <-- Interfaz del bus de comandos (Puerto)
        │   │   ├── QueryBus.ts          <-- Interfaz del bus de queries (Puerto)
        │   │   └── EventBus.ts          <-- Interfaz del bus de eventos (Puerto)
        │   ├── Events/
        │   │   ├── DomainEvent.ts       <-- Clase base para todos los eventos de dominio
        │   │   └── DomainEventSubscriber.ts <-- Interfaz para suscriptores de eventos
        │   ├── AggregateRoot.ts         <-- Clase base para entidades que disparan eventos
        │   └── Response.ts              <-- Interfaz base para los DTOs de respuesta
        └── Infrastructure/
            └── Bus/
                ├── InMemoryCommandBus.ts   <-- Implementación del CommandBus
                ├── InMemoryQueryBus.ts     <-- Implementación del QueryBus
                ├── InMemoryEventBus.ts     <-- Implementación del EventBus (en memoria)
                └── EventEmitterEventBus.ts <-- Implementación del EventBus (EventEmitter)
```

## Flujo de un Command y CommandHandler
Este diagrama resume el "viaje" que realiza un comando desde que el usuario hace clic en un botón (o envía una petición API) hasta que el sistema ejecuta la lógica de negocio.
```bash
[ Capa de Infraestructura ]       [ Capa de Aplicación ]       [ Capa de Dominio ]
 +-----------------------+        +----------------------+      +------------------+
 |                       |        |                      |      |                  |
 |  1. CONTROLADOR API   |        |  3. BUS DE COMANDOS  |      |  5. REPOSITORIO  |
 |  (Express / Fastify)  |        |    (InMemory / MQ)   |      |   (TypeORM/Mongo)|
 |           |           |        |           |          |      |         ^        |
 |           v           |        |           v          |      |         |        |
 |  2. CREA EL COMMAND   |------->|  4. BUSCA EL HANDLER |----->|  6. PERSISTE     |
 | (DTO: Datos planos)   |        |  (Usa el Name Map)   |      |     LA ENTIDAD   |
 |                       |        |           |          |      |                  |
 +-----------------------+        +-----------|----------+      +------------------+
                                              |
                                              v
                                   [ 4.5 COMMAND HANDLER ]
                                   (Ejecuta Caso de Uso)
```
Los hitos del viaje:
El Disparador (Controlador): Es la puerta de entrada. Su única misión es validar que la petición HTTP sea correcta y transformarla en un objeto Command.

El Mensajero (Bus): El controlador le entrega el comando al CommandBus y le dice: "Despacha esto". El controlador ya no sabe qué pasará después (desacoplamiento).

El Mapa (Registry): El Bus mira su mapa interno (donde registramos los handlers con constructor.name) y encuentra quién es el experto para ese comando específico.

El Ejecutor (Handler): El Bus llama al método handle() del experto. Aquí es donde se "abre" el comando, se comunica con el Repositorio y se cambia el estado del sistema (se guarda en la base de datos).

Repasar este flujo te ayudará a ver que el Command es simplemente un mensaje viajando por una tubería organizada. ¡Hasta la próxima!

## Flujo de un Query y QueryHandler
El flujo de una Query es similar al del Command, pero con una diferencia crítica: el camino de vuelta. Mientras que el Command es una orden de "disparar y olvidar", la Query es una conversación donde esperas una respuesta específica.

Aquí tienes el diagrama del flujo para que lo compares con el anterior:

```bash
[ Capa de Infraestructura ]       [ Capa de Aplicación ]       [ Capa de Dominio ]
 +-----------------------+        +----------------------+      +------------------+
 |                       |        |                      |      |                  |
 |  1. CONTROLADOR API   |        |  3. BUS DE QUERIES   |      |  5. MODELO LEER  |
 |     (GET Request)     |        |      (InMemory)      |      |   (Query Model)  |
 |           |           |        |           |          |      |         |        |
 |           v           |        |           v          |      |         v        |
 |  2. CREA LA QUERY     |------->|  4. BUSCA EL HANDLER |----->|  6. RECUPERA     |
 |  (Ej: FindUserQuery)  |        |  (Usa el Name Map)   |      |     DATOS PLANOS |
 |           ^           |        |           |          |      |         |        |
 +-----------|-----------+        +-----------|----------+      +---------|--------+
             |                                |                           |
             |        [ 8. RESPONSE ] <-------+ <------- [ 7. DTO / MAP ] +
             +------- (UserResponse)
```

Las diferencias clave en el viaje de la Query:
1. La Pregunta (Query): A diferencia del Command (que suele llevar muchos datos para crear/modificar), la Query suele llevar solo filtros o identificadores (ej: userId, courseId).
2. El Contrato de Respuesta (Response/DTO): El Handler no devuelve la entidad del dominio directamente. Construye un objeto Response (un DTO plano). Esto es vital para que, si tu base de datos cambia, tu API no se rompa.
3. El camino de retorno: El QueryBus tiene un return. En el código verás un return await handler.handle(query). Esa respuesta viaja de regreso por todas las capas hasta llegar al Controlador, que la envía al cliente (JSON).
4. Efectos secundarios: Por definición, una Query nunca debe modificar la base de datos. Es una operación segura y repetible (idempotente).

Con estos dos diagramas (Command y Query) ya tienes el mapa mental completo de cómo se mueve la información en tu sistema. ¡Mucho ánimo con ese repaso!

## Flujo de un Domain Event
El flujo de eventos muestra cómo las entidades del dominio pueden publicar eventos que serán procesados por suscriptores de forma asíncrona.

```bash
[ Capa de Dominio ]              [ Capa de Aplicación ]       [ Capa de Infraestructura ]
 +---------------------+        +---------------------+      +------------------------+
 |                     |        |                     |      |                        |
 |  1. ENTIDAD        |        |  3. COMMAND HANDLER |      |  4. EVENT BUS         |
 |  (AggregateRoot)   |        |  (Persiste entidad) |      |  (InMemory/Emitter)   |
 |         |          |        |           |          |      |           |            |
 |         v          |        |           v          |      |           v            |
 |  2. PUBLICA EVENTO |------->|  5. DISPARA EVENTOS |<-----|  6. BUSCA SUSCRIPTORES|
 | (DomainEvent)      |        | (eventBus.publish) |      |  (por nombre clase)   |
 |                     |        |           |          |      |           |            |
 +---------------------+        +-----------|----------+      +-----------|------------+
                                               |
                    +--------------------------+--------------------------+
                    |                          |                          |
                    v                          v                          v
             [ 7. SUBSCRIPTOR 1 ]      [ 7. SUBSCRIPTOR 2 ]      [ 7. SUBSCRIPTOR N ]
             (Ej: Enviar email)         (Ej: Notificar audit)       (Ej: Actualizar cache)
```

Ejemplo de uso:
```typescript
// 1. El CommandHandler recibe un comando
class RegisterUserCommandHandler {
    constructor(
        private readonly userRepository: UserRepository,
        private readonly eventBus: EventBus
    ) {}

    async handle(command: RegisterUserCommand): Promise<void> {
        // 2. Crea la entidad (AggregateRoot)
        const user = User.create(command.email, command.password);

        // 3. Persiste la entidad
        await this.userRepository.save(user);

        // 4. Publica los eventos que generó la entidad
        this.eventBus.publish(user.pullEvents());
    }
}

// 5. El suscriptor escucha y reacciona
class SendWelcomeEmailSubscriber implements DomainEventSubscriber<UserRegisteredDomainEvent> {
    async on(event: UserRegisteredDomainEvent): Promise<void> {
        // Enviar email de bienvenida
        console.log(`Enviando email a ${event.email}`);
    }

    subscribedTo(): new (...args: any[]) => UserRegisteredDomainEvent {
        return UserRegisteredDomainEvent;
    }
}
```

## Nombre de clases: .name vs .constructor.name

En los Buses usamos ambos indistintamente según el contexto:

| Contexto | Input tipo | Código |
|----------|------------|--------|
| **Constructor** (registrar handler) | Clase | `handler.subscribedTo().name` |
| **dispatch/ask/publish** (buscar handler) | Instancia | `command.constructor.name` |

**¿Por qué?**

- `.name` → funciona en **clases** directamente (`MiClase.name` → "MiClase")
- `.constructor.name` → necesario en **instancias** (`new MiClase().constructor.name` → "MiClase")

Ejemplo:
```typescript
// Tienes la CLASE → .name directo
const MyClass = RegisterUserCommand;
MyClass.name  // "RegisterUserCommand" ✅

// Tienes una INSTANCIA → necesitas .constructor
const command = new RegisterUserCommand("email@test.com");
command.name        // undefined ❌
command.constructor.name  // "RegisterUserCommand" ✅
```