# NyanDR (TsuserverDR)

## Project Overview

TsuserverDR is a Python-based multiplayer game server for **Danganronpa Online (DRO)**, a fan-made roleplaying community. It is a fork of [tsuserver3](https://github.com/AttorneyOnline/tsuserver3), originally written for Attorney Online (AO). TsuserverDR extends it with Danganronpa-specific features including trial minigames (Nonstop Debate), blood trail mechanics, zone systems, hub management, movement handicaps, and a rich set of OOC moderation and roleplay commands.

- **License:** GNU General Public License v3.0
- **Current version:** 5.4.2 (internal: `250628a`)
- **Original authors:** argoneus (tsuserver3, 2016), Chrezm/Iuvee (TsuserverDR, 2018–2022), Tricky Leifa (2022+)

## Tech Stack

| Component | Detail |
|-----------|--------|
| Language | Python 3.9–3.11 (3.9 minimum enforced at startup) |
| Async framework | `asyncio` (stdlib) with `asyncio.WindowsSelectorEventLoopPolicy` on Win32 |
| Configuration | YAML (via `PyYAML`) |
| WebSocket | `websockets` library |
| Master server listing | `aiohttp` (HTTP client) |
| IP discovery | `urllib.request` against `https://api.ipify.org` |
| Testing | `unittest` (stdlib) |

### Dependencies (`requirements.txt`)
```
PyYAML
aiohttp
websockets
```

## Repository Structure

```
NyanDR/
├── start_server.py              # Entry point
├── test.py                      # Top-level test runner
├── requirements.txt
├── CHANGELOG.md
├── CODINGPRACTICES.md           # Contribution and coding style guidelines
├── BACKCOMPATIBILITY.md         # Backwards compatibility policy
├── LICENSE.md
│
├── config_sample/               # Template — rename to config/ before running
│   ├── config.yaml              # Main server settings
│   ├── areas.yaml               # Area definitions
│   ├── backgrounds.yaml
│   ├── characters.yaml
│   ├── music.yaml
│   ├── gimp.yaml                # Gimp (word-filter replacement) list
│   ├── area_lists/              # Named alternate area list files
│   ├── bg_lists/
│   ├── char_lists/
│   └── music_lists/
│
├── config/                      # Active config (created by operator; not in repo)
├── logs/                        # Monthly logs: logs/server-YYYY-MM.log
├── storage/                     # Persistent user/ban data
│
├── server/                      # Core server package
│   ├── tsuserver.py             # TsuserverDR main class; wires all managers
│   ├── client_manager.py        # Client lifecycle, IC/OOC packet dispatch
│   ├── clients.py               # Client state model
│   ├── client_changearea.py     # Area-change logic
│   ├── commands.py              # All OOC command implementations
│   ├── commands_alt.py          # Overflow command implementations
│   ├── area_manager.py          # Area state, passage graph
│   ├── hub_manager.py           # Hub (collection of areas) management
│   ├── game_manager.py          # Base game management abstractions
│   ├── trial_manager.py         # Trial session management
│   ├── nonstopdebate.py         # Nonstop Debate minigame
│   ├── party_manager.py         # Party system
│   ├── ban_manager.py           # IP/HDID ban list
│   ├── timer_manager.py         # User-created server-side timers
│   ├── zone_manager.py          # Zone system (areas watched by staff)
│   ├── evidence.py              # Evidence list per area
│   ├── constants.py             # Shared constants and utilities
│   ├── exceptions.py            # Custom exception hierarchy
│   ├── logger.py                # Logging utilities
│   ├── subscriber.py            # Publisher/subscriber event system
│   ├── network/
│   │   ├── ao_protocol.py       # TCP AO2 protocol handler (asyncio Protocol)
│   │   ├── ao_protocol_ws.py    # WebSocket protocol adapter
│   │   ├── ao_commands.py       # Low-level AO packet command handlers
│   │   └── ms3_protocol.py      # Master server v3 client protocol
│   └── validate/                # YAML config validators
│       ├── areas.py, backgrounds.py, characters.py, config.py, gimp.py, music.py
│
└── tests/                       # Automated test suite
    ├── structures.py             # Base test classes
    ├── test_aaa_client_connection.py
    ├── test_authorization.py
    ├── test_blood.py / test_bloodclean.py / test_bloodsmear.py
    ├── test_bloodnotify_existsmear.py / test_bloodnotify_existtrail.py
    ├── test_blind.py / test_deafen.py / test_gag.py
    ├── test_characters.py / test_global.py / test_ic.py / test_ooc.py
    ├── test_lights.py / test_roll.py / test_scream.py
    ├── test_senseblock.py / test_whisper.py / test_poisoncure.py / test_info.py
    ├── test_zonebasic.py / test_zonechangeareas.py / test_zonechangewatchers.py
    ├── test_zoneeffect.py / test_zoneextranotifications.py
    └── test_zzz_nodebug.py
```

## Data Flow

### Startup Sequence
1. `start_server.py` runs `_pre_launch_checks()`: verifies `config/` folder, enforces Python >= 3.9.
2. `TsuserverDR.__init__()`: loads config, instantiates all managers (BanManager, TimerManager, PartyManager, ClientManager, HubManager), creates default `'Main'` hub, loads YAML configs.
3. `TsuserverDR.start()` (async):
   - Binds TCP socket with `AOProtocol` as factory.
   - If WebSocket enabled, starts `ao_protocol_ws` on `ws_port`.
   - Optionally connects to master server via `aiohttp`.
   - Enters `asyncio` event loop.

### Client Connection Lifecycle
1. `AOProtocol.connection_made()` — TCP established; `ClientManager.Client` created.
2. Server sends handshake: `decryptor` → `ID` → `PN` → `FL` → `ASS`.
3. Client requests character/background/music lists.
4. Client sends OOC commands or IC messages; `ao_commands.py` dispatches them.
5. `ClientManager` routes IC to area members, enforcing locks, sneak, gag, blind, deafen, etc.
6. `connection_lost()` — cleanup: remove from area, notify zone watchers, cancel async tasks.

### Command Dispatch
OOC messages starting with `/` are parsed in `ao_commands.py`, looked up in `commands`/`commands_alt` modules, permission-checked against the client's role, and executed. Errors raise `ServerError` subclasses returned to the client as OOC messages.

## Installation

```bash
# Python 3.9-3.11 required
python --version

# Install dependencies
python -m pip install --upgrade pip
python -m pip install --user -r requirements.txt

# Create config from sample
cp -r config_sample config   # Linux/macOS
# Windows: rename config_sample to config in Explorer

# Edit config/config.yaml and other YAML files
```

## Running the Server

```bash
python start_server.py
# Windows with both Python 2 and 3:
py -3 start_server.py
```

Shutdown: `Ctrl+C`. Restart requires stopping and re-launching.

## Configuration

All config lives in `config/` (created from `config_sample/`). YAML with 2-space indentation; no tabs.

### `config/config.yaml` — Key Settings

| Key | Default | Description |
|-----|---------|-------------|
| `port` | `50000` | TCP port |
| `ws_enabled` | `true` | Enable WebSocket |
| `ws_port` | `50001` | WebSocket port |
| `playerlimit` | `100` | Maximum concurrent clients |
| `timeout` | `250` | Client timeout in seconds |
| `local` | `false` | Localhost-only mode |
| `use_masterserver` | `true` | List on AO master server |
| `masterserver_name` | — | Server display name |
| `modpass` | — | Moderator password |
| `cmpass` | — | Community Manager password |
| `gmpass` | — | Universal Game Master password |
| `gmpass1`–`gmpass7` | — | Day-of-week GM passwords (Mon–Sun) |
| `motd` | — | Message of the day |
| `blackout_background` | `Blackout_HD` | Background shown when lights off / blinded |
| `showname_max_length` | `30` | Max custom showname length |
| `sneak_handicap` | `5` | Movement delay (seconds) when sneaking starts |
| `debug` | `false` | Enable debug logging |

### `config/areas.yaml`

Each area entry has mandatory keys `area` (name) and `background`, plus optional flags:
- `reachable_areas`, `visible_areas`: comma-separated names or `<ALL>` / `<REACHABLE_AREAS>`
- `iniswap_allowed`, `bullet`, `cbg_allowed`, `locking_allowed`, `lobby_area`, `private_area`, `has_lights`, `global_allowed`, `rollp_allowed`, `song_switch_allowed`
- `scream_range`: `<ALL>`, `<REACHABLE_AREAS>`, or comma-separated
- `restricted_chars`: list of barred character folder names
- `afk_delay`, `afk_sendto`: AFK kick configuration

## Role Hierarchy

1. **Player** — standard user
2. **Game Master (GM)** — event-running privileges (`/login gmpass`)
3. **Community Manager (CM)** — elevated moderation (`/login cmpass`)
4. **Moderator (Mod)** — full server control (`/login modpass`)

Daily GM passwords (`gmpass1`–`gmpass7`) expire at end of their respective day.

## Code Style (from CODINGPRACTICES.md)

- **Python version:** Must be compatible with Python 3.9.
- **Dependencies:** No non-standard library imports without approval. Only `PyYAML`, `aiohttp`, `websockets` are approved.
- **PEP 8** with project rules:
  - 4-space indentation, never tabs.
  - **Max line length: 100 characters** (exception: OOC command docstring examples).
- **Docstrings:**
  - Commands in `commands.py`: document rank restriction, purpose, functionality, syntax, argument meanings, and examples.
  - All other modules: NumPy docstring style.
- **YAML:** 2-space indentation with comments describing each key.

### Versioning Fields (`server/tsuserver.py`)

| Field | Purpose |
|-------|---------|
| `self.release` | Primary version (X in X.Y.Z) |
| `self.major_version` | Major version (Y) |
| `self.minor_version` | Minor version (Z) |
| `self.segment_version` | Pre/post-release tag: `''`, `'a1'`, `'b1'`, `'RC1'`, `'post1'` |
| `self.internal_version` | Date-based build stamp: `[stage][O?]yymmdd[letter]` |

Must be updated in every pull request.

## Testing

```bash
# From the repository root
python test.py
```

`test.py` discovers all `test_*.py` files in `tests/` using `unittest`. Tests run in alphabetical order: `test_aaa_*` first (connection), `test_zzz_*` last (no-debug checks).

`tests/structures.py` provides `_Unittest` and `_TestTsuserverDR` (in-process server + synthetic clients; no real TCP socket needed).

### Test Coverage

| Test file | Feature |
|-----------|---------|
| `test_aaa_client_connection` | Connection handshake |
| `test_authorization` | Role-based login and permissions |
| `test_blind` / `test_deafen` / `test_gag` | Sense/speech status effects |
| `test_blood*` | Blood trail mechanics and notifications |
| `test_characters` | Character selection and restrictions |
| `test_ic` / `test_ooc` / `test_global` | Message handling |
| `test_lights` | Area light toggle |
| `test_roll` | Dice roll commands |
| `test_scream` / `test_whisper` | Cross-area/targeted messages |
| `test_senseblock` / `test_poisoncure` | Special mechanics |
| `test_zone*` (5 files) | Zone system |
| `test_zzz_nodebug` | Server behavior with debug off |

## Backwards Compatibility

### Semantic Versioning Policy

| Version change | Backwards compatibility |
|---------------|------------------------|
| `W` increases (post-release) | No backwards-incompatible changes |
| `Z` increases (minor) | Incompatible changes may be announced with warnings |
| `Y` increases (major) | Incompatible changes may be introduced (if previously announced) |
| `X` increases (primary) | Backwards-incompatible changes are definitely introduced |

### Audience Tags in BACKCOMPATIBILITY.md
- **[O]** — affects server operators
- **[P]** — affects players
- Untagged — affects developers only

### Notable Changes (v4.2.1)
- `logs/server.log` replaced by monthly `logs/server-YYYY-MM.log`.
- Several legacy command names deprecated (still functional with warnings).
- `cccc_ic_support` included in `FL` packet.

## Key Architectural Notes

- **Single-process, single-threaded async:** All I/O is non-blocking via one asyncio event loop.
- **Manager pattern:** Every domain (clients, areas, hubs, bans, timers, parties, zones, games) has a dedicated `*Manager` class held by `TsuserverDR`.
- **Hub/Area hierarchy:** Hubs contain areas. Default hub is always `'Main'`.
- **Subscriber pattern:** `server/subscriber.py` provides pub/sub for cross-manager notifications.
- **Validation layer:** `server/validate/` runs at startup to give clear YAML error messages.
- **Protocol abstraction:** `ao_protocol.py` (TCP) and `ao_protocol_ws.py` (WebSocket) both feed into the same `ao_commands.py` dispatch layer.
- **Error handling:** `ServerError` subclasses are caught and returned to the client as OOC messages. Unhandled asyncio exceptions route through `error_queue` for graceful shutdown.
