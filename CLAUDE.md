# CLAUDE.md

## Project Overview

Recipe Box — an end-to-end encrypted multi-room chat PWA. Full-stack TypeScript monorepo with a React frontend and Cloudflare Workers backend.

## Architecture

- **Frontend** (`app/`): React 18 + Vite + TypeScript, deployed to Cloudflare Pages
- **Backend** (`api/`): Cloudflare Workers + Durable Objects (WebSocket fanout) + D1 (SQLite)
- **Encryption**: AES-GCM via WebCrypto API, PBKDF2 key derivation — server never sees plaintext
- **Database schema**: `migrations/0001_init.sql`

## Development

```bash
# Install dependencies (both root and app)
npm install && cd app && npm install && cd ..

# Set up local database
npm run db:migrate:local

# Start dev servers (API on :8787, app on :5173 with /api proxy)
npm run dev
```

## Build & Deploy

```bash
npm run build             # Build frontend (runs tsc + vite build)
npm run build:api         # Dry-run Worker deploy
npm run deploy:api        # Deploy Worker to production
npm run deploy:app        # Deploy frontend to Cloudflare Pages
```

## Code Style

- TypeScript strict mode enabled — no unused locals or parameters
- camelCase for variables/functions, PascalCase for components/classes
- No ESLint, Prettier, or pre-commit hooks configured
- Type checking is the primary code quality gate (`tsc` runs before build)

## Testing

No test framework is configured. The build (`npm run build`) runs `tsc` for type checking.

## Key Files

- `api/src/index.ts` — Worker entry point, API routes
- `api/src/durable-object.ts` — WebSocket Durable Object for real-time messaging
- `app/src/utils/crypto.ts` — Client-side encryption helpers
- `app/src/utils/api.ts` — API client
- `app/src/pages/Room.tsx` — Chat room UI
- `wrangler.toml` — Cloudflare Worker/D1/Durable Object bindings

## CI/CD

GitHub Actions (`.github/workflows/deploy.yml`) deploys on push to `main`. Two jobs: deploy Worker, then build and deploy Pages. Requires `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` secrets.
