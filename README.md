# Recipe Box

An end-to-end encrypted (E2EE) multi-room chat PWA themed as a "recipe box."

## Features

- **End-to-end encryption**: Messages are encrypted client-side using AES-GCM with keys derived via PBKDF2
- **No discoverability**: No room directory, no browsing - you need both Recipe Code and Passphrase to join
- **PWA support**: Installable on mobile and desktop
- **Real-time chat**: WebSocket-based messaging via Cloudflare Durable Objects
- **Admin controls**: Only admins can create recipes and rotate passphrases

## Stack

- **Frontend**: Vite + React + TypeScript, deployed on Cloudflare Pages
- **Backend**: Cloudflare Worker + Durable Objects for realtime WebSocket fanout
- **Persistence**: Cloudflare D1 (SQLite) for room metadata + encrypted message history

## Security Model

- Server never sees plaintext messages
- Server never stores passphrases or derived keys
- Each room has public metadata: salt, KDF params, version
- Clients derive room keys from passphrase + room salt using PBKDF2-HMAC-SHA256
- Messages encrypted with AES-GCM (WebCrypto) with fresh 96-bit IV per message
- AAD (Additional Authenticated Data) binds ciphertext to room_id + version + msg_id
- Rate limiting prevents spam (no cryptographic handshake required)

## Project Structure

```
/
├── app/                    # Vite React TypeScript frontend
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.tsx    # "Small kitchen" gate
│   │   │   ├── Room.tsx    # Recipe chat room
│   │   │   └── Admin.tsx   # Admin panel
│   │   ├── utils/
│   │   │   ├── api.ts      # API client
│   │   │   └── crypto.ts   # WebCrypto helpers
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── index.css
│   └── public/
├── api/                    # Cloudflare Worker
│   └── src/
│       ├── index.ts        # Worker + API endpoints
│       └── durable-object.ts # WebSocket Durable Object
├── migrations/
│   └── 0001_init.sql       # D1 schema
├── wrangler.toml           # Wrangler config
└── package.json
```

## Local Development

### Prerequisites

- Node.js 18+
- npm or yarn
- Wrangler CLI (`npm install -g wrangler`)

### Setup

1. Install dependencies:

```bash
npm install
cd app && npm install && cd ..
```

2. Login to Cloudflare (for D1):

```bash
wrangler login
```

3. Create local D1 database and run migrations:

```bash
npm run db:migrate:local
```

4. Start development servers:

```bash
npm run dev
```

This runs:
- Wrangler dev server for the API at `http://localhost:8787`
- Vite dev server for the frontend at `http://localhost:5173`

The Vite dev server proxies `/api` requests to the Wrangler dev server.

### Testing Locally

1. Go to `http://localhost:5173/admin`
2. Enter any string as admin token (e.g., `test-token`)
3. Create a recipe and note the Recipe Code and Passphrase
4. Go to `http://localhost:5173/` and enter the Recipe Code and Passphrase
5. Open another browser/incognito window and join the same recipe
6. Chat in real-time!

## Deployment

### 1. Create D1 Database

```bash
wrangler d1 create recipe-box-db
```

Copy the `database_id` from the output and update `wrangler.toml`:

```toml
[[d1_databases]]
binding = "DB"
database_name = "recipe-box-db"
database_id = "<your-database-id>"
```

### 2. Run Migrations

```bash
npm run db:migrate
```

### 3. Set Admin Token Secret

```bash
wrangler secret put ADMIN_TOKEN --env production
```

Enter a secure random token when prompted.

### 4. Deploy Worker

```bash
npm run deploy:api
```

This deploys to the production environment with `ENVIRONMENT=production`.

### 5. Deploy Frontend to Cloudflare Pages

Option A: Via Wrangler CLI

```bash
npm run deploy:app
```

Option B: Via Cloudflare Dashboard

1. Go to Cloudflare Dashboard > Pages
2. Create a project connected to your Git repository
3. Set build command: `cd app && npm install && npm run build`
4. Set output directory: `app/dist`
5. Add environment variable: `VITE_API_BASE_URL=https://your-worker.workers.dev` (required when `/api/*` is not routed to the Worker).
   - If deploying via GitHub Actions, set this in **Repository Variables** (`vars.VITE_API_BASE_URL`) or **Secrets** (`secrets.VITE_API_BASE_URL`).

### 6. Configure Routing (Optional)

For same-domain setup (recommended):

1. Add a custom domain to your Worker
2. Add the same domain to your Pages project
3. Configure a route pattern so `/api/*` goes to the Worker

Or simply set `VITE_API_BASE_URL` to point to your Worker's URL.

## API Endpoints

### Public

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/rooms/:roomId` | GET | Get room metadata (salt, KDF params, version) |
| `/api/rooms/:roomId/history` | GET | Get encrypted message history |
| `/api/rooms/:roomId/ws` | GET | WebSocket upgrade for real-time chat |

### Admin (requires `Authorization: Bearer <ADMIN_TOKEN>`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/rooms` | POST | Create a new room |
| `/api/admin/rooms/:roomId/rotate` | POST | Rotate passphrase (bump version, new salt) |
| `/api/admin/rooms/:roomId` | PATCH | Update room title |

## Configuration

### Crypto Options (app/src/utils/crypto.ts)

```typescript
// true = display name encrypted inside ciphertext (default)
// false = display name stored in plaintext on server
export const ENCRYPT_DISPLAY_NAME = true;
```

### Rate Limits (api/src/durable-object.ts)

```typescript
const MAX_CONNECTIONS_PER_IP = 10;
const MAX_MESSAGES_PER_MINUTE = 30;
const MAX_MESSAGE_SIZE = 16384; // 16KB
```

## Data Model

### rooms

| Column | Type | Description |
|--------|------|-------------|
| room_id | TEXT | Primary key, URL-safe random ID |
| title | TEXT | Optional room title (plaintext) |
| salt_b64 | TEXT | Base64-encoded salt for KDF |
| kdf_iters | INTEGER | PBKDF2 iteration count |
| version | INTEGER | Current key version |
| created_at | TEXT | ISO timestamp |

### messages

| Column | Type | Description |
|--------|------|-------------|
| room_id | TEXT | Foreign key to rooms |
| msg_id | TEXT | ULID-like message ID |
| version | INTEGER | Key version used for encryption |
| created_at | TEXT | ISO timestamp |
| iv_b64 | TEXT | Base64-encoded AES-GCM IV |
| ciphertext_b64 | TEXT | Base64-encoded ciphertext |
| sender_name | TEXT | Optional plaintext sender (if enabled) |

## License

MIT
