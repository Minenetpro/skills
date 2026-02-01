# Minenet Client API

Complete API reference and workflow guide for managing Minecraft servers on Minenet.pro. This document is designed for AI agents and automated systems.

## Overview

The Minenet Client API provides programmatic access to:

- **Server Management**: Create, list, power control, and reinstall servers
- **File Operations**: Read, write, delete, and manage server files
- **Console**: Send commands and read console output
- **Plugin/Mod Installation**: Search and install from Modrinth
- **Modpack Installation**: Search and install from CurseForge

## Base URL

```
https://minenet.pro/api/client/v1
```

All paths in this document are relative to this base URL.

## Authentication

Include your API token in the `Authorization` header:

```
Authorization: Bearer mnp_<token>
```

Tokens are created in Team Settings and provide full access to all servers belonging to that team.

### Token Format

- **Prefix**: `mnp_` (Minenet Pro)
- **Length**: 36 characters total
- **Expiration**: Never expires (valid until revoked)

### Rate Limiting

- **Limit**: 100 requests per minute per token
- **Scope**: Per-token (not per-team)
- **429 Response**: Includes `Retry-After` header

---

# Workflows

This section provides step-by-step instructions for common tasks. Follow these workflows to accomplish goals efficiently.

## Server Discovery

Before any server operation, you must discover available servers:

```http
GET /servers
```

**Response:**

```json
{
  "servers": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "My Server",
      "status": "running",
      "address": "play.example.com:25565"
    }
  ],
  "count": 1
}
```

**Decision Logic:**

- If **one server**: Use it automatically
- If **multiple servers**: Ask user which server to operate on
- Always mention which server you're acting on before operations

---

## Plugin/Mod Installation (Modrinth)

Install plugins and mods from Modrinth using this workflow.

### Directory Conventions

| Server Type             | Directory  |
| ----------------------- | ---------- |
| Paper, Spigot, Bukkit   | `/plugins` |
| Fabric, Forge, NeoForge | `/mods`    |

### Installation Steps

**Step 1: Search for the project**

```http
GET /modrinth/search?query=LuckPerms&loaders=paper
```

**Step 2: Analyze results and select project**

Decision logic:

- **Clear match** (exact/close title, high downloads): Proceed automatically
- **Ambiguous** (multiple similar results): Ask user to choose
- **No results**: Inform user, suggest alternatives

**Step 3: Get compatible versions**

```http
GET /modrinth/projects/{slug}/versions?loaders=paper&game_versions=1.21.4
```

Select the first version that:

1. Matches the server's loader (paper, fabric, etc.)
2. Matches the server's Minecraft version
3. Is marked as `release` (prefer over `beta`/`alpha`)

**Step 4: Get download URL**

From the version response, extract:

- `files[0].url` (the primary file URL)
- `files[0].filename` (the filename)
- `files[0].size` (file size in bytes)

**Step 5: Download to server**

```http
POST /servers/{id}/files/pull
Content-Type: application/json

{
  "url": "https://cdn.modrinth.com/data/.../LuckPerms-5.4.102.jar",
  "directory": "/plugins",
  "filename": "LuckPerms-5.4.102.jar"
}
```

**Step 6: Verify installation**

```http
GET /servers/{id}/files/list?directory=/plugins
```

Confirm the file appears in the directory listing.

**Step 7: Restart server (if running)**

```http
POST /servers/{id}/power
Content-Type: application/json

{"action": "restart"}
```

### Complete Example

```bash
# 1. Search
curl -H "Authorization: Bearer $TOKEN" \
  "https://minenet.pro/api/client/v1/modrinth/search?query=LuckPerms&loaders=paper"

# 2. Get versions for the project
curl -H "Authorization: Bearer $TOKEN" \
  "https://minenet.pro/api/client/v1/modrinth/projects/luckperms/versions?loaders=paper&game_versions=1.21.4"

# 3. Download to server
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://cdn.modrinth.com/.../LuckPerms-5.4.102.jar", "directory": "/plugins", "filename": "LuckPerms-5.4.102.jar"}' \
  "https://minenet.pro/api/client/v1/servers/$SERVER_ID/files/pull"
```

---

## Modpack Installation (CurseForge)

Install modpacks from CurseForge. This reinstalls the server with the modpack.

### Installation Steps

**Step 1: Search for modpack**

```http
GET /curseforge/search?searchFilter=RLCraft&classId=4471
```

Note: `classId=4471` filters to modpacks only.

**Step 2: Analyze and select**

Decision logic (same as plugins):

- **Clear match**: Proceed automatically
- **Ambiguous**: Ask user to choose

**Step 3: Get available versions**

```http
GET /curseforge/projects/{projectId}/versions
```

**Step 4: Find server pack version**

Look for a version with `server_pack_file_id` field. This is **required** for server installation.

```json
{
  "id": 456789,
  "display_name": "RLCraft 2.9.3",
  "server_pack_file_id": 456790, // <-- Use THIS as version_id
  "game_versions": ["1.12.2"]
}
```

If no `server_pack_file_id` exists, the modpack doesn't support server installation.

**Step 5: Install modpack**

```http
POST /servers/{id}/modpacks/curseforge/install
Content-Type: application/json

{
  "project_id": 285109,
  "version_id": 456790
}
```

**Step 6: Wait for installation**

Modpack installation takes 60-120 seconds. Poll server status:

```http
GET /servers
```

Wait until `status` changes from `installing` to `offline` or `running`.

**Step 7: Accept EULA**

After modpack installation, EULA must be accepted:

```http
POST /servers/{id}/files/write?file=/eula.txt
Content-Type: text/plain

eula=true
```

**Step 8: Start server**

```http
POST /servers/{id}/power
Content-Type: application/json

{"action": "start"}
```

### Java Version

Java version is auto-detected based on Minecraft version:

- MC 1.21+ → Java 21
- MC 1.17-1.20 → Java 17
- MC 1.12-1.16 → Java 8/11

---

## Server Reinstallation

Change the server software type (Paper, Forge, etc.) or Minecraft version.

**WARNING**: This is a **destructive operation** that wipes ALL server files. Always confirm with the user before proceeding.

### Steps

**Step 1: Get available software**

```http
GET /servers/software
```

Returns available types: `paper`, `forge`, `neoforge`, `vanilla`, `fabric`

**Step 2: Confirm with user**

Explain that reinstallation will:

- Delete all server files
- Reset configuration
- Remove installed plugins/mods

**Step 3: Reinstall**

```http
POST /servers/{id}/reinstall
Content-Type: application/json

{
  "software_type": "paper",
  "minecraft_version": "1.21.4"
}
```

**Step 4: Wait for installation**

Installation takes 60-120 seconds. Poll server status.

**Step 5: Accept EULA**

```http
POST /servers/{id}/files/write?file=/eula.txt
Content-Type: text/plain

eula=true
```

**Step 6: Start server**

```http
POST /servers/{id}/power
Content-Type: application/json

{"action": "start"}
```

---

## Server Creation

Create a new Minecraft server.

### Steps

**Step 1: Gather requirements**

Required information:

- **name**: Server name (1-64 characters)
- **memory_gb**: RAM allocation (1-32 GB)
- **disk_gb**: Disk space (10-500 GB, default 10, first 10GB free)
- **region_id**: Currently only `dallas`

### Pricing

- RAM: $3/GB per month
- Disk: $0.04/GB per month (first 10GB free)
- Daily cost = monthly cost / 30

**Step 2: Create server**

```http
POST /servers
Content-Type: application/json

{
  "name": "My New Server",
  "memory_gb": 4,
  "disk_gb": 20,
  "region_id": "dallas"
}
```

**Step 3: Wait for provisioning**

Server provisioning takes ~60 seconds. Poll until status is no longer `installing`.

**Step 4: (Optional) Change software**

New servers start with Paper (latest Minecraft). Use the reinstall workflow to change software type.

### Limits

- Maximum 5 servers per user (credit-billed)
- Requires active subscription

---

## Configuration Changes

Modify server configuration files.

### Steps

**Step 1: Read current config**

```http
GET /servers/{id}/files/contents?file=/server.properties
```

**Step 2: Parse and modify**

Parse the file content, make changes, preserve other settings.

**Step 3: Write updated config**

```http
POST /servers/{id}/files/write?file=/server.properties
Content-Type: text/plain

# Minecraft server properties
server-port=25565
gamemode=survival
difficulty=normal
max-players=20
```

**Step 4: Restart server**

```http
POST /servers/{id}/power
Content-Type: application/json

{"action": "restart"}
```

---

## Detecting Server Type

Determine what type of server is running.

### Method 1: Check console output

```http
GET /servers/{id}/console?size=100
```

Look for indicators:

- `[Paper]` or `Paper version` → Paper
- `Forge Mod Loader` → Forge
- `NeoForge` → NeoForge
- `Fabric Loader` → Fabric

### Method 2: Check file structure

```http
GET /servers/{id}/files/list?directory=/
```

- Has `/plugins` directory → Paper/Spigot/Bukkit
- Has `/mods` directory → Forge/Fabric/NeoForge
- Has `fabric.mod.json` → Fabric
- Has `forge-*.jar` → Forge

### Method 3: Run version command

```http
POST /servers/{id}/command
Content-Type: application/json

{"command": "version"}
```

Then read console output for version info.

---

# Best Practices

## Wait Times

Operations take time. Use these guidelines:

| Operation               | Wait Time | Check Method                 |
| ----------------------- | --------- | ---------------------------- |
| Server start (plugins)  | 10-20s    | Poll `/servers` for status   |
| Server start (modpacks) | 60-120s   | Poll `/servers` for status   |
| Modpack installation    | 60-120s   | Poll `/servers` for status   |
| Server creation         | ~60s      | Poll `/servers` for status   |
| Server reinstall        | 10-30s    | Poll `/servers` for status   |
| File download           | 5-30s     | Check `/files/pull` progress |

**Polling strategy**: Check every 5-10 seconds rather than waiting the full duration at once.

## EULA Acceptance

After reinstall or modpack installation, the EULA is reset to `false`. The server won't start until accepted:

```http
POST /servers/{id}/files/write?file=/eula.txt
Content-Type: text/plain

eula=true
```

Always inform the user that accepting the EULA means they agree to Minecraft's terms.

## Error Handling

Common errors and how to handle them:

| Error                     | Cause                     | Solution                     |
| ------------------------- | ------------------------- | ---------------------------- |
| `SERVER_INSTALLING` (409) | Server is being installed | Wait and retry later         |
| `SERVER_OFFLINE` (502)    | Server not running        | Start server first           |
| `SERVER_NOT_FOUND` (404)  | Invalid server ID         | Re-fetch server list         |
| `FILE_NOT_FOUND` (404)    | File doesn't exist        | Check path, create if needed |
| `RATE_LIMITED` (429)      | Too many requests         | Wait `Retry-After` seconds   |

## Autonomous vs Interactive

When to proceed automatically:

- Search result is an exact or near-exact name match
- Project has significantly more downloads than alternatives
- User specified the exact project name

When to ask the user:

- Multiple similar results with no clear winner
- User's query is ambiguous (e.g., "permissions plugin")
- Destructive operation (reinstall, delete files)
- User explicitly asks for options or recommendations

---

# API Reference

Complete endpoint documentation follows.

## Server Endpoints

### List Servers

```http
GET /servers
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

**Status Values**: `running`, `starting`, `stopping`, `offline`, `installing`, `unknown`

---

### Create Server

```http
POST /servers
```

**Request Body**

```json
{
  "name": "My New Server",
  "memory_gb": 4,
  "disk_gb": 20,
  "region_id": "dallas"
}
```

| Field       | Type    | Required | Default | Description              |
| ----------- | ------- | -------- | ------- | ------------------------ |
| `name`      | string  | Yes      | -       | Server name (1-64 chars) |
| `memory_gb` | integer | Yes      | -       | Memory in GB (1-32)      |
| `disk_gb`   | integer | No       | 10      | Disk in GB (10-500)      |
| `region_id` | string  | No       | dallas  | Region ID                |

**Response**

```json
{
  "ok": true,
  "server_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My New Server",
  "specs": {
    "memory_gb": 4,
    "disk_gb": 20,
    "region_id": "dallas"
  },
  "cost": {
    "estimated_monthly_usd": 12.4
  }
}
```

**Error Codes**

| Status | Code                    | Description                |
| ------ | ----------------------- | -------------------------- |
| 402    | `SUBSCRIPTION_REQUIRED` | No active subscription     |
| 402    | `SERVER_LIMIT_REACHED`  | Maximum 5 servers per user |

---

### List Available Software

```http
GET /servers/software
```

**Response**

```json
{
  "count": 5,
  "software": [
    {
      "id": 15,
      "name": "Paper",
      "type": "paper",
      "docker_images": [
        { "value": "ghcr.io/parkervcp/yolks:java_21", "label": "Java 21" }
      ]
    }
  ]
}
```

---

### Reinstall Server

```http
POST /servers/{id}/reinstall
```

**WARNING**: Destructive operation - wipes all files.

**Request Body**

```json
{
  "software_type": "paper",
  "minecraft_version": "1.21.4"
}
```

| Field               | Type   | Required | Description                                       |
| ------------------- | ------ | -------- | ------------------------------------------------- |
| `software_type`     | string | Yes      | `paper`, `forge`, `neoforge`, `vanilla`, `fabric` |
| `minecraft_version` | string | Yes      | Version like `1.21.4` or `latest`                 |
| `java_image`        | string | No       | Auto-detected if omitted                          |

---

### Install CurseForge Modpack

```http
POST /servers/{id}/modpacks/curseforge/install
```

**Request Body**

```json
{
  "project_id": 285109,
  "version_id": 456790
}
```

| Field        | Type    | Required | Description           |
| ------------ | ------- | -------- | --------------------- |
| `project_id` | integer | Yes      | CurseForge project ID |
| `version_id` | integer | Yes      | Server pack file ID   |

---

### Get Console History

```http
GET /servers/{id}/console?size=100
```

| Parameter | Type    | Default | Range | Description     |
| --------- | ------- | ------- | ----- | --------------- |
| `size`    | integer | 100     | 1-500 | Number of lines |

**Response**

```json
{
  "lines": ["[12:00:00 INFO]: Server started"],
  "count": 1
}
```

---

### Send Power Action

```http
POST /servers/{id}/power
```

**Request Body**

```json
{
  "action": "restart"
}
```

**Valid Actions**: `start`, `stop`, `restart`, `kill`

---

### Send Command

```http
POST /servers/{id}/command
```

**Request Body**

```json
{
  "command": "say Hello from API!"
}
```

---

## File Endpoints

### List Directory

```http
GET /servers/{id}/files/list?directory=/plugins
```

**Response**

```json
{
  "files": [
    {
      "name": "LuckPerms.jar",
      "size": 1234567,
      "is_file": true,
      "is_directory": false,
      "modified_at": "2024-01-15T10:30:00Z"
    }
  ],
  "count": 1
}
```

---

### Read File

```http
GET /servers/{id}/files/contents?file=/server.properties
```

**Response**: Raw file contents (text/plain)

**Limits**: Maximum 10MB file size

---

### Write File

```http
POST /servers/{id}/files/write?file=/config.yml
Content-Type: text/plain

key: value
another_key: another_value
```

---

### Delete Files

```http
POST /servers/{id}/files/delete
Content-Type: application/json

{
  "root": "/",
  "files": ["old-plugin.jar", "backup/"]
}
```

---

### Search Files

```http
GET /servers/{id}/files/search?pattern=*.yml&directory=/plugins
```

**Response**

```json
{
  "results": [
    {
      "name": "config.yml",
      "directory": "/plugins/MyPlugin"
    }
  ],
  "count": 1
}
```

---

### Remote Download

**Start download:**

```http
POST /servers/{id}/files/pull
Content-Type: application/json

{
  "url": "https://example.com/plugin.jar",
  "directory": "/plugins",
  "filename": "plugin.jar"
}
```

**List active downloads:**

```http
GET /servers/{id}/files/pull
```

**Cancel download:**

```http
DELETE /servers/{id}/files/pull/{downloadId}
```

---

### Create Directory

```http
POST /servers/{id}/files/create-directory
Content-Type: application/json

{
  "root": "/",
  "name": "backups"
}
```

---

### Rename/Move Files

```http
POST /servers/{id}/files/rename
Content-Type: application/json

{
  "root": "/",
  "files": [
    { "from": "old-name.txt", "to": "new-name.txt" }
  ]
}
```

---

### Copy File

```http
POST /servers/{id}/files/copy
Content-Type: application/json

{
  "location": "/plugins/config.yml"
}
```

---

### Compress Files

```http
POST /servers/{id}/files/compress
Content-Type: application/json

{
  "root": "/",
  "files": ["world", "world_nether"]
}
```

---

### Decompress Archive

```http
POST /servers/{id}/files/decompress
Content-Type: application/json

{
  "root": "/",
  "file": "backup.tar.gz"
}
```

---

## Modrinth Endpoints

### Search Projects

```http
GET /modrinth/search?query=LuckPerms&loaders=paper&versions=1.21.4
```

| Parameter      | Type    | Default   | Description                              |
| -------------- | ------- | --------- | ---------------------------------------- |
| `query`        | string  | -         | Search text                              |
| `limit`        | integer | 10        | Results (1-100)                          |
| `offset`       | integer | 0         | Pagination offset                        |
| `index`        | string  | relevance | Sort: `relevance`, `downloads`, `newest` |
| `project_type` | string  | -         | `mod` or `plugin`                        |
| `versions`     | string  | -         | Comma-separated MC versions              |
| `loaders`      | string  | -         | Comma-separated: `paper`, `fabric`, etc. |

**Response**

```json
{
  "count": 5,
  "total_hits": 150,
  "results": [
    {
      "project_id": "Vebnzrzj",
      "slug": "luckperms",
      "title": "LuckPerms",
      "description": "A permissions plugin",
      "downloads": 5000000,
      "icon_url": "https://...",
      "project_type": "mod"
    }
  ]
}
```

---

### List Project Versions

```http
GET /modrinth/projects/{slug}/versions?loaders=paper&game_versions=1.21.4
```

| Parameter       | Type   | Description                 |
| --------------- | ------ | --------------------------- |
| `loaders`       | string | Comma-separated loaders     |
| `game_versions` | string | Comma-separated MC versions |
| `featured`      | string | `true` for featured only    |

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
      "loaders": ["paper", "spigot"],
      "game_versions": ["1.21.4", "1.20.1"],
      "version_type": "release",
      "files": [
        {
          "filename": "LuckPerms-5.4.102.jar",
          "url": "https://cdn.modrinth.com/...",
          "primary": true,
          "size": 1234567
        }
      ]
    }
  ]
}
```

---

### Get Version Details

```http
GET /modrinth/versions/{versionId}
```

Returns full version details including changelog and file hashes.

---

## CurseForge Endpoints

### Search Projects

```http
GET /curseforge/search?searchFilter=RLCraft&classId=4471
```

| Parameter      | Type    | Default | Description           |
| -------------- | ------- | ------- | --------------------- |
| `searchFilter` | string  | -       | Search text           |
| `pageSize`     | integer | 10      | Results (1-50)        |
| `index`        | integer | 0       | Pagination offset     |
| `classId`      | integer | 4471    | 4471=Modpacks, 6=Mods |
| `gameVersion`  | string  | -       | Filter by MC version  |

**Response**

```json
{
  "count": 5,
  "total_count": 150,
  "results": [
    {
      "id": 285109,
      "name": "RLCraft",
      "slug": "rlcraft",
      "summary": "A modpack focused on...",
      "download_count": 20000000,
      "logo_url": "https://..."
    }
  ]
}
```

---

### Get Project Details

```http
GET /curseforge/projects/{projectId}
```

---

### List Project Versions

```http
GET /curseforge/projects/{projectId}/versions
```

**Response**

```json
{
  "project_id": "285109",
  "count": 10,
  "versions": [
    {
      "id": 456789,
      "display_name": "RLCraft 2.9.3",
      "file_name": "RLCraft-2.9.3.zip",
      "game_versions": ["1.12.2"],
      "server_pack_file_id": 456790
    }
  ]
}
```

**Important**: Use `server_pack_file_id` as the `version_id` when installing.

---

## Error Codes

| Status | Code                    | Description                  |
| ------ | ----------------------- | ---------------------------- |
| 400    | `INVALID_SERVER_ID`     | Server ID not valid UUID     |
| 400    | `VALIDATION_ERROR`      | Input validation failed      |
| 401    | `INVALID_TOKEN`         | Token not found or revoked   |
| 402    | `SUBSCRIPTION_REQUIRED` | No active subscription       |
| 404    | `SERVER_NOT_FOUND`      | Server doesn't exist         |
| 404    | `FILE_NOT_FOUND`        | File doesn't exist           |
| 409    | `SERVER_INSTALLING`     | Server is being installed    |
| 413    | `FILE_TOO_LARGE`        | File exceeds 10MB            |
| 429    | `RATE_LIMITED`          | Rate limit exceeded          |
| 502    | `SERVER_OFFLINE`        | Server is not running        |
| 502    | `UPSTREAM_ERROR`        | Backend communication failed |

---

## Rate Limits

| Endpoint            | Limit             |
| ------------------- | ----------------- |
| Global              | 100/min per token |
| `files/search`      | 30/min            |
| `files/write`       | 60/min            |
| `files/delete`      | 30/min            |
| `files/pull` (POST) | 10/min            |
| `files/compress`    | 20/min            |
| `files/decompress`  | 20/min            |
