# glassbar — Design Spec

**Date:** 2026-04-25
**Status:** Approved for planning
**Owner:** Pikammmmm

## Summary

`glassbar` is a Windows desktop app that replaces the visual role of the Windows taskbar with two glassy, always-on-top floating panels:

1. A **dock** at bottom-center for launching and managing running apps.
2. A **HUD** in the top-right corner showing clock, network throughput, and now-playing media.

It coexists with the OS taskbar (which the user keeps auto-hidden) rather than replacing `explorer.exe`. Built in Rust + Tauri.

## Goals

- Clean, modern aesthetic — acrylic/Mica glass blur, transparent, undecorated.
- Fast — startup under 1s, idle CPU under 1%, RAM under ~80 MB.
- Functional taskbar replacement: launch pinned apps, switch between running windows, see what's playing.
- Easy to extend with new HUD widgets later.

## Non-Goals (v1)

- Replacing `explorer.exe` as the Windows shell.
- Hosting third-party system tray icons (deferred — see Risks).
- Multi-monitor support beyond the primary display.
- In-app settings UI (edit JSON config for now).
- Theming. One look only.
- Touch / pen input.

## Architecture

Single Tauri process with two independent windows. Rust backend owns all OS interaction; HTML/CSS/JS frontends are thin and reactive.

```
┌─────────────────────────────────────────────────────────┐
│                     Tauri Process                        │
│                                                           │
│  ┌─────────────────┐         ┌──────────────────────┐   │
│  │  Dock Window    │         │   HUD Window         │   │
│  │  (HTML/CSS/JS)  │         │   (HTML/CSS/JS)      │   │
│  └────────┬────────┘         └──────────┬───────────┘   │
│           │ Tauri events / commands     │                │
│           └─────────────┬────────────────┘                │
│                         │                                 │
│  ┌──────────────────────▼────────────────────────────┐  │
│  │             Rust Backend                          │  │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────────┐ │  │
│  │  │AppTracker  │ │ Pinned   │ │ WidgetState    │ │  │
│  │  │(EnumWindows│ │ (JSON)   │ │ (clock/net/    │ │  │
│  │  │ poller)    │ │          │ │  media)        │ │  │
│  │  └────────────┘ └──────────┘ └────────────────┘ │  │
│  │              ┌──────────────────┐                │  │
│  │              │ windows_api.rs   │                │  │
│  │              │ (Win32 wrappers) │                │  │
│  │              └──────────────────┘                │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Components

### Window setup (`main.rs`)

Two windows configured in `tauri.conf.json`:

| Window | Position | Size | Always-on-top | Decorations | Transparent |
|--------|----------|------|---------------|-------------|-------------|
| `dock` | bottom-center, 12px above edge | 700×64 (auto-sizes to icon count) | yes | none | yes |
| `hud`  | top-right, 12px inset             | 280×220 (auto-sizes to widgets)   | yes | none | yes |

Both windows apply blur via the `window-vibrancy` crate at startup:
- Try **Mica** (Windows 11). On failure, fall back to **Acrylic** (Windows 10 1809+).

Empty regions are click-through via `set_ignore_cursor_events(true)` on transparent areas (handled per-element with CSS `pointer-events`).

### `AppTracker` (`app_tracker.rs`)

Polls `EnumWindows` every 500 ms. For each top-level visible HWND:
1. Resolve owning PID via `GetWindowThreadProcessId`.
2. Resolve full executable path via `QueryFullProcessImageNameW`.
3. Group HWNDs by exe path → `RunningApp { exe_path, windows: Vec<HwndInfo> }`.
4. Diff against the previous snapshot; only emit a Tauri event `apps:changed` if something changed (set of exe paths, or window count per exe).

Exclusions: invisible windows, tool windows, our own windows, the shell (`explorer.exe` desktop window).

### `Pinned` (`pinned.rs`)

`%APPDATA%\glassbar\pinned.json`:

```json
[
  { "path": "C:\\Program Files\\Mozilla Firefox\\firefox.exe", "display_name": "Firefox" },
  { "path": "C:\\Windows\\System32\\notepad.exe", "display_name": "Notepad", "icon_path": "C:\\path\\to\\custom.ico" }
]
```

`load()` reads + parses; `save()` serializes back. Watched via `notify` crate so manual edits hot-reload without restart.

### `Icons` (`icons.rs`)

For dock display, we need an icon per app. Resolution order:
1. Pinned entry has `icon_path` → load from disk.
2. Otherwise extract from exe via `SHGetFileInfoW` with `SHGFI_ICON | SHGFI_LARGEICON`, convert HICON to PNG bytes.
3. Cache results in-memory keyed by exe path; persist to `%APPDATA%\glassbar\icon-cache\<hash>.png` so subsequent starts skip re-extraction.

### Widgets (`widgets/`)

Each widget exposes `current() -> Value` and `subscribe() -> impl Stream<Value>`. The `WidgetState` aggregator subscribes to all of them and emits a unified `hud:update` event when any value changes (debounced at 200 ms minimum gap to avoid flooding the webview).

| Widget | Source | Refresh |
|--------|--------|---------|
| Clock  | `chrono::Local::now()` | every 1s, fires only when minute changes (seconds rendered client-side via JS) |
| Network | `sysinfo::Networks` totals across interfaces, smoothed over 3-sample sliding window | every 1s |
| Media   | `windows::Media::Control::GlobalSystemMediaTransportControlsSessionManager` | session changed events |

### Tauri Commands (`commands.rs`)

Frontend → backend RPCs:

| Command | Purpose |
|---------|---------|
| `launch(exe_path)` | Spawn a process via `Command::new` |
| `focus_window(hwnd)` | `SetForegroundWindow` + restore if minimized |
| `minimize_window(hwnd)` | `ShowWindow(SW_MINIMIZE)` |
| `close_window(hwnd)` | Post `WM_CLOSE` to the window |
| `pin_app(path)` / `unpin_app(path)` | Mutate `pinned.json` |
| `set_hud_position(x, y)` | Persist new HUD coords (after drag) |

## Data Flow

**Dock click loop:**
```
user clicks icon
  → JS sends Tauri command (launch | focus | minimize)
  → Rust performs Win32 call
  → AppTracker poll picks up state change ≤500ms later
  → emits apps:changed → JS re-renders running indicators
```

**HUD update loop:**
```
WidgetState aggregator (clock + network sampler + media listener)
  → debounced merge
  → emits hud:update with full snapshot
  → JS replaces widget DOM
```

## Project Layout

```
glassbar/
├── src-tauri/
│   ├── src/
│   │   ├── main.rs              // app entry, window setup, blur init
│   │   ├── windows_api.rs       // thin Win32 wrappers
│   │   ├── app_tracker.rs       // running-app polling + diff
│   │   ├── pinned.rs            // load/save pinned.json + file watcher
│   │   ├── icons.rs             // exe → icon extraction + cache
│   │   ├── widgets/
│   │   │   ├── mod.rs
│   │   │   ├── clock.rs
│   │   │   ├── network.rs
│   │   │   └── media.rs
│   │   ├── widget_state.rs      // aggregator + event emitter
│   │   └── commands.rs          // Tauri command handlers
│   ├── tauri.conf.json
│   ├── build.rs
│   └── Cargo.toml
├── ui/
│   ├── dock/
│   │   ├── index.html
│   │   ├── style.css
│   │   └── app.js
│   ├── hud/
│   │   ├── index.html
│   │   ├── style.css
│   │   └── app.js
│   └── shared/
│       └── glass.css
├── pinned.example.json
├── docs/superpowers/specs/2026-04-25-glassbar-design.md
├── .gitignore
└── README.md
```

## Dependencies (Cargo)

- `tauri` 2.x
- `windows` (with features: `Win32_UI_WindowsAndMessaging`, `Win32_System_ProcessThreads`, `Win32_UI_Shell`, `Media_Control`)
- `window-vibrancy`
- `sysinfo`
- `chrono`
- `serde`, `serde_json`
- `notify` (config hot-reload)
- `tokio` (async runtime, already pulled by Tauri)
- `anyhow`, `thiserror` (errors)
- `tracing`, `tracing-subscriber` (logging)

## Performance Targets

| Metric | Target |
|--------|--------|
| Cold start to visible windows | < 1s |
| Idle CPU | < 1% on a modern machine |
| Idle RAM | < 80 MB total for the process |
| Dock click → action | < 100 ms perceived |
| HUD update tick | ≤ 1 / sec, ≤ 200ms debounce on bursts |

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **System tray hosting is undocumented and brittle** | Deferred from v1. Hidden behind a feature flag. v1 ships without it. |
| **Acrylic blur jank on low-end GPUs** | Detect and fall back to a flat semi-transparent dark background if blur underperforms. |
| **`EnumWindows` polling cost on busy systems** | Diff before emitting; tune interval (500ms default, configurable). |
| **Icon extraction occasionally returns nothing** for some exes (UWP, etc.) | Fallback to a generic monochrome icon; log the path so user can set `icon_path` manually. |
| **Multi-monitor edge cases** | Out of scope for v1. Document as known limitation. Position relative to primary monitor only. |
| **Auto-start at login** | Use registry `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` entry; opt-in checkbox in JSON config (`auto_start: true`), not enabled by default. |

## Testing Strategy

**Unit tests (Rust):**
- `pinned::load/save` round-trip with various JSON shapes (empty, malformed, missing fields).
- `app_tracker::diff` correctness — additions, removals, count changes.
- `widgets::network` smoothing math given known sample sequences.

**Manual testing checklist (per release):**
- Cold start on Win11 — both windows appear, blur applied.
- Click pinned non-running app → launches.
- Click running app → focuses; click again → minimizes.
- Right-click pinned app → menu shows correct windows + close-all works.
- Edit `pinned.json` while running → dock updates within 1 second.
- Play music in Spotify → HUD shows track + artist; pause → HUD updates.
- Network transfer → up/down numbers move sensibly.
- Drag HUD to new corner → position persists across restart.
- Run for 8h → memory stays under target, no leaks.

E2E via webdriver is **not** in scope for v1 — overhead exceeds value at this size.

## Out of Scope / Future Work

- System tray icon hosting (Shell_TrayWnd interception).
- Multi-monitor positioning + per-monitor docks.
- In-app settings UI (replace JSON editing).
- Theme system / color customization.
- Additional widgets: CPU/RAM, weather, calendar, GPU.
- Keyboard shortcut to summon dock when fully hidden.
- Workspace/virtual-desktop awareness in dock.

## Open Questions

None at design time. Implementation may surface platform-specific issues (Win11 build variation in Mica behavior) — those will be handled in the implementation plan's discovery phase.
