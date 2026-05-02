# CQRS: Aprendiendo CQRS en Node.js (CQRS, Hexagonal y DDD)

## Diagrama para Usuarios
```bash
src/
└── Context/
    └── Users/
        ├── Application/
        │   ├── Commands/           <-- Escrituras (C)
        │   │   └── RegisterUser/
        │   │       ├── RegisterUserCommand.ts        (DTO: email, password, etc.)
        │   │       └── RegisterUserCommandHandler.ts (Lógica de orquestación)
        │   ├── Queries/            <-- Lecturas (Q)
        │   │   └── FindUserById/
        │   │       ├── FindUserByIdQuery.ts
        │   │       └── FindUserByIdQueryHandler.ts
        │   └── Response/           <-- DTOs de salida (UserResponse.ts)
        ├── Domain/
        │   ├── User.ts             <-- Entidad (id: string, email: string, etc.)
        │   ├── UserRepository.ts   <-- Interfaz (Puerto)
        │   ├── UserPassword.ts     <-- Value Object (Opcional, para lógica de hash)
        │   └── Events/
        │       └── UserRegisteredDomainEvent.ts
        └── Infrastructure/
            ├── Persistence/
            │   └── MongoUserRepository.ts (O la implementación que prefieras)
            └── Services/
                └── BcryptPasswordHasher.ts (Implementación de seguridad)
```

## Diagrama para Cursos
```bash
src/
└── Context/
    └── Courses/
        ├── Application/
        │   ├── Commands/           <-- Acciones de escritura (Create, Update, Delete)
        │   │   ├── CreateCourse/
        │   │   │   ├── CreateCourseCommand.ts        (DTO de entrada)
        │   │   │   └── CreateCourseCommandHandler.ts (La lógica del "Caso de Uso")
        │   ├── Queries/            <-- Acciones de lectura (Find, Search, List)
        │   │   ├── GetCourseById/
        │   │   │   ├── GetCourseByIdQuery.ts
        │   │   │   └── GetCourseByIdQueryHandler.ts
        │   └── Response/           <-- DTOs de salida para las Queries
        ├── Domain/
        │   ├── Course.ts           <-- Tu Entidad (usando UUID string plano)
        │   ├── CourseRepository.ts <-- Interfaz del puerto
        │   └── Events/             <-- Domain Events (fundamentales en CQRS)
        └── Infrastructure/
            └── Persistence/
                └── TypeOrmCourseRepository.ts
```

## Shared context Diagram
```bash
src/
└── Context/
    └── Shared/
        ├── Application/
        │   ├── Command/
        │   │   └── Command.ts           <-- Interfaz/Clase base que extienden todos los Commands
        │   ├── Query/
        │   │   ├── Query.ts             <-- Interfaz base para todas las Queries
        │   │   └── Response.ts          <-- Interfaz base para los DTOs de respuesta
        │   └── Bus/
        │       ├── CommandBus.ts        <-- Interfaz del bus de comandos (Puerto)
        │       ├── QueryBus.ts          <-- Interfaz del bus de queries (Puerto)
        │       └── EventBus.ts          <-- Interfaz para disparar eventos (Puerto)
        ├── Domain/
        │   ├── AggregateRoot.ts         <-- Clase base para entidades que disparan eventos
        │   ├── ValueObject/
        │   │   └── StringValueObject.ts <-- Utilidad para envolver strings (opcional)
        │   └── Events/
        │       └── DomainEvent.ts       <-- Clase base para todos los eventos de dominio
        └── Infrastructure/
            ├── Bus/
            │   ├── InMemoryCommandBus.ts <-- Implementación real usando una librería o nativo
            │   ├── InMemoryQueryBus.ts
            │   └── EventEmitterEventBus.ts
            └── Persistence/
                └── MongoClientFactory.ts <-- Factoría para conexiones de DB (si usas Mongo/TypeORM)
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