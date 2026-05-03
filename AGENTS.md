# AGENTS.md

## Project Overview
Node.js CQRS-DDD template with TypeScript, Express, and in-memory buses.

## Commands
- `npm run dev` - Start backend in watch mode
- `npm run cli` - Run CLI app
- `npm run build` - Compile TypeScript with tsc-alias

## Key Paths
- `src/Apps/Backend/` - Express server
- `src/Apps/CLI/` - CLI application
- `src/Context/Users/` - User bounded context
- `src/Context/Shared/` - Shared domain infrastructure

## Architecture
- CQRS pattern with CommandBus/QueryBus
- DDD with Domain Events
- In-memory event bus for domain events