# SCTracker Tray IPC Protocol

Reverse-engineered from `tray_windows_release.exe` (v2.1.4, MD5: `402c2c2425ba5b4475f77c6a0ed91036`).

---

## Binary Overview

| Field | Value |
|-------|-------|
| Format | PE64 (64-bit Windows executable) |
| Language | Go (entry: `_rt0_amd64_windows`) |
| Image base | `0x400000` |
| Image size | `0x3B5000` (~3.7 MB) |
| Entry point | `0x46BEA0` (`_rt0_amd64_windows`) |
| `main.main` | `0x5B09C0` |
| Total functions | 3,787 (3,346 stdlib/runtime, 441 app-level) |
| Systray library | `github.com/getlantern/systray` |

The binary is a **Windows system tray subprocess**. It has no GUI of its own beyond the tray icon and its menu. All behavior is driven by a parent process over a newline-delimited JSON pipe on stdin/stdout.

---

## Architecture

```
Parent Process
     │  stdin  ──►  tray (JSON lines)
     │  stdout ◄──  tray (JSON lines)
     │  stderr ◄──  tray (log/error text)
```

The parent launches the tray as a child process and communicates over its standard streams. The tray never opens network connections, named pipes, or other IPC channels — only stdin/stdout/stderr.

---

## Startup Sequence

```
Parent                          Tray (main.onReady.func2)
  │                               │
  │   [launch subprocess]         │
  │                               ├─ os.signal.Notify(SIGINT, SIGTERM)
  │                               ├─ goroutine: signal handler
  │                               ├─ goroutine: action reader (func2.4)
  │                               │
  │  ◄── {"type": "ready"}\n ─────┤  (stdout)
  │                               │
  ├──── <Menu JSON>\n ───────────►│  (stdin, decoded as main_Menu)
  │                               │
  │                               ├─ SetIcon(base64_decode(menu.icon))
  │                               ├─ SetTooltip(menu.tooltip)
  │                               ├─ for each item: addMenuItem(...)
  │                               └─ goroutine: click event loop
```

After the ready handshake, the **first and only message the parent must send** is a complete `Menu` object that initializes the entire tray. After that, the parent may send `Action` objects at any time to update the tray state.

---

## Message Formats

All messages are **newline-terminated JSON** (`\n` after each JSON object). The JSON encoder uses `encoding/json` with `escapeHTML = true`.

### Tray → Parent (stdout)

#### Ready message
Sent once immediately at startup, before any stdin is read.

```json
{"type": "ready"}
```

#### Click event (`main_ClickEvent`)
Sent each time the user clicks an enabled, visible menu item.

```json
{
  "type":    "clicked",
  "item":    <Item>,
  "seq_id":  42,
  "__id":    7
}
```

| Field | JSON key | Go type | Description |
|-------|----------|---------|-------------|
| Type | `"type"` | string | Always `"clicked"` |
| Item | `"item"` | Item | Snapshot of the clicked item at click time |
| SeqID | `"seq_id"` | int64 | Index of this item in the flat item list (0-based) |
| InternalID | `"__id"` | int64 | Value of the item's `__id` field, used for subsequent `update-item` targeting |

---

### Parent → Tray (stdin)

#### Initial menu (`main_Menu`)
The very first message. Must arrive before any `Action` messages.

```json
{
  "icon":           "<base64-PNG>",
  "title":          "SCTracker",
  "tooltip":        "Some tooltip text",
  "items":          [<Item>, ...],
  "isTemplateIcon": false
}
```

| Field | JSON key | Go type | Description |
|-------|----------|---------|-------------|
| Icon | `"icon"` | string | Base64-encoded PNG; decoded and written to a temp file then loaded as tray icon |
| Title | `"title"` | string | Menu title (display use; exact behavior depends on OS) |
| Tooltip | `"tooltip"` | string | Tray tooltip shown on hover |
| Items | `"items"` | []Item | Top-level menu items; see Item format below |
| IsTemplateIcon | `"isTemplateIcon"` | bool | Passed to systray template icon API |

#### Action messages (`main_Action`)
Any number of these may follow the initial Menu, in any order.

```json
{
  "type":   "update-item",
  "item":   <Item>,
  "menu":   <Menu>,
  "seq_id": 42
}
```

| Field | JSON key | Go type | Description |
|-------|----------|---------|-------------|
| Type | `"type"` | string | One of the action types listed below |
| Item | `"item"` | Item | Used by `update-item` and `update-item-and-menu` |
| Menu | `"menu"` | Menu | Used by `update-menu` and `update-item-and-menu` |
| SeqID | `"seq_id"` | int64 | Sequence number (ordering hint) |

##### Action types

| Type string | Length | Effect |
|-------------|--------|--------|
| `"exit"` | 4 | Calls `systray.quit()` → `os.Exit(0)` |
| `"update-item"` | 11 | Updates a single menu item (see item update logic) |
| `"update-menu"` | 11 | Replaces the tray icon and tooltip from `menu.icon` / `menu.tooltip` |
| `"update-item-and-menu"` | 20 | Applies both `update-item` and `update-menu` in sequence |

The dispatcher (`main.onReady.func2.3`) switches on the **byte length** of the type string first, then compares content, so unknown type strings with the same length as a known type are silently discarded.

---

### Item format (`main_Item`)

Used inside `Menu.items`, `Action.item`, and `ClickEvent.item`.

```json
{
  "icon":           "<base64-PNG>",
  "title":          "Open Dashboard",
  "tooltip":        "Opens the SCTracker dashboard",
  "enabled":        true,
  "checked":        false,
  "hidden":         false,
  "items":          [],
  "__id":           7,
  "isTemplateIcon": false
}
```

| Field | JSON key | Go type | Notes |
|-------|----------|---------|-------|
| Icon | `"icon"` | string | Base64 PNG; empty string = no icon |
| Title | `"title"` | string | Displayed label; `"<SEPARATOR>"` triggers special handling (see below) |
| Tooltip | `"tooltip"` | string | Hover tooltip |
| Enabled | `"enabled"` | bool | `false` → item is disabled (grayed out) |
| Checked | `"checked"` | bool | `true` → item shows a checkmark |
| Hidden | `"hidden"` | bool | `true` → item is hidden from menu |
| Items | `"items"` | []Item | Sub-menu items; triggers `AddSubMenuItem` instead of `AddMenuItem` |
| InternalID | `"__id"` | int64 | Used by `update-item` to find the correct menu item |
| IsTemplateIcon | `"isTemplateIcon"` | bool | Passed to systray template icon API |

---

## Special Cases

### Separator items
If an item's `title` is exactly `"<SEPARATOR>"` (11 bytes), `addMenuItem` calls `systray.addSeparator()` instead of creating a menu item. The item is still appended to the internal item list for index tracking, but a `null` is stored at its menu item slot.

### Submenu items
If an item has a non-empty `items` array, it is created via `AddSubMenuItem(parent, title, tooltip)` rather than the top-level `AddMenuItem`. Submenus are processed recursively. Only one level of nesting is supported by the getlantern/systray library on Windows.

### Icon handling
Icons are base64-standard-encoded PNGs. The tray calls `encoding/base64.DecodeString` using the standard encoding (`qword_768EA8 = base64.StdEncoding`). Decode errors are printed to stderr and the icon is skipped (no icon set for that item). The tray icon itself goes through `iconBytesToFilePath` (writes to a temp file) then `winTray.loadIconFrom`.

---

## Update Item Logic (`main.onReady.func2.1`)

When an `update-item` action arrives, the tray:

1. **Item targeting** (mutually exclusive strategies):
   - If `seq_id >= 0`: use `seq_id` directly as the flat item list index (O(1)).
   - If `seq_id < 0`: look up by `item.__id` in the `map[int64]int` (also O(1)).
   - The click event echoes back both `seq_id` (flat index) and `__id`, so the parent can use either strategy for subsequent updates.
2. Updates the live `main_Item` snapshot in the flat item list.
3. On the systray `MenuItem` object:
   - If `hidden = true` → calls `hideMenuItem()`; stops here.
   - Sets `checked` and calls `menuItem.update()`.
   - Sets `disabled = !enabled` and calls `menuItem.update()`.
   - Updates `title` string and calls `menuItem.update()`.
   - Updates `tooltip` string and calls `menuItem.update()`.
   - If `icon` is non-empty: base64-decodes and calls `menuItem.SetIcon()`.
4. Calls `addOrUpdateMenuItem()` to flush the change to the Win32 tray.
5. Recursively updates sub-items by looking up each child's `InternalID` in the same map.

---

## Update Menu Logic (`main.onReady.func2.2`)

When an `update-menu` action arrives:

1. Compares `menu.tooltip` with the current tooltip; if different, updates the stored tooltip string.
2. Compares `menu.icon` with the current icon; if different:
   - Base64-decodes the new PNG.
   - Calls `systray.SetIcon(bytes)`.
3. Calls `systray.SetTooltip(tooltip)`.

---

## Shutdown Flow

### Parent-initiated (`"exit"` action)
```
Parent ──► {"type":"exit"} ──► action reader
                                    │
                            sync.Once → systray.quit()
                                    │
                              os.Exit(0)
```

### Signal-initiated (SIGTERM / SIGINT)
```
OS signal (SIGTERM=15 or SIGINT=2)
    │
    ├─ Prints "Quit" to stderr
    ├─ sync.Once → systray.quit()
    └─ main.onExit() → os.Exit(0)
```
The `sync.Once` at `0x79DE60` ensures `systray.quit()` is called at most once regardless of which path triggers shutdown.

### Stdin EOF / pipe break
```
stdin broken ──► main_readJSON returns error
                     │
              fmt.Fprintln(stderr, error)
              sync.Once → systray.quit() → os.Exit(0)
```

---

## Click Event Path (Win32 → stdout)

```
User right-clicks tray icon
    │
    ▼
wndProc receives wmSystrayMessage
    WM_RBUTTONUP (514) or WM_RBUTTONDBLCLK (517)
    │
    ▼
winTray.showMenu()  →  TrackPopupMenu (Win32)
    │
User clicks menu item
    │
    ▼
wndProc receives WM_COMMAND (273)
    lParam != -1  →  systrayMenuItemSelected(menuID)
    │
    ▼
systrayMenuItemSelected:
    map[uint32]*MenuItem lookup by Win32 menu ID
    runtime.selectnbsend(item.ClickedCh, struct{}{})  ← non-blocking!
    │
    ▼
reflect.Select in main.onReady.func2  receives on ClickedCh
    │
    ▼
Build main_ClickEvent{Type:"clicked", Item:snapshot, SeqID:idx, InternalID:__id}
    │
    ▼
encoding/json.Encoder.Encode → stdout
```

**Important**: The channel send is **non-blocking** (`selectnbsend`). If the click event loop is not ready to receive (e.g., currently encoding a previous event), the click is **silently dropped**. Rapid clicks may be lost.

### wndProc Windows messages handled

| Message | Value | Handler |
|---------|-------|---------|
| `WM_CREATE` | 1 | Calls the registered "ready" callback |
| `WM_DESTROY` | 2 | Removes tray icon from shell (Shell_NotifyIcon NIM_DELETE) |
| `WM_CLOSE` | 16 | DestroyWindow + unregister window class |
| `WM_ENDSESSION` | 22 | Removes tray icon; Windows session ending |
| `WM_COMMAND` | 273 | Menu item selected if lParam ≠ -1 |
| `wmSystrayMessage` | dynamic | Right-click/dblclick → showMenu |
| `wmTaskbarCreated` | dynamic | Re-registers tray icon after Explorer restart |
| (other) | — | DefWindowProc |

`wmSystrayMessage` and `wmTaskbarCreated` are registered at runtime via `RegisterWindowMessage`. The `wmTaskbarCreated` handler makes the tray icon automatically reappear when Windows Explorer crashes and restarts.

### systray.quit() implementation

```
systray.quit()
    PostMessageW(hwnd, WM_CLOSE, 0, 0)
        │
    wndProc(WM_CLOSE=16)
        DestroyWindow(hwnd)
        wndClassEx.unregister()
            │
    wndProc(WM_DESTROY=2)
        Shell_NotifyIcon(NIM_DELETE, nid)
        nativeLoop callback → os.Exit(0)
```

---

## Internal Data Structures

### Flat item index
The tray maintains two parallel slices:
- `[]*systray.MenuItem` — live Win32 menu item handles (null for separators)
- `[]*main_Item` — Go-side snapshots of each item's last-known state

And a map:
- `map[int64]int` — `InternalID → flat slice index`

These are built during initial menu setup and kept in sync on every update.

### Goroutine layout at runtime
| Goroutine | Function | Role |
|-----------|----------|------|
| G1 | `main.onReady.func1` | OS signal receiver loop |
| G2 | `main.onReady.func2` | Initial setup + click event `reflect.Select` loop |
| G3 | `main.onReady.func2.4` | Reads `Action` messages from stdin in a loop |

### Click event loop details (`reflect.Select` behavior)

The click loop in G2 works in two nested layers:

**Outer loop** (rebuilds select set):
- Iterates the `[]*MenuItem` slice and counts non-nil entries (visible items only).
- Allocates a new `[]reflect.SelectCase` of exactly that size.
- Calls `reflect.Select` on all active channels simultaneously.

**Inner loop** (drains one cycle):
- Each call to `reflect.Select` returns the index of whichever channel fired.
- If the fired channel carried a value (click) → encodes and emits the `ClickEvent`.
- If the channel was nil/closed (hidden item removed from set) → decrements the active count.
- Loops until active count reaches 0, then outer loop rebuilds.

**Consequence**: There is no inherent item limit from this design, but `update-item` with `hidden=true` permanently removes an item's channel from the select set — it can never be shown again in the current process. Adding new items at runtime is not supported (only updates).

### Win32 menu item ID map
The systray library maintains `map[uint32]*MenuItem` (`qword_768F08`) protected by `stru_79DF90` (a `sync.RWMutex`). The map key is the Win32 menu item ID, assigned sequentially starting from 1. This is separate from the `__id` used in the JSON protocol.

---

## Known Constraints and Edge Cases

| Behavior | Details |
|----------|---------|
| **Items are permanent** | Once created, items cannot be deleted. `hidden=true` hides them from display but they remain in the flat index and take up a slot. |
| **No dynamic addition** | New menu items cannot be added after the initial `Menu` message. The item list is fixed at startup. |
| **Click drops on congestion** | The Win32 → channel send is non-blocking (`selectnbsend`). Rapid double-clicks may lose events. |
| **Separator has no channel** | Separators store `null` in the `[]*MenuItem` slice. They never emit click events and are skipped in the `reflect.Select`. |
| **Icon decode errors silently skip** | A bad base64 icon logs to stderr and the item gets no icon; no error propagates to the parent. |
| **Tooltip comparison** | `update-menu` only calls `SetIcon`/`SetTooltip` if the values actually changed (string equality check), avoiding redundant Win32 calls. |
| **Taskbar restart resilience** | `wmTaskbarCreated` re-registers the icon with the shell after Explorer crashes, so the tray icon reappears automatically. |
| **Single level of submenus** | `getlantern/systray` on Windows only supports one level of nested submenus. Deeper nesting is silently flattened by the library. |
| **JSON is HTML-escaped** | The JSON encoder has `escapeHTML=true` (default), so `<`, `>`, `&` in item titles will be escaped in the click event output. |

---

## Win32 Implementation Details

### Window registration
- **Window class**: `"SystrayClass"` (registered via `RegisterClassExW`)
- **Custom tray message**: `RegisterWindowMessageW("wmSystrayMessage")` — the window message used for shell notification callbacks
- **Taskbar-restart message**: `RegisterWindowMessageW("TaskbarCreated")` — re-registers the tray icon if Explorer restarts

### Tray icon handling
- Icons are written to temp files with prefix `"systray_temp_icon_"` before being loaded via `LoadImageW`
- Icon PNG data is converted to HBITMAP via GDI (`CreateCompatibleDC`, `CreateCompatibleBitmap`)
- The library links: `User32.dll`, `Shell32.dll`, `Kernel32.dll`, `Gdi32.dll`

### Win32 APIs used
| DLL | Functions |
|-----|-----------|
| User32.dll | `CreateWindowExW`, `RegisterClassExW`, `UnregisterClassW`, `DefWindowProcW`, `DestroyWindow`, `ShowWindow`, `GetMessageW`, `TranslateMessage`, `DispatchMessageW`, `PostQuitMessage`, `PostMessageW`, `TrackPopupMenu`, `GetCursorPos`, `SetForegroundWindow`, `CreateMenu`, `CreatePopupMenu`, `DeleteMenu`, `InsertMenuItemW`, `SetMenuItemInfoW`, `SetMenuInfo`, `GetMenuItemID`, `GetSystemMetrics`, `LoadCursorW`, `LoadImageW`, `DrawIconEx`, `LoadIconW`, `ReleaseDC` |
| Shell32.dll | `Shell_NotifyIconW` |
| Kernel32.dll | `GetModuleHandleW`, `RegisterWindowMessageW` |
| Gdi32.dll | `CreateCompatibleDC`, `CreateCompatibleBitmap`, `SelectObject`, `DeleteDC` |

---

## Stderr Log Messages

All of these are written to `os.Stderr`, not stdout. They are diagnostic only and not part of the IPC protocol.

| Message | Trigger |
|---------|---------|
| `"Quit"` | SIGTERM (signal 15) received |
| `"Unhandled signal: <sig>"` | Any signal other than SIGTERM/SIGINT |
| `"Empty line"` | `readJSON` read an empty line from stdin |
| `"No menu item with ID %v"` | Win32 WM_COMMAND received for unknown menu item ID |
| `"Unable to set icon: %v"` | `Shell_NotifyIconW` call failed |
| `"Unable to set tooltip: %v"` | `Shell_NotifyIconW` call failed |
| `"Unable to addSeparator: %v"` | InsertMenuItemW failed for separator |
| `"Unable to hideMenuItem: %v"` | Win32 menu hide operation failed |
| `"Unable to addOrUpdateMenuItem: %v"` | InsertMenuItemW/SetMenuItemInfoW failed |
| `"Unable to create menu: %v"` | `CreatePopupMenu` failed at startup |
| `"Unable to init instance: %v"` | Window creation / class registration failed at startup |
| `"Unable to convert icon to bitmap: %v"` | GDI bitmap conversion failed |
| `"Unable to determine system directory"` | `GetSystemDirectoryA` failed |
| `"Unable to load icon from temp file: %v"` | `LoadImageW` failed on temp file |
| `"Unable to write icon data to temp file: %v"` | Writing temp icon file failed |
| `"Unable to log: %v\n"` | Logging subsystem error |
| `<JSON error from stdin>` | `readJSON` returned error (stdin broken/EOF) → triggers quit |

---

## Key Addresses (IDA)

| Symbol | Address | Notes |
|--------|---------|-------|
| `main.main` | `0x5B09C0` | Entry: calls `systray.Run(onReady, onExit)` |
| `main.onReady` | `0x5B13A0` | Sets up signals, spawns goroutines |
| `main.onReady.func2` | `0x5B2360` | Main message loop (0xE38 bytes) |
| `main.onReady.func2.1` | `0x5B16E0` | `update-item` handler |
| `main.onReady.func2.2` | `0x5B1C20` | `update-menu` handler |
| `main.onReady.func2.3` | `0x5B1EE0` | Action type dispatcher |
| `main.onReady.func2.4` | `0x5B2120` | Stdin action reader goroutine |
| `main.onExit` | `0x5B0A20` | `os.Exit(0)` |
| `main.readJSON` | `0x5B0A80` | Reads one newline-terminated JSON line |
| `main.addMenuItem` | `0x5B0C60` | Recursive menu item builder |
| `stru_79DE60` | `0x79DE60` | `sync.Once` for quit |
| `off_61EF90` | `0x61EF90` | → `systray.quit` (`0x5AE820`) |
| `sTypeReady` | `0x647320` | Go string: `{"type": "ready"}` |
| `qword_769CA8` | `0x769CA8` | `os.Stdout` |
| `qword_769C98` | `0x769C98` | `os.Stderr` |
| `qword_768EA8` | `0x768EA8` | `base64.StdEncoding` |
| `qword_768F08` | `0x768F08` | `map[uint32]*MenuItem` (Win32 ID → MenuItem) |
| `qword_76BB40` | `0x76BB40` | Global `winTray` instance |
| `qword_76BB58` | `0x76BB58` | Window HWND (used by `systray.quit`) |

---

## Protocol Wire Example

Complete session from startup to click and shutdown:

**stdout** (tray → parent):
```
{"type": "ready"}
{"type":"clicked","item":{"icon":"","title":"Open Dashboard","tooltip":"","enabled":true,"checked":false,"hidden":false,"items":null,"__id":3,"isTemplateIcon":false},"seq_id":2,"__id":3}
```

**stdin** (parent → tray, initial menu then updates):
```json
{"icon":"<base64>","title":"SCTracker","tooltip":"Star Citizen Tracker","items":[
  {"icon":"","title":"v2.1.4","tooltip":"","enabled":false,"checked":false,"hidden":false,"items":null,"__id":1,"isTemplateIcon":false},
  {"icon":"","title":"<SEPARATOR>","tooltip":"","enabled":true,"checked":false,"hidden":false,"items":null,"__id":2,"isTemplateIcon":false},
  {"icon":"","title":"Open Dashboard","tooltip":"","enabled":true,"checked":false,"hidden":false,"items":null,"__id":3,"isTemplateIcon":false},
  {"icon":"","title":"Exit","tooltip":"","enabled":true,"checked":false,"hidden":false,"items":null,"__id":4,"isTemplateIcon":false}
],"isTemplateIcon":false}
{"type":"update-item","item":{"icon":"","title":"v2.1.4 (updating...)","tooltip":"","enabled":false,"checked":false,"hidden":false,"items":null,"__id":1,"isTemplateIcon":false},"menu":{},"seq_id":-1}
{"type":"exit","item":{},"menu":{},"seq_id":0}
```
