---
title: Production Deployment
parent: Getting Started
nav_order: 3
---

# Production Deployment

---

## Option A: Pre-built Demo binary (no Docker, no license required)

The fastest way to evaluate InventoryFramework. Download the binary and run it, no setup needed.

### 1. Download

Go to the [latest GitHub Release](https://github.com/b-altuncay/InventoryFramework/releases/latest) and download the archive for your platform:

| File | Platform |
|---|---|
| `InventoryFramework-Server-Demo-linux-x64-vX.Y.Z.zip` | Linux x64 |
| `InventoryFramework-Server-Demo-win-x64-vX.Y.Z.zip` | Windows x64 |

**Prerequisite:** [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8) must be installed.

### 2. Extract and run

**Windows:**
```
Unzip, then double-click start-demo.bat
```

**Linux:**
```bash
unzip InventoryFramework-Server-Demo-linux-x64-vX.Y.Z.zip -d inventoryframework
cd inventoryframework
chmod +x start-demo.sh InventoryFramework.Server
./start-demo.sh
```

The server starts on:

| Endpoint | URL |
|---|---|
| gRPC-Web (cleartext) | `http://localhost:5210` |
| gRPC (cleartext) | `http://localhost:7289` |
| Health check | `http://localhost:5210/health` |

### 3. Configure API keys

Edit `appsettings.json` before running:

```json
"Auth": {
  "ApiKeys": [
    { "Key": "sk-admin-CHANGE-THIS", "IsAdmin": true,  "Description": "Admin" },
    { "Key": "sk-game-CHANGE-THIS",  "IsAdmin": false, "Description": "Game server" }
  ]
}
```

### 4. Demo tier limits

| Feature | Demo |
|---|---|
| Inventory CRUD, stack ops, slot locking | Full access |
| Basic crafting | Enabled |
| Craft preview, recipe availability | Disabled |
| Trading, QuickStore, Player Progression | Disabled |
| Concurrent actors | Max 3 |
| Items per actor | Max 50 |

### 5. Upgrade to Pro

1. [Activate your license](https://inventoryframework-license.mbaltuncay99.workers.dev/activate) to download `license.json`
2. Place `license.json` next to the server executable
3. Set `INVENTORY_TIER=Pro` and restart:

**Windows:**
```bat
set INVENTORY_TIER=Pro
InventoryFramework.Server.exe
```

**Linux:**
```bash
INVENTORY_TIER=Pro ./InventoryFramework.Server
```

---

## Option B: Docker (recommended for production)

This guide covers self-hosting the InventoryFramework server in production using Docker.

### Prerequisites

- Docker 24+ and Docker Compose v2
- A domain name or static IP (for TLS)
- A valid license file (`license.json`) for Pro or Enterprise tier

---

## Quick start with Docker

### B.1. Clone and configure

```bash
git clone https://github.com/b-altuncay/InventoryFramework
cd InventoryFramework
cp .env.example .env
```

Edit `.env`:

```env
CERT_PASSWORD=your-strong-password
PERSISTENCE_TYPE=File
ASPNETCORE_ENVIRONMENT=Production
```

### B.2. Generate a TLS certificate

For development / internal use, generate a self-signed certificate:

```bash
bash scripts/generate-dev-cert.sh
```

This creates `certs/server.pfx` and writes `CERT_PASSWORD` to `.env`.

For production, use a real certificate (see [TLS in production](#tls-in-production)).

### B.3. Add your items and recipes

Copy your JSON definition files into the Data directories:

```
InventoryFramework.Server/Data/
  Items/     ← items.json (and any additional files)
  Recipes/   ← recipes.json
  affixes.json
```

### B.4. Configure API keys

Edit `InventoryFramework.Server/appsettings.json` and replace the placeholder keys:

```json
"Auth": {
  "RequireApiKey": true,
  "ApiKeys": [
    { "Key": "sk-admin-YOUR-SECRET", "IsAdmin": true,  "Description": "Admin key" },
    { "Key": "sk-game-YOUR-SECRET",  "IsAdmin": false, "Description": "Game server key" }
  ]
}
```

Never commit real API keys to source control. Use environment variables instead:

```env
# docker-compose environment section (or .env):
InventoryFramework__Auth__ApiKeys__0__Key=sk-admin-YOUR-SECRET
InventoryFramework__Auth__ApiKeys__1__Key=sk-game-YOUR-SECRET
```

### B.5. Apply license (Pro / Enterprise)

Place your `license.json` in the project root. The `docker-compose.yml` mounts it at `/app/license.json`.

```bash
# Verify the license before starting
dotnet InventoryFramework.LicenseGenerator.dll validate --license ./license.json
```

### B.6. Start the server

```bash
docker compose up -d
```

Check it's running:

```bash
curl http://localhost:5210/health
# → Healthy
```

---

## TLS in production

### Option A: Caddy (recommended, automatic Let's Encrypt)

```yaml
# caddy/Caddyfile
inventory.yourdomain.com {
    reverse_proxy h2c://inventory-server:5210
}
```

```yaml
# add to docker-compose.yml services:
  caddy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      - inventory-server
```

Update your SDK clients to use `https://inventory.yourdomain.com`:

```csharp
ServerAddress = "https://inventory.yourdomain.com"
```

### Option B: nginx + certbot

```nginx
server {
    listen 443 ssl http2;
    server_name inventory.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/inventory.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/inventory.yourdomain.com/privkey.pem;

    location / {
        grpc_pass grpcs://127.0.0.1:7289;
    }
}
```

### Option C: Direct Kestrel TLS (VPS without reverse proxy)

Obtain a certificate (e.g. via `certbot certonly --standalone`) and convert to PFX:

```bash
openssl pkcs12 -export \
  -out certs/server.pfx \
  -inkey /etc/letsencrypt/live/yourdomain.com/privkey.pem \
  -in  /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
  -passout pass:your-password
```

Set in `.env`:
```env
CERT_PASSWORD=your-password
```

The server will pick up `certs/server.pfx` automatically.

---

## Rate limiting

By default, requests are limited to **300 per minute per API key**. Adjust in `appsettings.json`:

```json
"InventoryFramework": {
  "RateLimit": {
    "PermitLimit": 300,
    "Window": "00:01:00"
  }
}
```

Or via environment variable:
```env
InventoryFramework__RateLimit__PermitLimit=500
InventoryFramework__RateLimit__Window=00:01:00
```

---

## Persistence backends

### File (default)

Zero configuration. Inventory state is stored as JSON files in `/app/Data/Aggregates/` and `/app/Data/Progression/`. Suitable for single-instance deployments.

### SQLite

```env
PERSISTENCE_TYPE=Sqlite
DB_CONNECTION_STRING=Data Source=/app/Data/inventory.db
```

Mount a volume for the database:
```yaml
volumes:
  - inventory-db:/app/Data
```

### SQL Server

```env
PERSISTENCE_TYPE=SqlServer
DB_CONNECTION_STRING=Server=sql;Database=InventoryDb;User Id=sa;Password=yourpassword;TrustServerCertificate=true
```

### PostgreSQL

```env
PERSISTENCE_TYPE=PostgreSql
DB_CONNECTION_STRING=Host=pg;Database=inventory;Username=app;Password=yourpassword
```

The server runs EF Core migrations automatically on startup.

---

## Environment variable reference

| Variable | Default | Description |
|---|---|---|
| `INVENTORY_TIER` | `Demo` | Active tier when no valid license is found. Values: `Demo`, `Pro`, `Enterprise` |
| `ASPNETCORE_ENVIRONMENT` | `Production` | `Development` disables API key requirement |
| `ASPNETCORE_Kestrel__Certificates__Default__Path` | (none) | Path to PFX cert inside container |
| `ASPNETCORE_Kestrel__Certificates__Default__Password` | (none) | PFX certificate password |
| `InventoryFramework__LicensePath` | (none) | Path to `license.json` |
| `InventoryFramework__Persistence__Type` | `File` | `File`, `Sqlite`, `SqlServer`, `PostgreSql` |
| `InventoryFramework__Persistence__ConnectionString` | (none) | DB connection string |
| `InventoryFramework__RateLimit__PermitLimit` | `300` | Requests allowed per window |
| `InventoryFramework__RateLimit__Window` | `00:01:00` | Rate limit window (TimeSpan) |
| `InventoryFramework__Auth__RequireApiKey` | `true` | Set to `false` for development only |
| `OpenTelemetry__OtlpEndpoint` | (none) | OTLP exporter endpoint (optional) |

---

## Health check

```
GET http://your-server:5210/health
```

Returns `200 Healthy` when the server is ready to accept requests. Use this in your load balancer or container orchestration health probe.
