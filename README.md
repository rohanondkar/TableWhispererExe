# Tablewhisper

Tablewhisper is a dungeon-master-only D&D 5e battle-map console. The DM asks for a check, the table shows the printed numbers, and the DM submits the d20. The app does not roll that d20 and does not decide hit or miss on its own.

This repository is the packaged Windows build. `Tablewhisper.exe` is the program Steam launches. The development source (the Vite app, the FastAPI project, and `scripts/build-steam.ps1`) lives in a separate project and is copied into this folder when the exe is rebuilt.

## Two editions

Both editions talk to an API on `127.0.0.1:8766`. They do not share a save file.

| | Browser edition | This build |
| --- | --- | --- |
| How it starts | `start-dev.bat` in the source project, UI on port 5173 | `Tablewhisper.exe` |
| Window title | DM Console | Tablewhisper |
| Title screen | Not shown | Shown, because the window is packaged |
| Saves | `data\` in the source project | `data\` beside this exe |
| API process | The source virtualenv, with reload | Bundled `resources\python\python.exe`, no reload |

If something is already answering `/health` on port 8766, this exe shows a dialog and quits. It does not kill that other API. Quitting this exe stops only the Python process it started. It does not run the development quit script.

## How a launch fits together

```
Tablewhisper.exe  (Electron)
  ├─ reads data\display.json and opens the window
  ├─ starts resources\python\python.exe
  │    uvicorn app.main:app  --host 127.0.0.1  --port 8766
  │    cwd: resources\api
  └─ loads the packaged page (dist\index.html inside the app)
       the page calls http://127.0.0.1:8766
```

The packaged Python process gets these environment variables:

- `TABLEWHISPER_DATA` — `data` beside the exe
- `TABLEWHISPER_PACKAGES` — `resources\packages`
- `PYTHONNOUSERSITE=1` — ignore any Python packages installed for the Windows user
- `DM_API_PORT` — `8766`

`resources\python\python312._pth` lists `python312.zip`, `.` (the folder that contains `python.exe`, not the current working directory), `Lib\site-packages`, and `import site`. Uvicorn still imports `app.main` because the working directory is `resources\api`.

Ollama and the Discord listener are not inside the exe. Ollama, when used, is expected at `http://127.0.0.1:11434`. Discord audio is posted into the API by a separate bot.

## What is in this folder

```
Tablewhisper.exe          Electron shell. Stored with Git LFS (about 180 MB).
locales\                  Electron / Chromium language packs
*.dll, *.pak, *.bin       Chromium runtime that Electron ships
resources\
  api\                    FastAPI application (the app package)
  python\                 Embeddable Python 3.12.8 plus the API's installed libraries
  packages\               Rules, monster cards, NPC cards, and token art
data\                     Saves for this install. Created beside the exe.
  dm_console.sqlite3      The table
  display.json            Last display choice
  uploads\                Imported character PDFs
  character_images\       Party portraits
  monster_images\         Custom foe art
  npc_images\
  map_images\             Battle-map backgrounds
  map_fog\                Fog masks
  gear_images\
  logs\                   api.log, written while the exe is running. Not in git.
```

Rebuilding from the source project moves `data` aside, replaces the rest of this folder, and puts `data` back. A rebuild does not wipe the table.

## Desktop

The page is a React app built with Vite (`base` is `./`, so it loads from the packaged file as well as from a dev server). Electron's preload exposes a small bridge: API base URL, whether this window is the packaged game, quit, and get/set display.

The console is three columns: party, the ask-and-answer middle, and foes. The log is a book on its own tab. The map is a Konva stage with trays for creatures and the selected token.

Column widths follow the window:

- Under 1400px wide, the party and foe columns start narrower, and map trays start narrower when the DM has not dragged them.
- From 1500px, the console uses the 300px / 340px columns.
- From 1800px, those columns grow a little. The log column stays capped so a line does not become a full-width paragraph.
- The layout does not collapse into one column at 1280×720. It stacks only below 1100px.

### Title screen

Shown only in this packaged build.

- **Continue** opens the table that is already saved.
- **New table** starts a session.
- **Options** opens display, audio, and graphics. Back does not stop the API.
- **Credits** names the title music.
- **Quit** closes this exe and the Python process it started.

Title music is *Planning* by Alexander Nakarada ([creatorchords.com](https://creatorchords.com)), licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It loops on the title screen only, including while Options is open from that screen. It stops when the table opens. **Options → Audio → Title music** is its volume, stored in the window as `tablewhisper-music-volume`.

### Options

**Display.** On launch, and whenever Options opens, Electron reads `screen.getAllDisplays()` (bounds, work area, scale, and which display is primary). Each monitor is its own block. A windowed preset is offered only when it fits that monitor's work area. The presets are 1280×720, 1600×900, 1920×1080, and 2560×1440. A 1080p panel does not offer 2560×1440.

- **Windowed — Free** lets the frame be dragged. The size is written back to `data\display.json`.
- **Windowed — Preset** locks the frame to that size. Fullscreen stays locked until Free is chosen.
- **Fullscreen** moves the window onto that display, then uses Electron's borderless fullscreen. It is not an exclusive mode.

The first launch, when `display.json` is missing, picks the primary monitor and the largest preset that fits. The minimum frame is 980×680.

**Audio.** Whisper model (`tiny`, `base`, `small`, `medium`; small is the default), listen length (10–180 seconds, default 45), and start or stop listening. A line shows whether the source is idle, system audio, or Discord. Those values go through `GET` and `PATCH /settings`. Title-music volume is local to the window.

**Graphics.** Spell effects are **Full**, **Reduced**, or **Off**, stored as `tablewhisper-spell-fx`. Full plays the cast. Reduced is the still flash also used for `prefers-reduced-motion` when no choice is saved. Off draws nothing. Night ink for the log stays on the log tab.

## API

FastAPI, SQLite, one process. The modules under `resources\api\app` split the work:

| Module | Role |
| --- | --- |
| `main.py` | Routes, static media mounts, and the process lifespan |
| `config.py` | Paths and defaults. Honors `TABLEWHISPER_DATA`, `TABLEWHISPER_PACKAGES`, and `TABLEWHISPER_ROOT` |
| `db.py` | SQLite connection, settings, characters, sessions, events |
| `pdf_import.py` | Read a character PDF into a sheet. Preview, then upload |
| `query_engine.py` | Turn the DM's question into a card: who rolls, what kind of check, and which printed numbers apply |
| `roll_factors.py` | Factors that change a roll, taken from the sheet and the card |
| `spell_cast.py` | Spell lookup used by the map |
| `monsters.py` | SRD and custom foes, portraits, encounter rows |
| `npcs.py` | SRD and custom NPCs, and who is in the scene |
| `maps.py` | Maps, tokens, walls, lights, fog, portals |
| `token_art.py` | Which portrait file a creature uses |
| `creature_size.py` | Size name to footprint in squares |
| `equipment.py`, `gear_rules.py`, `gear_images.py` | Gear on the sheet and the pictures for it |
| `xp.py` | Award XP |
| `voice.py` | Microphone / system-audio capture, Discord PCM ingest, Whisper transcription |
| `rules.py` | The active ruleset package |
| `ollama_client.py` | Optional local model at port 11434 |
| `session_save.py` | Save and load a table into the data folder |
| `portraits.py` | Character portrait files |

Media directories are created before the static mounts. The process used to crash on a fresh `data` folder because those mounts ran before the lifespan created the folders.

### SQLite

`data\dm_console.sqlite3` holds the table.

- `settings` — active ruleset, Ollama model, Whisper model, listen length, hotkey
- `characters` — party sheets as JSON, plus the source PDF name
- `sessions`, `events` — named tables and the log. An event records the question, the card, and whether it came from the console or the map
- `custom_monsters`, `encounter_enemies` — foe templates and the foes on this table (AC, HP, and the rest of the card)
- `custom_npcs`, `scene_npcs` — NPC templates and who is in the scene
- `maps`, `map_tokens`, `map_walls`, `map_lights`, `map_fog`, `map_portals` — the battle map

HP for a character lives inside that character's JSON. HP for a foe lives on the encounter row.

### Routes

Everything is on port 8766. Groups, not every path:

- `/health`, `/status`, `/settings`, `/shutdown`
- `/characters`, PDF preview and upload, portraits, gear images, `/xp/award`
- `/query` — the DM's question becomes a card
- `/sessions`, the active session's events, notes, export, and restore
- `/saves` — files in the data folder
- `/monsters`, `/encounter`
- `/npcs`, `/scene`
- `/maps` and nested tokens, walls, lights, fog, and portals. A portal can move a token onto another map
- `/voice/start`, `/voice/stop`, `/voice/capture`, `/voice/discord/ingest`
- `/rulesets`

## Asking for a check

`POST /query` parses the question and returns a card. The desktop shows that card and waits.

- A weapon attack is the attacker's d20 against AC. A natural 20 hits. A natural 1 misses. Otherwise the face is compared with the printed number needed.
- A spell that asks for a save is the defender's d20. The app does not invent a DC. If the card has no printed DC, the DM types the damage.
- A breath weapon stored with an attack bonus of 0 is not an attack against AC. The card names the creature and the breath, lists the printed damage formula, and leaves the DC empty.
- After the DM submits the d20, the client rolls the printed damage formula. A successful save halves that damage, rounded down, only when the ruling text says half. Otherwise HP does not change for the save itself.
- The log line names the creature who is acting. A breath or an attack uses that creature, not the player character who was merely the target.

## Map

The map is drawn with Konva. Tokens come from the encounter and the scene, and their portraits come from `packages\token-portraits`. A creature whose encounter row was removed still gets its portrait and size back the next time the map loads, matched from the token's label to the SRD card.

Placing a spell starts its travel effect immediately. A weapon's effect plays when the check is committed. Spell looks are chosen from the spell name: fire, lightning, cold, acid, poison, thunder, radiant, necrotic, heal, force, or a plain arcane shimmer. Fireball is fire.

Walls block movement and sight. Lights and a fog mask sit on top of the grid. The DM can calibrate the grid, measure, and run a player-preview of what a token can see.

## Packages

`resources\packages` is the rules content, copied from the source `packages` folder:

- `dnd5e-srd` — the active ruleset by default
- `monsters-srd` — foe cards and the art those cards ship with
- `token-portraits` — the round token used on the map, one file per monster id
- `npcs-srd` — sample NPCs

A token URL looks like `/media/tokens/monsters/{id}.png`. The file check keeps the `monsters/` folder. Checking only the file name used to send every creature to the same humanoid token.

## Voice

Whisper runs inside the bundled Python (`faster-whisper`). The default model is `small`. Listening uses the system audio device, or PCM posted to `/voice/discord/ingest` by the separate Discord bot. Status reports the source as idle, system audio, or Discord.

## Build

From the source project, `scripts/build-steam.ps1` does the following:

1. `npm run build` in `apps\desktop` (`tsc`, then Vite).
2. Uses a cached embeddable Python 3.12.8 under `G:\Discord Bot\Tablewhisper-build-cache` and installs `apps\api\requirements.txt` into it.
3. Copies the API source, skipping `.venv` and bytecode, and copies `packages`.
4. Runs electron-builder for Windows (`dir` output, asar, no code signing).
5. Replaces this folder with that output, after setting `data` aside and moving it back.

`Tablewhisper.exe` in git is a Git LFS pointer. Clone with Git LFS or the exe will not run.

Runtime logs stay out of git (`data/logs/` and `*.log` in `.gitignore`).
