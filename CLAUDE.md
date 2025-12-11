# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ED AFK Monitor is a Python application that monitors Elite Dangerous journal files in real-time for AFK massacre farming events. It outputs to terminal and optionally to Discord via webhooks with configurable notification levels.

## Running the Application

**Python version:**
```bash
py afk_monitor.py
```

**Common command-line arguments:**
- `-p <profile>` - Load a specific config profile
- `-f` or `--fileselect` - Show list of recent journals to choose from
- `-s <filename>` or `--setfile` - Load specific journal file
- `-j <path>` or `--journal` - Override journal folder path
- `-w <url>` or `--webhook` - Override Discord webhook URL
- `-r` or `--resetsession` - Reset session stats after preloading
- `-t` or `--test` - Re-route Discord messages to terminal (testing)
- `-d` or `--debug` - Print debug information

**Examples:**
```bash
# Load with auto-selected commander profile
py afk_monitor.py

# Load specific profile
py afk_monitor.py -p MyAltProfile

# Choose from recent journal files
py afk_monitor.py --fileselect

# Debug mode
py afk_monitor.py --debug
```

## Configuration

Configuration is in `afk_monitor.toml` (copy from `afk_monitor.example.toml` to start).

**Config structure:**
- `[Settings]` - General settings (journal folder, timezone, warnings, display options)
- `[Discord]` - Webhook URL, user ID, forum channel support, timestamps
- `[LogLevels]` - Per-event output levels (0=none, 1=terminal, 2=Discord, 3=Discord+ping)
- `[CustomProfileName]` - Optional profiles for different accounts/configs

**Profile system:**
Profiles auto-load by commander name or via `-p` argument. Profiles only need to specify overrides:
```toml
[MyAltProfile]
Discord.WebhookURL = "https://discord.com/api/webhooks/..."
LogLevels.ScanHard = 1
```

## Code Architecture

### Single-File Design
The entire application is in `afk_monitor.py` (~850 lines). This is intentional for ease of distribution.

### State Management
Two main classes track state:

**`Instance` class** - Resets per combat session:
- `scans` - Ships scanned since last kill
- `lastkill` - Timestamp of last kill
- `kills`, `bounties`, `merits` - Session counters
- `killsrecent` - Last N kill intervals for rate calculation

**`Tracking` class** - Persists across sessions:
- `deploytime` - When current session started
- `totalkills`, `totalbounties`, `totalmerits` - Overall totals
- `cmdrname`, `cmdrship`, `cmdrcombatrank` - Commander info
- `missionsactive` - List of active massacre mission IDs
- `preloading` - Whether initial journal read is in progress

### Event Processing Flow

1. **Startup** - Load config, find journal file, preload entire journal
2. **Preloading** - Read journal from start to build state (Discord disabled)
3. **Monitoring** - Tail journal file, process new lines as they appear
4. **Event matching** - `processevent()` uses match/case on `j["event"]`
5. **Logging** - `logevent()` handles terminal/Discord output based on log levels

### Key Event Handlers

Events in `processevent()` function (line 416+):
- `ShipTargeted` - Ship scans (easy vs hard ships)
- `Bounty`/`FactionKillBond` - Kills with stats calculation
- `MissionRedirected` - Mission stack progress
- `ShieldState` - Shield status changes
- `HullDamage` - Hull damage for ship/fighter
- `ReservoirReplenished` - Fuel level and consumption tracking
- `Died` - Ship destruction
- `LoadGame` - Commander/ship info
- `Music` (MainMenu) - Session end detection

### Session Tracking

Sessions start via `track.sessionstart()` on:
- First ship scan
- First kill
- Supercruise drop at combat zone
- Location event in planetary ring

Sessions end via `track.sessionend()` on:
- Main menu
- FSD jump
- Supercruise entry

### Discord Integration

Optional dependency: `discord-webhook` library

Webhook features:
- Custom identity (name/avatar)
- Forum channel support with auto-thread creation
- Duplicate message suppression (max 5 of same event)
- Per-event ping control via log levels

### Constants and Configuration

**Hard-coded constants** (lines 24-44):
- `VERSION` - Current version number
- `SHIPS_EASY`/`SHIPS_HARD` - Ship classifications for difficulty
- `LOGLEVEL_DEFAULTS` - Default log levels for all event types
- `WARN_NOKILLS`, `WARN_COOLDOWN` - Warning timing
- `KILLS_RECENT` - Number of recent kills for rate calculation (10)

### Utility Functions

- `time_format(seconds)` - Format seconds as "Xh Ym" or "Xm Ys"
- `num_format(number)` - Format large numbers as "1.2m" or "15k"
- `perhour(seconds, precision)` - Calculate per-hour rate
- `getconfig(category, setting, default)` - Get setting with profile override
- `getloglevel(key)` - Get log level with fallback to default
- `updatetitle()` - Windows console title with live stats

## Development Notes

### Version Updates
Update `VERSION` constant (line 27) using format YYMMDD (e.g., 251009 for 2025-10-09).

### Adding New Events
1. Add event case in `processevent()` match statement
2. Add log level default to `LOGLEVEL_DEFAULTS` (line 43)
3. Add log level to example config file
4. Document in changelog

### Error Handling
- Journal parsing errors are caught and logged but don't crash
- Discord webhook errors are caught and printed
- TOML decode errors now show helpful message and exit gracefully

### Platform Specifics
- Windows title updates use `ctypes.windll.kernel32.SetConsoleTitleW`
- Path handling uses `pathlib.Path` for cross-platform compatibility
- Default journal folder is Windows-specific but can be overridden

### Journal File Format
Elite Dangerous writes JSON-per-line logs:
- Each line is a separate JSON object with "event" and "timestamp" keys
- Journal files follow pattern: `Journal.YYYY-MM-DDTHHMMSS.NN.log`
- Monitor tails the file using `file.seek(0, 2)` then `readline()` loop

### Real-Time Monitoring Features
Every 60 seconds (line 792+) the monitor checks:
- Time since last kill (warn if > `WarnNoKills` minutes)
- Average kill rate (warn if < `WarnKillRate` per hour)
- Updates window title with live stats (Windows only)

### Testing
Use `--test` flag to route Discord messages to terminal without sending webhooks.
Use `--debug` flag to see detailed processing information.

## Common Modifications

### Adding Ship Types
Add ship internal names to `SHIPS_EASY` or `SHIPS_HARD` lists (line 40-41).

### Changing Warning Thresholds
Edit config file `WarnKillRate` and `WarnNoKills` or modify constants `WARN_NOKILLS`, `WARN_COOLDOWN`.

### Customizing Log Output
Modify the `logevent()` calls - first parameter is terminal output (uses `Col.*` color codes), second is Discord output (uses markdown).

### Profile Configuration
Profiles use dotted notation for overrides: `Category.Setting = value` (e.g., `Discord.WebhookURL`, `LogLevels.ScanHard`).
