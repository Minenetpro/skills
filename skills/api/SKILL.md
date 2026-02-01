# Client API v1

The Client API provides programmatic access to team resources via bearer token authentication. It is designed for external integrations, CI/CD pipelines, monitoring bots, and other automated systems.

## Base URL

```
/api/client/v1
```

## Authentication

All requests must include an API token in the `Authorization` header:

```
Authorization: Bearer mnp_<token>
```

Tokens are created in Team Settings and are scoped to a single team. They provide full access to all servers belonging to that team.

### Token Format

- **Prefix**: `mnp_` (Minenet Pro)
- **Length**: 36 characters total (`mnp_` + 32 hex chars)
- **Storage**: SHA-256 hashed (raw token never stored)
- **Expiration**: Never expires (valid until revoked)

### Error Responses

| Status | Code                   | Description                         |
| ------ | ---------------------- | ----------------------------------- |
| 401    | `MISSING_AUTH_HEADER`  | No Authorization header             |
| 401    | `INVALID_AUTH_FORMAT`  | Header doesn't start with "Bearer " |
| 401    | `EMPTY_TOKEN`          | Token is empty                      |
| 401    | `INVALID_TOKEN_FORMAT` | Token doesn't start with "mnp\_"    |
| 401    | `INVALID_TOKEN`        | Token not found or revoked          |
| 429    | `RATE_LIMITED`         | Rate limit exceeded                 |

## Rate Limiting

- **Limit**: 100 requests per minute per token
- **Window**: Rolling 60-second window
- **Scope**: Per-token (not per-team)

When rate limited, response includes:

```
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>
```

---

## Endpoints

### List Servers

```
GET /api/client/v1/servers
```

Returns all servers belonging to the authenticated team.

**Response**

```json
{
  "servers": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "My Minecraft Server",
      "status": "running",
      "address": "play.example.com:25565",
      "resources": {
        "cpu": 45.2,
        "memory_bytes": 2147483648,
        "disk_bytes": 10737418240,
        "uptime_ms": 86400000
      }
    }
  ],
  "count": 1
}
```

**Status Values**: `running`, `starting`, `stopping`, `offline`, `unknown`

---

### Get Console History

```
GET /api/client/v1/servers/{id}/console
```

Returns recent console output from the server.

**Query Parameters**

| Parameter | Type    | Default | Range | Description               |
| --------- | ------- | ------- | ----- | ------------------------- |
| `size`    | integer | 100     | 1-500 | Number of lines to return |

**Response**

```json
{
  "lines": [
    "[12:00:00 INFO]: Server started",
    "[12:00:01 INFO]: Player joined the game"
  ],
  "count": 2
}
```

**Error Codes**

| Status | Code                | Description                            |
| ------ | ------------------- | -------------------------------------- |
| 400    | `INVALID_SERVER_ID` | Server ID is not a valid UUID          |
| 404    | `SERVER_NOT_FOUND`  | Server doesn't exist or not authorized |
| 409    | `SERVER_INSTALLING` | Server is still being installed        |
| 502    | `UPSTREAM_ERROR`    | Failed to fetch from backend           |

---

### Send Power Action

```
POST /api/client/v1/servers/{id}/power
```

Sends a power action to the server.

**Request Body**

```json
{
  "action": "restart"
}
```

**Valid Actions**: `start`, `stop`, `restart`, `kill`

**Response**

```json
{
  "ok": true,
  "action": "restart",
  "message": "Power action 'restart' sent successfully"
}
```

**Error Codes**

| Status | Code                | Description                            |
| ------ | ------------------- | -------------------------------------- |
| 400    | `INVALID_SERVER_ID` | Server ID is not a valid UUID          |
| 400    | `VALIDATION_ERROR`  | Invalid action value                   |
| 404    | `SERVER_NOT_FOUND`  | Server doesn't exist or not authorized |
| 409    | `SERVER_INSTALLING` | Server is still being installed        |
| 502    | `UPSTREAM_ERROR`    | Failed to send to backend              |

---

### Send Command

```
POST /api/client/v1/servers/{id}/command
```

Sends a console command to the server.

**Request Body**

```json
{
  "command": "say Hello from API!"
}
```

**Constraints**

- `command`: Required, 1-10000 characters

**Response**

```json
{
  "ok": true,
  "message": "Command sent successfully"
}
```

**Error Codes**

| Status | Code                | Description                            |
| ------ | ------------------- | -------------------------------------- |
| 400    | `INVALID_SERVER_ID` | Server ID is not a valid UUID          |
| 400    | `VALIDATION_ERROR`  | Command is empty or too long           |
| 404    | `SERVER_NOT_FOUND`  | Server doesn't exist or not authorized |
| 409    | `SERVER_INSTALLING` | Server is still being installed        |
| 502    | `SERVER_OFFLINE`    | Server is not running                  |
| 502    | `UPSTREAM_ERROR`    | Failed to send to backend              |

---

## File Management Endpoints

All file endpoints use direct Wings node communication for lower latency.

### Rate Limits by Endpoint

| Endpoint                 | Limit  | Notes               |
| ------------------------ | ------ | ------------------- |
| `files/list`             | None   | Read-only           |
| `files/contents`         | None   | Read-only, 10MB max |
| `files/search`           | 30/min | Can be expensive    |
| `files/write`            | 60/min | Write operation     |
| `files/delete`           | 30/min | Destructive         |
| `files/rename`           | 30/min | Write operation     |
| `files/copy`             | 30/min | Write operation     |
| `files/create-directory` | 30/min | Write operation     |
| `files/compress`         | 20/min | Heavy operation     |
| `files/decompress`       | 20/min | Heavy operation     |

---

### List Directory

```
GET /api/client/v1/servers/{id}/files/list
```

Lists contents of a directory on the server.

**Query Parameters**

| Parameter   | Type   | Default | Description            |
| ----------- | ------ | ------- | ---------------------- |
| `directory` | string | `/`     | Directory path to list |

**Response**

```json
{
  "files": [
    {
      "name": "server.properties",
      "size": 1234,
      "is_file": true,
      "is_directory": false,
      "is_symlink": false,
      "mime": "text/plain",
      "mode": "-rw-r--r--",
      "mode_bits": "0644",
      "modified_at": "2024-01-15T10:30:00Z",
      "created_at": "2024-01-10T08:00:00Z"
    }
  ],
  "count": 1
}
```

---

### Read File Contents

```
GET /api/client/v1/servers/{id}/files/contents
```

Reads the contents of a file.

**Query Parameters**

| Parameter | Type   | Required | Description       |
| --------- | ------ | -------- | ----------------- |
| `file`    | string | Yes      | File path to read |

**Constraints**

- Maximum file size: 10MB
- Returns `413 FILE_TOO_LARGE` if file exceeds limit

**Response**: Raw file contents with `Content-Type: text/plain`

---

### Search Files

```
GET /api/client/v1/servers/{id}/files/search
```

Searches for files matching a glob pattern.

**Query Parameters**

| Parameter   | Type   | Required | Default | Description                                |
| ----------- | ------ | -------- | ------- | ------------------------------------------ |
| `pattern`   | string | Yes      | -       | Search pattern (glob-style, e.g., `*.yml`) |
| `directory` | string | No       | `/`     | Directory to search in                     |

**Rate Limit**: 30 requests/minute

**Response**

```json
{
  "results": [
    {
      "name": "config.yml",
      "directory": "/plugins/MyPlugin",
      "file": {
        "name": "config.yml",
        "size": 1234,
        "is_file": true,
        "is_directory": false,
        "mime": "text/yaml",
        "mode": "-rw-r--r--",
        "mode_bits": "0644",
        "modified_at": "2024-01-15T10:30:00Z"
      }
    }
  ],
  "count": 1
}
```

---

### Write File

```
POST /api/client/v1/servers/{id}/files/write
```

Writes content to a file.

**Query Parameters**

| Parameter | Type   | Required | Description        |
| --------- | ------ | -------- | ------------------ |
| `file`    | string | Yes      | File path to write |

**Request Body**: Raw file contents (`Content-Type: text/plain`)

**Rate Limit**: 60 requests/minute

**Audit Event**: `server.file_edited` (with diff, redacted for sensitive files)

**Response**

```json
{ "ok": true }
```

---

### Delete Files

```
POST /api/client/v1/servers/{id}/files/delete
```

Deletes files or directories.

**Request Body**

```json
{
  "root": "/",
  "files": ["file1.txt", "folder/"]
}
```

**Constraints**: Maximum 100 files per request

**Rate Limit**: 30 requests/minute

**Audit Event**: `server.file_deleted`

**Response**

```json
{ "ok": true }
```

---

### Rename/Move Files

```
POST /api/client/v1/servers/{id}/files/rename
```

Renames or moves files.

**Request Body**

```json
{
  "root": "/",
  "files": [{ "from": "old-name.txt", "to": "new-name.txt" }]
}
```

**Constraints**: Maximum 100 files per request

**Rate Limit**: 30 requests/minute

**Audit Event**: `server.file_renamed`

**Response**

```json
{ "ok": true }
```

---

### Copy File

```
POST /api/client/v1/servers/{id}/files/copy
```

Copies a file.

**Request Body**

```json
{
  "location": "/path/to/file.txt"
}
```

**Rate Limit**: 30 requests/minute

**Audit Event**: `server.file_copied`

**Response**

```json
{ "ok": true }
```

---

### Create Directory

```
POST /api/client/v1/servers/{id}/files/create-directory
```

Creates a new directory.

**Request Body**

```json
{
  "root": "/",
  "name": "new-folder"
}
```

**Rate Limit**: 30 requests/minute

**Audit Event**: `server.folder_created`

**Response**

```json
{ "ok": true }
```

---

### Compress Files

```
POST /api/client/v1/servers/{id}/files/compress
```

Compresses files into an archive.

**Request Body**

```json
{
  "root": "/",
  "files": ["folder1", "file1.txt"]
}
```

**Constraints**: Maximum 100 files per request

**Rate Limit**: 20 requests/minute

**Audit Event**: `server.file_compressed`

**Response**

```json
{
  "ok": true,
  "archive": {
    "name": "archive-2024-01-15T103000Z.tar.gz",
    "size": 1048576,
    "is_file": true,
    "is_directory": false,
    "mime": "application/gzip",
    "mode": "-rw-r--r--",
    "mode_bits": "0644",
    "modified_at": "2024-01-15T10:30:00Z"
  }
}
```

---

### Decompress Archive

```
POST /api/client/v1/servers/{id}/files/decompress
```

Decompresses an archive.

**Request Body**

```json
{
  "root": "/",
  "file": "archive.tar.gz"
}
```

**Rate Limit**: 20 requests/minute

**Audit Event**: `server.file_decompressed`

**Response**

```json
{ "ok": true }
```

---

## Remote Downloads

Remote download endpoints allow you to download files from external URLs directly to the server.

### Rate Limits

| Endpoint                   | Limit  |
| -------------------------- | ------ |
| `files/pull` (GET)         | 60/min |
| `files/pull` (POST)        | 10/min |
| `files/pull/{id}` (DELETE) | 30/min |

---

### List Active Downloads

```
GET /api/client/v1/servers/{id}/files/pull
```

Returns all active remote downloads for the server.

**Rate Limit**: 60 requests/minute

**Response**

```json
{
  "downloads": [
    {
      "identifier": "abc123",
      "url": "https://example.com/file.jar",
      "progress": 45.5,
      "size": 10485760,
      "error": null
    }
  ],
  "count": 1
}
```

**Download Object**

| Field        | Type           | Description                      |
| ------------ | -------------- | -------------------------------- |
| `identifier` | string         | Unique download identifier       |
| `url`        | string         | Source URL being downloaded      |
| `progress`   | number         | Download progress (0-100)        |
| `size`       | number         | Total file size in bytes         |
| `error`      | string \| null | Error message if download failed |

---

### Start Remote Download

```
POST /api/client/v1/servers/{id}/files/pull
```

Starts downloading a file from an external URL.

**Rate Limit**: 10 requests/minute

**Request Body**

```json
{
  "url": "https://example.com/file.jar",
  "directory": "/plugins",
  "filename": "custom-name.jar",
  "use_header": false,
  "foreground": false
}
```

**Request Fields**

| Field        | Type    | Required | Default | Description                                     |
| ------------ | ------- | -------- | ------- | ----------------------------------------------- |
| `url`        | string  | Yes      | -       | URL to download from (must be HTTPS)            |
| `directory`  | string  | Yes      | -       | Target directory on server                      |
| `filename`   | string  | No       | -       | Custom filename (uses URL filename if omitted)  |
| `use_header` | boolean | No       | false   | Use Content-Disposition header for filename     |
| `foreground` | boolean | No       | false   | Wait for download to complete before responding |

**Audit Event**: `server.file_downloaded`

**Response**

```json
{ "ok": true }
```

---

### Cancel Remote Download

```
DELETE /api/client/v1/servers/{id}/files/pull/{downloadId}
```

Cancels an active remote download.

**Rate Limit**: 30 requests/minute

**Path Parameters**

| Parameter    | Type   | Description                            |
| ------------ | ------ | -------------------------------------- |
| `downloadId` | string | Download identifier from list endpoint |

**Audit Event**: `server.file_download_cancelled`

**Response**

```json
{ "ok": true }
```

**Error Codes**

| Status | Code                  | Description            |
| ------ | --------------------- | ---------------------- |
| 400    | `INVALID_DOWNLOAD_ID` | Download ID is empty   |
| 404    | `DOWNLOAD_NOT_FOUND`  | Download doesn't exist |

---

### File Endpoint Error Codes

| Status | Code                | Description                            |
| ------ | ------------------- | -------------------------------------- |
| 400    | `INVALID_SERVER_ID` | Server ID is not a valid UUID          |
| 400    | `NO_NODE_ASSIGNED`  | Server has no assigned node            |
| 400    | `VALIDATION_ERROR`  | Input validation failed                |
| 404    | `SERVER_NOT_FOUND`  | Server doesn't exist or not authorized |
| 404    | `FILE_NOT_FOUND`    | File doesn't exist                     |
| 413    | `FILE_TOO_LARGE`    | File exceeds 10MB limit                |
| 429    | `RATE_LIMITED`      | Per-endpoint rate limit exceeded       |
| 502    | `NODE_NOT_FOUND`    | Node not registered                    |
| 502    | `WINGS_ERROR`       | Wings node returned an error           |

---

## Modrinth API

Endpoints for searching and retrieving information from Modrinth (mods, plugins, modpacks).

### Search Modrinth

```
GET /api/client/v1/modrinth/search
```

Search for mods and plugins on Modrinth.

**Query Parameters**

| Parameter      | Type    | Default   | Description                                                    |
| -------------- | ------- | --------- | -------------------------------------------------------------- |
| `query`        | string  | -         | Search text                                                    |
| `limit`        | integer | 10        | Results per page (1-100)                                       |
| `offset`       | integer | 0         | Pagination offset                                              |
| `index`        | string  | relevance | Sort: `relevance`, `downloads`, `follows`, `newest`, `updated` |
| `project_type` | string  | -         | Filter: `mod` or `plugin`                                      |
| `versions`     | string  | -         | Comma-separated game versions (e.g., `1.21.4,1.20.1`)          |
| `loaders`      | string  | -         | Comma-separated loaders (e.g., `paper,fabric`)                 |
| `categories`   | string  | -         | Comma-separated categories                                     |
| `server_side`  | string  | -         | Filter: `required` or `optional`                               |

**Response**

```json
{
  "count": 5,
  "total_hits": 150,
  "offset": 0,
  "limit": 10,
  "query": "luckperms",
  "results": [
    {
      "project_id": "Vebnzrzj",
      "slug": "luckperms",
      "title": "LuckPerms",
      "description": "A permissions plugin",
      "downloads": 5000000,
      "icon_url": "https://...",
      "project_type": "mod",
      "server_side": "required",
      "client_side": "optional",
      "latest_version": "5.4.102",
      "categories": ["utility"],
      "versions": ["1.21.4", "1.20.1"]
    }
  ]
}
```

---

### List Project Versions

```
GET /api/client/v1/modrinth/projects/{id}/versions
```

List available versions for a Modrinth project.

**Path Parameters**

| Parameter | Type   | Description        |
| --------- | ------ | ------------------ |
| `id`      | string | Project ID or slug |

**Query Parameters**

| Parameter       | Type   | Description                                    |
| --------------- | ------ | ---------------------------------------------- |
| `loaders`       | string | Comma-separated loaders (e.g., `paper,fabric`) |
| `game_versions` | string | Comma-separated game versions (e.g., `1.21.4`) |
| `featured`      | string | Set to `true` to only return featured versions |

**Response**

```json
{
  "project_id": "luckperms",
  "count": 25,
  "versions": [
    {
      "id": "abc123",
      "version_number": "5.4.102",
      "name": "LuckPerms v5.4.102",
      "date_published": "2024-01-15T10:00:00Z",
      "loaders": ["paper", "spigot"],
      "game_versions": ["1.21.4", "1.20.1"],
      "featured": true,
      "version_type": "release",
      "downloads": 50000,
      "files": [
        {
          "filename": "LuckPerms-5.4.102.jar",
          "url": "https://...",
          "primary": true,
          "size": 1234567
        }
      ],
      "dependencies": []
    }
  ]
}
```

---

### Get Version Details

```
GET /api/client/v1/modrinth/versions/{versionId}
```

Get details for a specific version.

**Path Parameters**

| Parameter   | Type   | Description |
| ----------- | ------ | ----------- |
| `versionId` | string | Version ID  |

**Response**

```json
{
  "id": "abc123",
  "project_id": "Vebnzrzj",
  "name": "LuckPerms v5.4.102",
  "version_number": "5.4.102",
  "changelog": "- Fixed bug...",
  "date_published": "2024-01-15T10:00:00Z",
  "downloads": 50000,
  "version_type": "release",
  "loaders": ["paper", "spigot"],
  "game_versions": ["1.21.4", "1.20.1"],
  "featured": true,
  "files": [
    {
      "filename": "LuckPerms-5.4.102.jar",
      "url": "https://...",
      "primary": true,
      "size": 1234567,
      "hashes": {
        "sha1": "abc...",
        "sha512": "def..."
      }
    }
  ],
  "dependencies": []
}
```

**Error Codes**

| Status | Code                 | Description             |
| ------ | -------------------- | ----------------------- |
| 400    | `MISSING_PROJECT_ID` | Project ID not provided |
| 400    | `MISSING_VERSION_ID` | Version ID not provided |
| 404    | `PROJECT_NOT_FOUND`  | Project doesn't exist   |
| 404    | `VERSION_NOT_FOUND`  | Version doesn't exist   |

---

## CurseForge API

Endpoints for searching and retrieving information from CurseForge (modpacks, mods).

### Search CurseForge

```
GET /api/client/v1/curseforge/search
```

Search for modpacks (or mods) on CurseForge.

**Query Parameters**

| Parameter      | Type    | Default | Description                               |
| -------------- | ------- | ------- | ----------------------------------------- |
| `searchFilter` | string  | -       | Search text (alias: `query`)              |
| `pageSize`     | integer | 10      | Results per page (1-50)                   |
| `index`        | integer | 0       | Pagination offset                         |
| `sortField`    | integer | 2       | Sort: 1=name, 2=popularity, 3=lastUpdated |
| `sortOrder`    | string  | desc    | Sort order: `asc` or `desc`               |
| `gameId`       | integer | 432     | Game ID (432 = Minecraft)                 |
| `classId`      | integer | 4471    | Class ID (4471 = Modpacks, 6 = Mods)      |
| `gameVersion`  | string  | -       | Filter by game version                    |

**Response**

```json
{
  "count": 5,
  "total_count": 150,
  "index": 0,
  "page_size": 10,
  "search_filter": "rlcraft",
  "results": [
    {
      "id": 285109,
      "name": "RLCraft",
      "slug": "rlcraft",
      "summary": "A modpack focused on...",
      "download_count": 20000000,
      "website_url": "https://...",
      "logo_url": "https://...",
      "categories": ["Adventure", "Hardcore"],
      "latest_files": []
    }
  ]
}
```

---

### Get Project Details

```
GET /api/client/v1/curseforge/projects/{projectId}
```

Get details for a specific CurseForge project.

**Path Parameters**

| Parameter   | Type    | Description           |
| ----------- | ------- | --------------------- |
| `projectId` | integer | CurseForge project ID |

**Response**

```json
{
  "id": 285109,
  "name": "RLCraft",
  "slug": "rlcraft",
  "summary": "A modpack focused on...",
  "download_count": 20000000,
  "date_created": "2019-07-01T00:00:00Z",
  "date_modified": "2024-01-15T10:00:00Z",
  "date_released": "2024-01-10T00:00:00Z",
  "website_url": "https://...",
  "wiki_url": null,
  "issues_url": null,
  "source_url": null,
  "logo_url": "https://...",
  "logo_thumbnail_url": "https://...",
  "categories": [{ "id": 1, "name": "Adventure", "slug": "adventure" }],
  "authors": [{ "id": 123, "name": "Shivaxi", "url": "https://..." }],
  "latest_files": [
    {
      "id": 456789,
      "display_name": "RLCraft 2.9.3",
      "file_name": "RLCraft-2.9.3.zip",
      "file_date": "2024-01-10T00:00:00Z",
      "game_versions": ["1.12.2"],
      "server_pack_file_id": 456790
    }
  ],
  "latest_files_indexes": []
}
```

---

### List Project Versions

```
GET /api/client/v1/curseforge/projects/{projectId}/versions
```

List available files/versions for a CurseForge project.

**Path Parameters**

| Parameter   | Type    | Description           |
| ----------- | ------- | --------------------- |
| `projectId` | integer | CurseForge project ID |

**Query Parameters**

| Parameter       | Type    | Default | Description             |
| --------------- | ------- | ------- | ----------------------- |
| `pageSize`      | integer | 10      | Results per page (1-50) |
| `index`         | integer | 0       | Pagination offset       |
| `gameVersion`   | string  | -       | Filter by game version  |
| `modLoaderType` | string  | -       | Filter by mod loader    |

**Response**

```json
{
  "project_id": "285109",
  "count": 10,
  "total_count": 50,
  "index": 0,
  "page_size": 10,
  "versions": [
    {
      "id": 456789,
      "display_name": "RLCraft 2.9.3",
      "file_name": "RLCraft-2.9.3.zip",
      "file_date": "2024-01-10T00:00:00Z",
      "file_size": 104857600,
      "download_count": 500000,
      "download_url": "https://...",
      "game_versions": ["1.12.2"],
      "release_type": "release",
      "server_pack_file_id": 456790,
      "parent_project_file_id": null,
      "is_server_pack": false,
      "dependencies": []
    }
  ]
}
```

**Error Codes**

| Status | Code                 | Description                       |
| ------ | -------------------- | --------------------------------- |
| 400    | `MISSING_PROJECT_ID` | Project ID not provided           |
| 404    | `PROJECT_NOT_FOUND`  | Project doesn't exist             |
| 500    | `CONFIG_ERROR`       | CurseForge API key not configured |

---

### Security Notes

**Path Validation**

All file paths are validated to prevent:

- Path traversal attacks (`..` not allowed)
- Null byte injection

**Sensitive File Protection**

Files matching these patterns have their diffs hidden in audit logs:

- `.env`, `.env.*`
- Files containing `secret`, `credential`, `password`
- `.key`, `.pem`, `.p12`, `.pfx`
- `id_rsa`, `id_ed25519`
- `.htpasswd`
