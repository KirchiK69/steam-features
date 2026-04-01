# Steam Engineer — ~~Spy~~ Problem around here

![Spy around here!](assets/engineer.gif)

## Problem

Users receive little to no meaningful feedback when games fail to launch via Proton (Steam Deck / Linux). A typical scenario:

- A game fails to launch without any visible error.
- Errors are scattered across multiple log files in different directories.
- Logs are located in `compatdata`, system paths, and inside Wine prefixes (e.g., `drive_c/users/steamuser/AppData`).
- There is no single point of aggregation or analysis.
- The root cause is often hidden behind a chain of secondary errors (e.g., DXVK fails → Proton emits a generic error → user only sees the final symptom).

In its current state, troubleshooting requires deep knowledge of Proton/Wine internals and manual inspection of dozens of files.

---

## Current State

- Steam typically shows only a generic error (e.g., "Application error") or nothing at all.
- Logs are written to multiple locations, often inaccessible from the UI.
- The community relies on forums, ProtonDB, and manual log analysis.

---

## Proposed Solution

Develop a standalone CLI tool that:

1. **Automatically discovers all relevant logs** — even in non-standard locations.
2. **Aggregates them into a unified stream** — including files and stdout/stderr.
3. **Normalizes data** into a machine-readable format (NDJSON).
4. **Analyzes the stream** — identifies root causes and filters cascade errors.
5. **Suggests solutions** — based on local rules and ProtonDB data.

The tool runs alongside the game, requires no terminal interaction, and preserves logs even if the process crashes. In the future, it can be integrated into the Steam UI (e.g., a “Logs” button).

---

## Algorithms and Components

### 1. Log Autodiscovery

Goal: detect all log files dynamically, without prior knowledge of their locations.

#### 1.1 Static Paths

Scan known Steam and Proton directories:

- `~/.steam/steam/logs/`
- `~/.steam/steam/steamapps/compatdata/<appid>/`
- `~/.local/share/vulkan/`
- `~/.cache/`
- Wine prefix paths:
  - `drive_c/users/steamuser/AppData/`
  - `drive_c/Program Files/`

---

#### 1.2 Process Inspection

- Identify the game process via appid or process name.
- Recursively build the process tree (game → Proton → Wine → children).
- Inspect `/proc/<pid>/fd/` to discover open file descriptors:
  - stdout/stderr
  - dynamically created log files

---

#### 1.3 Heuristic Scoring

Each candidate file is assigned a **score** to determine whether it is a log.

**Scoring criteria:**

- File extension:
  - `.log` → +10  
  - `.txt` → +5  
  - `.err` → +8  
  - binary/media files → -50  

- Path keywords:
  - `log`, `wine`, `proton`, `dxvk` → +5 each  

- File behavior:
  - Growing size → +15  
  - Size change over time → +20  

- Content patterns:
  - Timestamps (ISO-8601, `[HH:MM:SS]`, etc.) → +5 each  
  - Keywords (`error`, `warn`, `failed`, `dxvk`, `wine`, `proton`) → +5 to +25  

- Structure:
  - Consistent line length → +10  

- Known log directories → +30  

**Threshold:** `score >= 30`

---

#### 1.4 Real-Time Monitoring

- Use `inotify` to watch directories.
- Newly created files are evaluated in real time.
- Files exceeding the threshold are added to the collection.

---

### 2. Collector

- Asynchronously reads all discovered logs:
  - file tailing (`tail -F` behavior)
  - stdout/stderr via `/proc/<pid>/fd/1` and `/proc/<pid>/fd/2`
- Streams all lines into a processing queue.

---

### 3. Parser

Selects parsing strategy based on:

- filename (`dxvk`, `wine`, `proton`)
- file header
- data format (JSON, XML, plain text)

Extracts:

- timestamp
- log level (`error`, `warn`, `info`)
- component (`dxvk`, `wine`, `proton`, etc.)
- raw message

Unparsed lines are marked as `raw`.

---

### 4. Normalizer

Transforms all events into a unified **NDJSON** format:

```json
{
  "ts": "2026-04-01T12:00:00+02:00",
  "layer": 4,
  "type": "error",
  "code": "102.4.2.15",
  "message": "DXVK: failed to create device",
  "source": "/home/user/.steam/steam/logs/dxvk.log",
  "raw": "[Error] DXVK: failed to create device"
}
```

---

#### Layer Classification

|Layer|Name|Example Issue|
|-|-|-|
|1|Hardware|Insufficient VRAM|
|2|Kernel|Missing Vulkan driver|
|3|Wine|Wine misconfiguration|
|4|DXVK / VKD3D|Failed to create device|
|5|Vulkan/OpenGL|API initialization failure|
|6|Proton|Incompatible Proton version|
|7|SteamOS/Client|Steam cannot launch game|
|8|Game Engine|Engine crash|
|9|Game|Game-specific issue|

#### Error Code Format

```group.layer.subsystem.error```

Example:

```102.4.2.15```

- 102 — launch failure
- 4 — DXVK layer
- 2 — device-related issue
- 15 — specific error

### 5. Analyzer

- Processes normalized events.
- Detects cascade errors and identifies the root cause.
- Matches patterns using rules engine:
  - keywords
  - regex
  - known signatures

Produces diagnostic events:

```json
{
  "root_cause": "DXVK device creation failed",
  "confidence": 0.87
}
```

---

### 6. Resolver

- Maps error codes to solutions.
- Uses:
  - local rules database
  - ProtonDB API (future)

Example output:

Try:

- Install Vulkan drivers
- Use Proton 8.0+

Unknown issues are stored for later analysis.

---

### 7. Local Rules Database

Stored in JSON format:

```json
{
  "error_code": "102.4.2.15",
  "layer": 4,
  "component": "dxvk",
  "pattern": "failed to create device",
  "fix": "Install Vulkan drivers (e.g., mesa-vulkan-drivers) or switch to Proton 8.0+",
  "community_verified": true
}
```

---

### 8. Game Launch Integration

- Enabled via launch option (e.g., checkbox in UI)
- Runs in background
- Preserves logs even after crash

---

## Expected Impact

- Reduced support load — users receive actionable diagnostics
- Improved UX — transparent and understandable error reporting
- Faster issue detection — structured log aggregation
- Future automation — automatic fixes (Proton switching, launch flags)
