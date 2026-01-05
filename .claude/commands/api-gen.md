---
description: Generate 4-layer API boilerplate (Types → Mappers → API → Hooks) from OpenAPI spec or interactive input
argument-hint: <entity-name or openapi-file-path>
---

Generate complete API integration layers for entity: $ARGUMENTS

## Two Ways to Use

### Option 1: With OpenAPI Spec (Recommended)
Provide an OpenAPI 3.x spec file path:
```
/api-gen ./openapi.yaml
/api-gen ./swagger.json
```

The agent will:
- Parse the OpenAPI spec
- Show available entities
- Let you select one entity
- Auto-detect CRUD operations
- Generate all 4 layers

### Option 2: Interactive Mode
Provide just the entity name:
```
/api-gen ticket
/api-gen user
```

The agent will ask you for the backend DTO schema or example JSON response.

## Generated Structure

Follow the 4-layer architecture from client-api-generate.md:

1. **Types** (`src/types/{entity}.ts`)
   - DTO interface (backend snake_case format)
   - Model interface (client camelCase format)

2. **Mappers** (`src/mappers/{entity}.ts`)
   - Conversion function: snake_case → camelCase, _id → id

3. **API** (`src/api/{entity}.ts`)
   - API functions using customFetch
   - Add paths to `src/constants/api.ts`

4. **TanStack Query Hooks** (`src/hooks/use{Entity}.ts`)
   - useQuery with select option for mapping

## Examples

```bash
# Using OpenAPI spec
/api-gen ./openapi.yaml

# Interactive mode
/api-gen ticket
# Then provide DTO schema when asked
```
