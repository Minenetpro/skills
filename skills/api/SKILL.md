# Minenet Client API v1

REST API for programmatic access to Minecraft game servers.

## Base URL

```
https://minenet.pro/api/client/v1
```

## Authentication

All requests require a Bearer token in the Authorization header:

```
Authorization: Bearer mnp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Tokens are team-scoped and created in the team settings dashboard. Token format: `mnp_` (can be `mna_`) prefix followed by 32 hex characters.

## Rate Limits

- **Global**: 100 requests/minute per token
- **File operations**: Additional per-endpoint limits (noted below)

When rate limited, response includes `Retry-After` header.

## Error Format

```json
{
  "error": "Human-readable error message",
  "code": "ERROR_CODE"
}
```

Common error codes:

- `MISSING_AUTH_HEADER` - No Authorization header
- `INVALID_TOKEN` - Token invalid or revoked
- `RATE_LIMITED` - Too many requests
- `SERVER_NOT_FOUND` - Server doesn't exist or not authorized
- `SERVER_INSTALLING` - Server is being installed (409)
- `SERVER_OFFLINE` - Server not running
- `VALIDATION_ERROR` - Invalid request parameters

---

## Endpoints

### Servers

#### List Servers

```http
GET /servers
```

Returns all servers belonging to your team with status information.

**Response:**

```json
{
  "servers": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "My Minecraft Server",
      "status": "running",
      "address": "mc.example.com:25565",
      "resources": {
        "cpu": 45.2,
        "memory_bytes": 2147483648,
        "disk_bytes": 5368709120,
        "uptime_ms": 3600000
      }
    }
  ],
  "count": 1
}
```

**Status values:** `running`, `starting`, `stopping`, `offline`, `unknown`

---

#### Send Power Action

```http
POST /servers/{id}/power
Content-Type: application/json

{
  "action": "start"
}
```

**Actions:** `start`, `stop`, `restart`, `kill`

**Response:**

```json
{
  "ok": true,
  "action": "start",
  "message": "Power action 'start' sent successfully"
}
```

---

#### Send Console Command

```http
POST /servers/{id}/command
Content-Type: application/json

{
  "command": "say Hello World!"
}
```

Command max length: 10,000 characters. Server must be running.

**Response:**

```json
{
  "ok": true,
  "message": "Command sent successfully"
}
```

---

#### Get Console History

```http
GET /servers/{id}/console?size=100
```

**Query Parameters:**

- `size` (optional): Number of lines, 1-500, default 100

**Response:**

```json
{
  "lines": [
    "[12:00:01] [Server thread/INFO]: Player joined the game",
    "[12:00:05] [Server thread/INFO]: Hello World!"
  ],
  "count": 2
}
```

---

### File Operations

All file paths are relative to the server root directory.

#### List Directory

```http
GET /servers/{id}/files/list?directory=/plugins
```

**Query Parameters:**

- `directory` (optional): Path to list, default `/`

**Response:**

```json
{
  "files": [
    {
      "name": "server.properties",
      "size": 1024,
      "is_file": true,
      "is_directory": false,
      "is_symlink": false,
      "mime": "text/plain",
      "mode": "0644",
      "mode_bits": "644",
      "modified_at": "2024-01-15T10:30:00Z",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "count": 1
}
```

---

#### Read File

```http
GET /servers/{id}/files/contents?file=/server.properties
```

**Query Parameters:**

- `file` (required): Path to file

**Response:** Raw file contents as `text/plain`. Max file size: 10MB.

---

#### Write File

```http
POST /servers/{id}/files/write?file=/server.properties
Content-Type: text/plain

motd=Welcome to my server!
max-players=20
```

**Query Parameters:**

- `file` (required): Path to file

**Request Body:** Raw file contents

**Rate Limit:** 60/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Delete Files

```http
POST /servers/{id}/files/delete
Content-Type: application/json

{
  "root": "/plugins",
  "files": ["OldPlugin.jar", "AnotherPlugin.jar"]
}
```

**Request Body:**

- `root` (optional): Base directory, default `/`
- `files` (required): Array of file/folder names (max 100)

**Rate Limit:** 30/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Rename/Move Files

```http
POST /servers/{id}/files/rename
Content-Type: application/json

{
  "root": "/",
  "files": [
    { "from": "old-name.txt", "to": "new-name.txt" },
    { "from": "file.txt", "to": "subfolder/file.txt" }
  ]
}
```

**Request Body:**

- `root` (optional): Base directory, default `/`
- `files` (required): Array of rename operations (max 100)

**Rate Limit:** 30/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Copy File

```http
POST /servers/{id}/files/copy
Content-Type: application/json

{
  "location": "/config/settings.yml"
}
```

Creates a copy with `copy` suffix (e.g., `settings copy.yml`).

**Rate Limit:** 30/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Create Directory

```http
POST /servers/{id}/files/create-directory
Content-Type: application/json

{
  "root": "/",
  "name": "backups"
}
```

**Request Body:**

- `root` (optional): Parent directory, default `/`
- `name` (required): New directory name

**Rate Limit:** 30/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Compress Files

```http
POST /servers/{id}/files/compress
Content-Type: application/json

{
  "root": "/",
  "files": ["plugins", "config"]
}
```

Creates a `.tar.gz` archive.

**Request Body:**

- `root` (optional): Base directory, default `/`
- `files` (required): Array of files/folders to compress (max 100)

**Rate Limit:** 20/minute

**Response:**

```json
{
  "ok": true,
  "archive": {
    "name": "archive-2024-01-15T103000Z.tar.gz",
    "size": 1048576,
    "is_file": true,
    "is_directory": false,
    "is_symlink": false,
    "mime": "application/gzip",
    "mode": "0644",
    "mode_bits": "644",
    "modified_at": "2024-01-15T10:30:00Z",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

---

#### Decompress Archive

```http
POST /servers/{id}/files/decompress
Content-Type: application/json

{
  "root": "/",
  "file": "backup.tar.gz"
}
```

Extracts archive contents to the root directory.

**Supported formats:** `.zip`, `.tar`, `.tar.gz`, `.tgz`, `.tar.bz2`

**Rate Limit:** 20/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### Search Files

```http
GET /servers/{id}/files/search?pattern=*.yml&directory=/
```

**Query Parameters:**

- `pattern` (required): Glob pattern (e.g., `*.yml`, `config*`)
- `directory` (optional): Directory to search, default `/`

**Rate Limit:** 30/minute

**Response:**

```json
{
  "results": [
    {
      "name": "config.yml",
      "directory": "/plugins/Essentials",
      "file": {
        "name": "config.yml",
        "size": 2048,
        "is_file": true,
        "is_directory": false,
        "is_symlink": false,
        "mime": "text/yaml",
        "mode": "0644",
        "mode_bits": "644",
        "modified_at": "2024-01-15T10:30:00Z",
        "created_at": "2024-01-01T00:00:00Z"
      }
    }
  ],
  "count": 1
}
```

---

#### Download File from URL

```http
POST /servers/{id}/files/pull
Content-Type: application/json

{
  "url": "https://example.com/plugin.jar",
  "directory": "/plugins",
  "filename": "MyPlugin.jar",
  "use_header": false,
  "foreground": false
}
```

Downloads a file from a URL to the server.

**Request Body:**

- `url` (required): URL to download from
- `directory` (optional): Target directory, default `/`
- `filename` (optional): Override filename
- `use_header` (optional): Use Content-Disposition header for filename
- `foreground` (optional): Wait for download to complete

**Rate Limit:** 10/minute

**Response:**

```json
{
  "ok": true
}
```

---

#### List Active Downloads

```http
GET /servers/{id}/files/pull
```

**Rate Limit:** 60/minute

**Response:**

```json
{
  "downloads": [
    {
      "identifier": "abc123",
      "url": "https://example.com/large-file.zip",
      "progress": 45.5,
      "size": 104857600,
      "error": null
    }
  ],
  "count": 1
}
```

---

#### Cancel Download

```http
DELETE /servers/{id}/files/pull/{downloadId}
```

Cancels an active download by its identifier.

**Response:**

```json
{
  "ok": true
}
```

---

## Code Examples

### JavaScript/TypeScript

```javascript
const API_BASE = "https://minenet.pro/api/client/v1";
const TOKEN = "mnp_your_token_here";

async function api(endpoint, options = {}) {
  const res = await fetch(`${API_BASE}${endpoint}`, {
    ...options,
    headers: {
      Authorization: `Bearer ${TOKEN}`,
      "Content-Type": "application/json",
      ...options.headers,
    },
  });
  if (!res.ok) {
    const error = await res.json();
    throw new Error(error.error || `HTTP ${res.status}`);
  }
  return res.json();
}

// List servers
const { servers } = await api("/servers");

// Start a server
await api(`/servers/${servers[0].id}/power`, {
  method: "POST",
  body: JSON.stringify({ action: "start" }),
});

// Send command
await api(`/servers/${serverId}/command`, {
  method: "POST",
  body: JSON.stringify({ command: "say Hello!" }),
});

// Read file
const response = await fetch(
  `${API_BASE}/servers/${serverId}/files/contents?file=/server.properties`,
  {
    headers: { Authorization: `Bearer ${TOKEN}` },
  }
);
const content = await response.text();

// Write file
await fetch(
  `${API_BASE}/servers/${serverId}/files/write?file=/server.properties`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${TOKEN}`,
      "Content-Type": "text/plain",
    },
    body: "motd=My Server\nmax-players=50",
  }
);
```

### Python

```python
import requests

API_BASE = "https://minenet.pro/api/client/v1"
TOKEN = "mnp_your_token_here"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

# List servers
response = requests.get(f"{API_BASE}/servers", headers=HEADERS)
servers = response.json()["servers"]

# Start server
requests.post(
    f"{API_BASE}/servers/{servers[0]['id']}/power",
    headers=HEADERS,
    json={"action": "start"}
)

# Send command
requests.post(
    f"{API_BASE}/servers/{server_id}/command",
    headers=HEADERS,
    json={"command": "say Hello!"}
)

# Read file
response = requests.get(
    f"{API_BASE}/servers/{server_id}/files/contents",
    headers=HEADERS,
    params={"file": "/server.properties"}
)
content = response.text

# Write file
requests.post(
    f"{API_BASE}/servers/{server_id}/files/write",
    headers={**HEADERS, "Content-Type": "text/plain"},
    params={"file": "/server.properties"},
    data="motd=My Server\nmax-players=50"
)
```

### cURL

```bash
# List servers
curl -H "Authorization: Bearer mnp_xxx" \
  "https://minenet.pro/api/client/v1/servers"

# Start server
curl -X POST -H "Authorization: Bearer mnp_xxx" \
  -H "Content-Type: application/json" \
  -d '{"action":"start"}' \
  "https://minenet.pro/api/client/v1/servers/{id}/power"

# Send command
curl -X POST -H "Authorization: Bearer mnp_xxx" \
  -H "Content-Type: application/json" \
  -d '{"command":"say Hello!"}' \
  "https://minenet.pro/api/client/v1/servers/{id}/command"

# Read file
curl -H "Authorization: Bearer mnp_xxx" \
  "https://minenet.pro/api/client/v1/servers/{id}/files/contents?file=/server.properties"

# Write file
curl -X POST -H "Authorization: Bearer mnp_xxx" \
  -H "Content-Type: text/plain" \
  -d "motd=My Server" \
  "https://minenet.pro/api/client/v1/servers/{id}/files/write?file=/server.properties"
```

---

## Common Workflows

### Check Server Status and Start if Offline

```javascript
const { servers } = await api("/servers");
const server = servers.find((s) => s.name === "My Server");

if (server.status === "offline") {
  await api(`/servers/${server.id}/power`, {
    method: "POST",
    body: JSON.stringify({ action: "start" }),
  });
  console.log("Server starting...");
}
```

### Backup Configuration Files

```javascript
// Compress config files
const { archive } = await api(`/servers/${serverId}/files/compress`, {
  method: "POST",
  body: JSON.stringify({
    root: "/",
    files: ["server.properties", "bukkit.yml", "spigot.yml", "plugins"],
  }),
});

console.log(`Backup created: ${archive.name}`);
```

### Install Plugin from URL

```javascript
await api(`/servers/${serverId}/files/pull`, {
  method: "POST",
  body: JSON.stringify({
    url: "https://cdn.modrinth.com/data/xxx/versions/yyy/plugin.jar",
    directory: "/plugins",
    foreground: true,
  }),
});

// Restart to load new plugin
await api(`/servers/${serverId}/power`, {
  method: "POST",
  body: JSON.stringify({ action: "restart" }),
});
```

### Update Server Configuration

```javascript
// Read current config
const res = await fetch(
  `${API_BASE}/servers/${serverId}/files/contents?file=/server.properties`,
  { headers: { Authorization: `Bearer ${TOKEN}` } }
);
let config = await res.text();

// Modify config
config = config.replace(/max-players=\d+/, "max-players=100");
config = config.replace(/motd=.*/, "motd=Welcome to our server!");

// Write updated config
await fetch(
  `${API_BASE}/servers/${serverId}/files/write?file=/server.properties`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${TOKEN}`,
      "Content-Type": "text/plain",
    },
    body: config,
  }
);

// Restart to apply changes
await api(`/servers/${serverId}/power`, {
  method: "POST",
  body: JSON.stringify({ action: "restart" }),
});
```

---

## Path Security

- Path traversal (`..`) is blocked
- Null bytes in paths are rejected
- Maximum path length: 4096 characters
- Maximum batch operations: 100 files per request
