# SCTrackerAgent.exe — Reverse Engineering Analysis

## Binary Overview

| Property | Value |
|---|---|
| File | SCTrackerAgent.exe |
| Size | ~47 MB |
| Format | PE64 (x86-64 Windows) |
| Bundler | pkg (Vercel) v5 |
| Runtime | Node.js (bundled) |
| Language | TypeScript (compiled to JS, then V8 bytecode) |
| Version | SCTracker v2.1.4 |

The executable is a [pkg](https://github.com/vercel/pkg)-bundled Node.js application.
pkg embeds a compressed VFS (virtual filesystem) and the Node.js runtime into a single PE.
The VFS was located at file offset `0x2eb8033` and contains 182 entries.
The blob base (VFS content start) is at file offset `0x28ee0ba`.

---

## Module Structure

The app source lives in `C:\snapshot\agent\dist\agent\src\` (the pkg virtual path).
TypeScript source was compiled to JS with `tsc`, then compiled to V8 bytecode by pkg.
No plain JS source is recoverable — only bytecode with embedded string constants.

### Source files in VFS

| File | Bytecode size | Role |
|---|---|---|
| `main.js` | 14 744 B | Entry point, orchestration |
| `config.js` | 6 816 B | Config file I/O, path detection |
| `watcher.js` | 5 368 B | Game.log file watcher |
| `parser/index.js` | 3 008 B | `LogParser` class, line dispatch |
| `client.js` | 10 168 B | WebSocket client (`AgentClient`) |
| `tray.js` | 12 768 B | Systray subprocess management |
| `tray-binary.js` | 4 851 032 B | `tray_windows_release.exe` as base64 |
| `parser/patterns.js` | 15 064 B | SC log regex patterns |

Third-party dependencies bundled: `chokidar` 3.6.0, `systray2`, `ws`.

---

## Startup Sequence

```
[agent] SC Tracker starting...
  → readConfig()             read AppData\SCTracker\config.json
  → detectLogPath()          probe known SC install paths
      C:\Roberts Space Industries\StarCitizen\LIVE\Game.log
      D:\Roberts Space Industries\StarCitizen\LIVE\Game.log
      C:\Program Files\Roberts Space Industries\StarCitizen\LIVE\Game.log
      (falls back to C: path with advisory log message)
  → If no token in config:
      [agent] First run — starting Discord auth...
      open browser → OAuth callback on localhost:9242
      store token → writeConfig()
      [agent] Config saved. Starting tracker...
  → If serverUrl is stale (http:// instead of wss://):
      [agent] Updating stale serverUrl: <new>
  → [agent] Watching: <logPath>
  → readAckCursor()          load saved byte-offset cursors from disk
      [agent] Loaded ack cursor: session@<N>, confirmed@<M>
  → new LogParser()
  → new AgentClient(serverUrl, token, ...)
  → client.connect()
  → watcher.watchLog(logPath)
  → createTray(dashboardUrl, logPath, ...)
      [agent] Tray spawned. If the icon is missing, check the Windows tray
               overflow (the ^ chevron) and drag SC Tracker out.
      [agent] Running. Right-click the tray icon to open the dashboard or quit.
```

Uncaught exceptions and unhandled rejections are caught globally and logged as
`[agent] Fatal: <err>` before the process exits.

---

## AppData Layout

All persistent files live in `%APPDATA%\SCTracker\`:

| Path | Contents |
|---|---|
| `config.json` | JWT token, log path, server URL, local port |
| `agent.log` | Redirected stderr log (created on startup) |
| `bin\<SYSTRAY2_VERSION>\tray_windows_release.exe` | Extracted tray binary (versioned subdirectory) |
| `launch.vbs` | Silent VBS launcher for startup-on-boot |

Observed on disk: `bin\2.1.4\tray_windows_release.exe` (3,637,760 bytes).

`config.json` schema:
```json
{
  "token":     "<JWT>",
  "logPath":   "D:\\Games\\StarCitizen\\LIVE\\Game.log",
  "serverUrl": "wss://sc-tracker.replit.app/ws/agent",
  "localPort": 9242
}
```

---

## WebSocket Protocol (Agent ↔ Server)

### Connection

- URL: `wss://sc-tracker.replit.app/ws/agent`
- Also uses `/tracker` path (likely for HTTP dashboard)
- Reconnect: exponential backoff, constants `RECONNECT_BASE_MS`, `RECONNECT_MAX_MS`
- Reconnect interval resets after `RECONNECT_RESET_AFTER_MS` of stable connection
- Heartbeat: ping every `PING_INTERVAL_MS`, disconnect if no pong within `PONG_TIMEOUT_MS`

### Message Types

#### Agent → Server

**Authentication** (sent immediately on connect):
```json
{ "type": "auth", "token": "<JWT>" }
```
*(type field name inferred; token field confirmed from config structure)*

**Event payload** (parsed SC log events):
```json
{ "type": "payload", "events": [ ... ], "sessionStartOffset": <N> }
```
*(exact envelope field names inferred from `stringify`, `payload`, `sessionStartOffset` strings)*

#### Server → Agent

**Auth success**:
```json
{ "type": "auth_ok", "userId": "<id>" }
```

**Auth failure**:
```json
{ "type": "auth_error", "message": "Auth failed: ..." }
```

**ACK / cursor update**:
```json
{ "type": "ack", "batchEndOffset": <N>, "confirmedOffset": <M> }
```
*(type name inferred; field names `batchEndOffset` and `confirmedOffset` confirmed)*

### Observed Reconnect Behavior

From agent.log, the agent re-authenticates every ~5 minutes (`[agent] Authenticated as user 63`).
This indicates the Replit-hosted WebSocket server closes connections periodically (~5 min).
The agent detects the close, reconnects with exponential backoff, and re-authenticates.
`userId` in the `auth_ok` response is a small integer (observed: `63`).

### Event Queue

- Events are queued in `pendingEvents` while disconnected
- Queue limit: `MAX_QUEUE_SIZE`; when full, oldest `QUEUE_DROP_BATCH` events are dropped
  - Log: `Queue overflow: dropped <N> oldest events (server unreachable too long)`
- On reconnect + auth: `flushQueue()` sends pending events

---

## Cursor / ACK System

The agent tracks read progress through Game.log to avoid re-sending events after restart.

```
sessionStartOffset  ─── byte offset of "Log started on" header in current Game.log
                         (scanned on startup using low-level I/O: openSync/readSync)
confirmedOffset     ─── last byte offset ACK'd by server
batchEndOffset      ─── end of most recently sent batch (from server ACK)
```

On watcher startup:
1. `findSessionStartOffset()` — scans Game.log backwards for latest `Log started on` marker
2. Compare `confirmedOffset` to `sessionStartOffset`:
   - If `confirmedOffset` > `sessionStartOffset`: resume from `confirmedOffset`
   - Log: `[watcher] Resuming from confirmed offset (skipping <N> already-acked bytes)`
   - Otherwise: start from `sessionStartOffset` (new session)
3. `writeAckCursor()` — persists cursors to disk after each server ACK

---

## Game.log File Watcher (`watcher.js`)

Uses **chokidar** for file change detection with these options:
```js
{
  persistent:       true,
  usePolling:       <configured>,
  interval:         <pollInterval>,
  awaitWriteFinish: { stabilityThreshold: <N> }
}
```

Uses Node.js **readline** with `createReadStream` + `createInterface` for line-by-line parsing.

Calls `parseLine(line, lineOffset)` for each new line, then calls `client.enqueue(event)`.

---

## Log Parser (`parser/patterns.js` + `parser/index.js`)

### Timestamp

Every Game.log line starts with an ISO-8601 timestamp:
```
TIMESTAMP_RE = ^<(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z)>
```

### Event Patterns

#### `LOCATION_CHANGE`
Log tag: *(location change context)*
```
Regex: on\. Landing \[(\d+)\] -> \[(\d+)\]\. Location \[(\d+)\] -> \[(\d+)\]
Fields:
  playerName      (from surrounding context)
  fromLandingId   \1
  toLandingId     \2
  fromLocationId  \3
  toLocationId    \4
```

#### `ATTACHMENT_RECEIVED`
Log tag: `<AttachmentReceived>`
```
Regex: Player\[([^\]]+)\] Attachment\[([^,]+), ([^,]+), (\d+)\] Status\[([^\]]+)\] Port\[([^\]]+)\]
Fields:
  playerName      \1
  attachmentName  \2
  itemClass       \3
  itemId          \4
  status          \5
  port            \6
```

#### `ITEM_STORED`
Log tag: `<StoreItem>` + `store '`
```
Regex: Request\[(\d+)\] store '([^']+)' \[(\d+)\] by '([^']+)' \[(\d+)\].*Class\[([^\]]+)\]
Fields:
  requestId   \1
  itemName    \2  (store name)
  (storeId)   \3
  (byName)    \4
  (byId)      \5
  itemClass   \6
```

#### `MISSION_START`
Log tag: `<CSubsumptionMissionComponent::CreateMissionInstance>`
```
Regex: Creating subsumption mission module (\S+) with seed (\d+) and EntityId (\d+)
Fields:
  missionType  \1
  seed         \2
  entityId     \3
```

#### `MISSION_END`
Log tag: `<CSubsumptionMissionComponent::StopMissionLogic>`
```
Regex: Stopping subsumption mission module with EntityId (\d+)
Fields:
  entityId  \1
```

#### `CONTRACT_COMPLETE`
Log context: `Added notification "(?:Mission|Contract) Complete:\s*([^:]*?):\s*" \[(\d+)\]`
```
Strip prefix: ^(?:Mission|Contract) Complete:\s*
Fields:
  contractName  \1  (name after stripping prefix)
  entityId      \2
```

#### `SHIP_CLAIM`
Log tag: `<CWallet::ProcessClaimToNextStep>` + `New Insurance Claim Request`
```
Regex: entitlementURN: ([^,]+), requestId : (\d+)
Fields:
  entitlementUrn  \1
  requestId       \2
```

#### `SHIP_NEARBY`
Log tag: `[CItemResourceHost::AddHostedNode]` + `-- Host`
```
Regex: -- Host\s+:(\w+)_(\d+)
Fields:
  shipClass  \1
  hostId     \2
State: seenShipHosts (set) — each (shipClass, hostId) pair reported only once per session
```

### Parser State (`LogParser` class)

```
sessionStartEmitted       bool    — whether SESSION_START event was sent
seenShipHosts             Set     — dedup for SHIP_NEARBY per session
filteredMissionEntityIds  Set     — entity IDs for missions to ignore
```

---

## Tray Integration (`tray.js`)

### Binary Extraction

On first run (or if missing), the tray binary is extracted from base64:
```js
// tray-binary.js exports:
TRAY_BINARY_BASE64 = "<4.6 MB base64 string of tray_windows_release.exe>"

// tray.js:
ensureBinaryStaged() {
  dest = getAppDataDir() + "\tray_windows_release.exe"
  if (!existsSync(dest))
    writeFileSync(dest, Buffer.from(TRAY_BINARY_BASE64, 'base64'))
}
```

### Startup on Boot

Registry key: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
Registry value: (app name, confirmed by `REG_KEY`, `REG_VALUE` constants)

Value data: `wscript.exe "<path>\launch.vbs"`

`launch.vbs` content (written by `writeStartupVbs`):
```vbscript
Dim sh
Set sh = CreateObject("WScript.Shell")
sh.Run Chr(34) & "C:\Users\...\SCTrackerAgent.exe" & Chr(34), 0, False
```

Registry operations use `execSync` with `reg query / reg add / reg delete`.

### Tray Menu Layout

Menu items by flat index (seq_id), confirmed from live log:
```
seq_id=0   "Open Dashboard"              IDX_DASHBOARD
seq_id=1   "View Agent Log"              IDX_VIEW_LOG    ← confirmed
seq_id=2   "Watching: <logPath>"         (status, disabled)
seq_id=3   "Game.log: <status>"          (status, disabled)
seq_id=4   "Start on Boot"               IDX_STARTUP     ← confirmed
           [separator]
seq_id=?   "Quit"                        IDX_QUIT
```
*(Exact positions of separators in seq_id sequence unclear; separators may or may not consume an index.)*

### Tray Click Handlers

| seq_id | Item | Action |
|---|---|---|
| 0 | Open Dashboard | `openUrl(dashboardUrl)` — `https://sc-tracker.replit.app/tracker` |
| 1 | View Agent Log | `execSync('notepad.exe "<agent.log path>"')` |
| 4 | Start on Boot | toggle `enableStartup()` / `disableStartup()` |
| ? | Quit | `onQuit()` → `client.disconnect()`, `watcher.destroy()`, `process.exit()` |

IPC with tray binary uses the protocol documented in `TRAY_IPC_PROTOCOL.md`.

Tray error/exit handlers:
- `[tray] helper error: <err>`
- `[tray] helper exited (code=<N>, signal=<S>). Tray icon gone.`
- `[tray] ready failed: <err>`
- `[tray] click seq_id=<N> title="<title>"`

---

## Discord OAuth Flow

Triggered on first run (no token in config):
1. Listen on `http://127.0.0.1:<localPort>/token` (default port 9242)
2. Open browser to OAuth entry point
3. If browser open fails: `[agent] Could not open browser automatically. Copy the URL and open it manually.`
4. Log: `[agent] Opening browser for Discord auth...`
5. Log: `[agent] Auth URL: <url>`
   - Observed URL: `https://sc-tracker.replit.app/auth/discord?callback=http%3A%2F%2F127.0.0.1%3A9242%2Ftoken`
6. If port in use: `[agent] Port <N> is already in use.`
7. Server redirects to `http://127.0.0.1:9242/token` with JWT token after Discord consent
8. On failure: `[agent] Auth failed: <err>` / `[agent] Restart the agent to try again.`
9. On success: write `token` to `config.json`, proceed to tracker startup

Dashboard URL: `https://sc-tracker.replit.app/tracker`

---

## Key Constants (from bytecode strings)

| Constant | Role |
|---|---|
| `RECONNECT_BASE_MS` | Base reconnect delay (exponential backoff) |
| `RECONNECT_MAX_MS` | Maximum reconnect delay cap |
| `RECONNECT_RESET_AFTER_MS` | Reset backoff after this long connected |
| `PING_INTERVAL_MS` | WebSocket heartbeat interval |
| `PONG_TIMEOUT_MS` | Max wait for pong before disconnect |
| `MAX_QUEUE_SIZE` | Max events held while disconnected |
| `QUEUE_DROP_BATCH` | Events dropped at once when queue overflows |
| `SYSTRAY2_VERSION` | Expected version of systray2 binary protocol |
| `REG_KEY` | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` |

---

## Process Lifecycle

```
main.js
  ├── require('./config')         config.json I/O
  ├── require('./watcher')        chokidar + readline
  ├── require('./parser/index')   LogParser class
  ├── require('./client')         AgentClient (WebSocket)
  └── require('./tray')           systray2 subprocess
         └── spawn tray_windows_release.exe  (IPC via stdio JSON)
```

Shutdown path (`onQuit` or `SIGINT`):
```
[agent] Quitting...
client.disconnect()
watcher.destroy()
tray.process.destroy()
process.exit(0)
```

---

## Tray Binary Embedded in Agent

`tray-binary.js` contains `TRAY_BINARY_BASE64` — the full `tray_windows_release.exe`
encoded as base64 (4.85 MB bytecode → ~3.6 MB decoded binary).
The Go binary's Go build ID is embedded: `AjqVB04y6Y9QdPq77KG2/PxC8QNYitmye9vPb_DlG/0ofNyRX-02PCJmZT6jw7/Y35E15aty6fQU_1dW8ZX`.
This matches the separately distributed `tray_windows_release.exe`.

---

---

## Server-Side (`tracker.noharmsc.org`)

### Infrastructure

| Property | Value |
|---|---|
| Primary domain | `tracker.noharmsc.org` |
| Legacy alias | `sc-tracker.replit.app` (301 → primary) |
| Health check | `GET /health` → `{"ok":true}` |
| Frontend | Server-rendered HTML (no JS bundle), single CSS file `/tracker.css` |
| Session auth | Discord OAuth session cookie (not the agent JWT) |

### Dashboard Page Structure

Reconstructed from `tracker.css` (17 KB, `server/public/tracker.css`).
The app is **server-rendered HTML** — no frontend JS bundle. Two themes: `dark-purple` (default) and `dark-amber`.

#### Pages / Routes

| Route | Page | Key components |
|---|---|---|
| `/tracker` | Login / Home | `.login-card`, sign-in button, stats row |
| `/tracker/sessions` | My Sessions | `.hero-card` (latest session), session table, `.pagination` |
| `/tracker/leaderboard` | Leaderboard | `.rank-num` (gold/silver/bronze), player table, `.row-me` (self-highlight) |
| `/tracker/setup` (inferred) | Setup & Download | `.setup-download-card`, `.btn-download`, `.setup-steps`, `.setup-faq`, `.setup-tree`, `.setup-token-hint` |
| `/tracker/profile` (inferred) | User Profile | `.profile-header`, `.profile-avatar`, `.profile-name`, `.profile-meta` |

#### Nav bar

```
[SC Tracker brand] [sessions] [leaderboard] [setup] ... [theme toggle] [avatar] [username] [logout]
```
Mobile: hamburger button (`.nav-burger`) collapses nav links.

#### Stat bar (6-column grid)

`.stat-bar` appears on sessions/overview pages, showing 6 counters with accent-coloured values.
Likely: Sessions · Events · Missions · Contracts · Ship Claims · Ships Nearby (or similar).

#### Hero session card

`.hero-card` with `.hero-stats` (4-column grid) — surfaces key metrics for the most recent/selected session:
likely Duration, Events, Missions, Contracts.

#### Timeline

`.timeline` with `.timeline-row` entries (time | dot | description).
Dot colours map to event categories:
- **purple** (`#a855f7`) — general/location events
- **green** (`#22c55e`) — ship/positive events
- **amber** (`#f59e0b`) — mission/contract events
- **blue** (`#3b82f6`) — informational events

#### Highlight feed

`.highlight-feed` rows with left-border severity colouring:
- `epic` — gold border (`#facc15`)
- `major` — accent purple/amber border
- `minor` — blue border
- `info` — gray border, 85% opacity

Each row: icon + title + description + relative time.

#### Event list

`.event-list` compact rows: `[EVENT_TYPE]  description  time`
`event-type` label is min-width 80px, monospace-style uppercase.

#### Setup page

`.setup-download-card` (full-width) → `.btn-download` → download `SCTrackerAgent.exe`
`.setup-steps` numbered list with step circles
`.setup-tree` monospace file-tree showing AppData layout
`.setup-token-hint` hint about token/auth after first install
`.setup-faq` definition list of FAQs

---

### Confirmed HTTP Routes

| Route | Auth required | Response |
|---|---|---|
| `GET /tracker` | No | Login page (server-rendered, 200) |
| `GET /tracker/sessions` | Yes → login | Login redirect |
| `GET /tracker/leaderboard` | Yes → login | Login redirect |
| `GET /auth/discord?dashboard=1` | No | 302 → Discord OAuth |
| `GET /auth/discord?callback=<url>` | No | Initiates agent OAuth |
| `GET /auth/discord/callback?code=...&state=...` | No | OAuth callback handler |
| `GET /tracker.css` | No | 200 (16 970 bytes) |
| `GET /health` | No | `{"ok":true}` |
| `WS /ws/agent` | JWT in first message | WebSocket (agent connection) |

### Discord OAuth

| Property | Value |
|---|---|
| Discord app client_id | `1497750335601639464` |
| OAuth redirect_uri | `https://sc-tracker.replit.app/auth/discord/callback` |
| OAuth scopes | `identify` (username + Discord ID only, no email) |
| Agent flow | `GET /auth/discord?callback=http://127.0.0.1:9242/token` |
| Dashboard flow | `GET /auth/discord?dashboard=1` |
| State parameter | JSON base64: `{dashboard:true, nonce:"...", sig:"..."}` |

### JWT Claims (agent token)

| Field | Value/Notes |
|---|---|
| `alg` | HS256 |
| `discordId` | Discord user snowflake |
| `tv` | `1` (token version) |
| `iat` | Issue timestamp |
| `exp` | `iat + 365 days` (1-year validity) |

The JWT is exclusively for `WS /ws/agent` authentication. The web dashboard uses a separate session cookie issued after Discord OAuth.

### WebSocket Protocol (confirmed by live connection)

**Agent → Server**:
```json
{ "type": "auth", "token": "<JWT>" }
```

**Server → Agent** (auth success):
```json
{ "type": "auth_ok", "userId": 63 }
```
`userId` is a small integer (server's internal ID for the user).

**Server → Agent** (auth failure):
```json
{ "type": "auth_error" }
```

**Agent → Server** (event batch, exact format inferred from bytecode):
```json
{
  "type": "payload",
  "sessionStartOffset": <N>,
  "batchEndOffset": <N>,
  "events": [ { "type": "<EVENT_TYPE>", "timestamp": "<ISO8601>", ... } ]
}
```

**Server → Agent** (ACK, field names from bytecode — not observed live):
```json
{ "type": "ack", "batchEndOffset": <N>, "confirmedOffset": <N> }
```

**WS-level**: Server sends WebSocket ping frames every ~30 seconds (Replit infrastructure keep-alive). Client auto-responds with pong. No JSON-level heartbeat observed from server.

**Reconnect**: Server closes connections approximately every 5 minutes (Replit free tier limit). Agent reconnects and re-authenticates automatically via `scheduleReconnect` with exponential backoff.

---

## Pattern Validation (against real Game.log backups)

Tested against 13 session logs (Oct 2025 → May 2026, SC 3.24–4.7). Largest session: 1.8 MB / 7007 lines.

| Pattern | Status | Notes |
|---|---|---|
| `ATTACHMENT_RECEIVED` | ✅ Confirmed — 159 hits | Player `DatGuySnowfox` equipping/unequipping gear |
| `ITEM_STORED` | ✅ Confirmed — 12 hits | Items stored in personal inventory |
| `MISSION_START` | ✅ Confirmed — 1 hit | `FrontendHangar.xml` (hangar mission) |
| `MISSION_END` | ✅ Confirmed — 2 hits | EntityId matching MISSION_START |
| `SHIP_CLAIM` | ✅ Confirmed — 2 hits | Insurance claim URN + requestId |
| `LOCATION_CHANGE` | ⏳ Not yet triggered | Regex correct; needs quantum travel to a landing zone — never appeared in any archived log |
| `CONTRACT_COMPLETE` | ⏳ Not yet triggered | Needs `"Mission/Contract Complete: ..."` notification — user has only seen `"Hangar Request Completed"` |
| `SHIP_NEARBY` | ⏳ Not yet triggered | `CItemResourceHost::AddHostedNode` never appeared in any log — likely requires specific multiplayer/ship proximity scenario |

Notifications actually seen in logs (for context): Entered/Exited Monitored Space, ArcCorp Jurisdiction, UEE Jurisdiction, Rough & Ready Jurisdiction, Armistice Zone enter/leave, Medical Bed, Medical Device tips, Quantum Travel obstruction, Hangar Request Completed.

Game.log session header fields captured by parser:
- `Log started on` — `findSessionStartOffset` anchor
- `FileVersion: 4.7.178.50402` → `_FILE_VERSION`
- `Branch: sc-alpha-4.7.0` → `_BRANCH`
- `Changelist: 11715810`

---

## Analysis Notes

- All app JS source files are bytecode-only in the pkg bundle (no readable source).
  Node_modules (chokidar, ws, systray2) have plain JS source in VFS key `"1"`.
- String constants are fully readable in the bytecode, enabling this analysis.
- The `_FILE_VERSION` and `_BRANCH` strings suggest the parser also captures
  the Game.log header metadata (SC version and build branch).
- `seenShipHosts` is a per-session Set — the `SHIP_NEARBY` event fires at most once
  per unique (shipClass, hostId) pair per game session, preventing flood from
  repeated `AddHostedNode` calls.
