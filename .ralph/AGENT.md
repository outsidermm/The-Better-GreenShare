# Agent Instructions

## Build
```bash
pnpm install
pnpm build
```

## Test
```bash
# Type checking (primary test for this project)
pnpm typecheck

# Linting
pnpm lint

# Full check (lint + typecheck)
pnpm check
```

## Run
```bash
# Development server (http://localhost:3000)
pnpm dev

# Production build and start
pnpm build && pnpm start
```

## Database

```bash
# Start local PostgreSQL (Docker/Podman container)
./start-database.sh

# Push schema changes (development only, no migration files)
pnpm db:push

# Generate Prisma Client + create migration (production/team workflow)
pnpm db:generate

# Open Prisma Studio (database GUI)
pnpm db:studio
```

## Lint & Format

```bash
# Run ESLint
pnpm lint

# Fix ESLint issues automatically
pnpm lint:fix

# Check code formatting with Prettier
pnpm format:check

# Format code with Prettier
pnpm format:write
```

## Important Notes

- **Always run `pnpm typecheck` after backend changes** to catch type errors
- After Prisma schema changes, run `pnpm db:push` (dev) or `pnpm db:generate` (migration)
- Prisma Client generates to `generated/prisma/client` (custom output path)
- Use `pnpm` (not npm or yarn) - lockfile is `pnpm-lock.yaml`
