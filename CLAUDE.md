# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **mod-playerbots fork** of AzerothCore (`https://github.com/mod-playerbots/azerothcore-wotlk`), branch `Playerbot`. The Playerbot AI system is built directly into the core (not a separate module).

**Server name:** Soul Survivors
**Vision:** Bleach-inspired WotLK private server — all NPCs, zones, and player content reskinned/converted to Bleach lore. Classless system via Draft Mode (random spell selection per level-up), unique character builds, prestige system. Target: 500 AI bots + friends playing together.

- Authserver port: **3724**
- Worldserver port: **8085**
- Install prefix: `$HOME/azeroth-server`
- Working directory: `/home/sykx/SS/azerothcore-wotlk`
- Tailscale IP: `100.107.146.103` (friends connect via Tailscale VPN)
- DB credentials: user `acore` / pass `acore` (local only, never exposed publicly)

## Network & Hosting

- Friends connect via **Tailscale** — they install Tailscale, get approved, set `realmlist.wtf` to `100.107.146.103`
- Realmlist DB must match: `UPDATE acore_auth.realmlist SET address='100.107.146.103' WHERE id=1;`
- Your local `realmlist.wtf`: `set realmlist 100.107.146.103`
- **ProtonVPN must be OFF** while server is running — it breaks Tailscale routing
- Never expose MySQL port 3306 publicly
- For truly public hosting: forward TCP/UDP 3724 and TCP 8085 on router only

**Create friend accounts** (worldserver console only, not in-game chat):
```
account create <username> <password>
```

## Build Commands

**Skip building unless explicitly requested. All builds must run inside WSL Ubuntu, not Windows/MSYS2.**

```bash
mkdir -p ~/SS/azerothcore-wotlk/build && cd ~/SS/azerothcore-wotlk/build
cmake .. \
  -DCMAKE_INSTALL_PREFIX=$HOME/azeroth-server \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DSCRIPTS=static \
  -DMODULES=static

make -j$(nproc)
make install
```

**When Eluna is added** (planned), append `-DELUNA=1` to the cmake command.

**Map extractor tools** (build separately after main build):
```bash
cd ~/SS/azerothcore-wotlk/build
cmake .. -DTOOLS_BUILD=maps-only
make -j$(nproc) map_extractor vmap4_extractor vmap4_assembler mmaps_generator
make install
```

**Map extraction** (run from WoW client dir, ~20-30 min):
```bash
cd '/mnt/c/Program Files (x86)/WoW'
~/azeroth-server/bin/map_extractor
~/azeroth-server/bin/vmap4_extractor
~/azeroth-server/bin/vmap4_assembler Buildings vmaps
~/azeroth-server/bin/mmaps_generator
# Move maps/ vmaps/ dbc/ Cameras/ mmaps/ to ~/azeroth-server/data/
```

### Key CMake options
- `SCRIPTS`: none, static, dynamic (default: static)
- `MODULES`: none, static, dynamic (default: static)
- `ELUNA`: 1 to enable Lua scripting engine (required for Draft Mode and Lua scripts)
- `TOOLS_BUILD`: none, all, db-only, maps-only (default: none)
- `USE_COREPCH` / `USE_SCRIPTPCH`: Precompiled headers (default: ON)

## Four Databases Required

- `acore_auth` — Accounts, realm list
- `acore_characters` — Characters, inventories
- `acore_world` — Game content
- `acore_playerbots` — Bot AI state (required by mod-playerbots)

**DB backup location:** `~/SS/backups/YYYY-MM-DD/`
```bash
# Backup all 4 DBs
for db in acore_auth acore_characters acore_world acore_playerbots; do
  mysqldump -u acore -pacore --single-transaction $db | gzip > ~/SS/backups/$(date +%Y-%m-%d)/$db.sql.gz
done

# Restore
for db in acore_auth acore_characters acore_world acore_playerbots acore_ale; do
  gunzip -c ~/SS/backups/YYYY-MM-DD/$db.sql.gz | mysql -u root $db
done
```

**HeidiSQL connection:** Host `172.24.163.59` (WSL IP, changes on reboot — run `ip addr show eth0 | grep inet`), port 3306, user `acore`, pass `acore`.

## Installed Modules (`modules/`)

| Module | Purpose |
|--------|---------|
| `mod-ah-bot` | Populates Auction House with bot listings |
| `mod-autobalance` | Scales dungeon/raid difficulty to party size |
| `mod-transmog` | Transmogrification NPC and system |
| `mod-npc-buffer` | NPC that casts buffs on players |
| `mod-1v1-arena` | 1v1 arena system |
| `mod-ar-ac` | Arena ratings |
| `mod-cfbg` | Cross-faction battlegrounds |
| `mod-faction-icons` | Faction icons in chat |
| `mod-gain-honor-guard` | Honor gain for guards |
| `mod-learn-spells` | Auto-learn spells on level |
| `mod-npc-services` | Service NPC |
| `mod-pvp-titles` | PvP title rewards |
| `mod-skip-dk-starting-area` | Skip DK intro |
| `mod-world-chat` | Global chat channel |
| `mod-zone-difficulty` | Per-zone difficulty scaling |
| `mod-playerbots` | 500 AI bots (built into core) |

## Planned Additions

- **Eluna Lua engine** (`-DELUNA=1`) — prerequisite for all Lua scripts
- **Prestige-and-Draft-Mode** (`https://github.com/Youpeoples/Prestige-and-Draft-Mode`) — classless random spell draft per level-up, prestige reset system. Core of the Bleach server vision.
- **OnApplyWeaponDamage hook** (cherry-pick `87fbdb796`) — C++ core change, needs rebuild
- **AzerothCore catalogue modules** — assess each individually: SQL/Lua = safe to add, C++ = needs rebuild + compat test with playerbot branch

### Module addition strategy
1. SQL-only scripts: drop in and apply, zero risk
2. Lua scripts: require Eluna build first, then just drop `.lua` files
3. C++ modules: clone to `modules/`, rebuild, test bot stability
4. Core patches: cherry-pick carefully, test merge conflicts with playerbot branch
5. **Always take a DB backup + git tag before adding anything new**

## Bot Configuration

Config file: `~/azeroth-server/etc/modules/playerbots.conf`

Key settings (current):
- `AiPlayerbot.MinRandomBots = 500` / `MaxRandomBots = 500`
- `AiPlayerbot.RandomBotAccountCount = 120`
- `AiPlayerbot.ReactDelay = 300` (slowed for performance)
- `AiPlayerbot.DisabledWithoutRealPlayer = 1` ← bots only run when players are online
- `AiPlayerbot.RpgDelay = 5000` (active/busy behavior)

**Bots log in 30 seconds after first real player, log out 5 min after last real player leaves.**

## Session & Performance Settings

In `worldserver.conf`:
- `SocketTimeOutTime = 600000` — kick idle players at character select after 10 min
- `SocketTimeOutTimeActive = 600000` — kick idle in-world players after 10 min
- `MapUpdate.Threads = 1`
- WSL memory limit: 12GB, 6 processors (`~/.wslconfig`)

**If worldserver hits ~100% CPU after long uptime:** restart it (`server restart 5` in console). Root cause is bots running with no players — fixed by `DisabledWithoutRealPlayer = 1`.

## Startup Procedure

1. **Windows reboot auto-fix:** Scheduled task `SoulSurvivors-Portproxy` runs at boot as SYSTEM, auto-detects WSL IP and sets portproxy rules for ports 3724 and 8085. Script at `C:\Users\sykx\startup-wow-server.ps1`. If portproxy breaks, run the script manually as Administrator.

2. Open WSL terminal — start authserver: `~/azeroth-server/bin/authserver`
3. Open second WSL terminal — start worldserver: `~/azeroth-server/bin/worldserver`
4. Wait for `AC>` prompt before connecting
5. Set `realmlist.wtf` to `192.168.40.181` (LAN/host play)
6. Launch WoW client

**Realmlist DB is set to `192.168.40.181` for local/LAN play. For external friends switch to `24.101.102.96` once port forwarding is confirmed working.**

**ProtonVPN must be OFF during server operation.**

## Known Connection Issues & Fixes

**"Bounced back to realm selection" every session:** Almost always caused by one of:
1. Portproxy rules stale (WSL IP changed after reboot) → run `C:\Users\sykx\startup-wow-server.ps1` as Admin
2. Realmlist DB set to wrong IP → `UPDATE acore_auth.realmlist SET address='192.168.40.181', localAddress='192.168.40.181', localSubnetMask='255.255.255.0' WHERE id=1;`
3. Worldserver overloaded (check CPU with `ps aux | grep worldserver`) → `server restart 5` in console

**Bots killing CPU:** Both settings required together:
- `AiPlayerbot.RandomBotLoginAtStartup = 0` — bots don't load on server start
- `AiPlayerbot.DisabledWithoutRealPlayer = 1` — bots only load 30s after a real player logs in

**AHBot killing CPU:** `ItemsPerCycle = 20` (was 200). If it spikes again, set `Account = 0` temporarily.

## Lua Scripts Version Control

Lua scripts have their own git repo at `~/azeroth-server/lua_scripts/` (separate from the core repo).

```bash
cd ~/azeroth-server/lua_scripts
git add -A && git commit -m "description"
```

Key scripts and their purpose:
| Script | Purpose |
|--------|---------|
| `spell_choice.lua` | Draft Mode — spell selection on level-up |
| `prestige_chromie.lua` | Prestige/reset NPC gossip |
| `prestige_and_spell_choice_config.lua` | Draft config (start level, rerolls, pool size) |
| `stat_boost_system.lua` | 4 random stat boost choices per draft level |
| `wanderer_start.lua` | Race restriction (Human only for players), Wanderer welcome |
| `spell_history_server.lua` | Spell history popup server handler |
| `paragon_*.lua` | Paragon XP/stat system (requires `acore_ale` DB) |
| `_package_aliases.lua` | ALE require() compatibility shim |
| `classic.lua` | OOP library (sets `_G.Object` global for paragon) |
| `.client_addons/SpellChoice.lua` | Draft UI client code |
| `.client_addons/spell_history_client.lua` | Spell history popup client |

**AIO addon distribution:** Server uses `AIO.AddAddon(path)` to push client code to players on login.
Client addons in `.client_addons/` — ALE skips this dir (starts with `.`), AIO distributes contents.

**Draft system config** (`prestige_and_spell_choice_config.lua`):
- `DRAFT_START_LEVEL = 10` — wanderers quest freely until level 10
- `STAT_BOOST_COUNT = 4` — 4 extra stat boost choices per level-up
- `DRAFT_MODE_SPELLS = 3` — spell choices per level
- `POOL_AMOUNT = 45` — spell pool size (increase carefully)

**Custom DB tables** (acore_characters):
- `prestige_stats` — draft state, rerolls, bans per player
- `drafted_spells` — all spells chosen via draft
- `draft_level_history` — history of choices per level (powers spell history UI)
- `draft_bans` — spells a player has banned
- `character_stat_boosts` — cumulative stat boosts from draft choices

**Custom DB** (acore_ale): Paragon system tables (auto-created on first load)

## Planned Features (Next Sessions)

### Human Race / Character Creation
- Players: Human only (enforced via `wanderer_start.lua` server-side kick on creation)
- "Body type" subrace system: NPC after login lets player pick size variant (morph from human/gnome/elf/etc models) — all remain Human race ID for faction/bot compatibility
- Hollow race: separate DBC model + client patch via `build-mpq`
- Bots: keep all races — they need variety for world population

### FFXI-style Combat System (Major C++ project)
- **Manual Attack (Strike)**: Replace auto-attack feel; give all level-1s a Strike ability
  - Short-term: Give `Heroic Strike r1` (renamed to Strike) at creation — manual feel
  - Long-term: C++ hook to disable auto-attack for player race=1 characters
- **TP/Weapon Skill bar**: New resource (0-1000 TP), fills on melee hits, drains on Weapon Skills
  - Requires C++ new power type OR persistent aura tracking
  - Client: AIO addon HUD bar
- **Stamina bar**: New resource, drains on run/jump/dodge/attack, regens at rest
- **Skill Chains**: 2-3 Weapon Skills in sequence trigger elemental burst
- **Sense ability**: Show mana-charging enemies from range (beam effect)

### UI Theme (Black/Gold — ongoing)
- All custom AIO windows use: `bgFile="Interface\\DialogFrame\\UI-DialogBox-Background-Dark"`, gold border `#C9A84C`
- Spell History popup: `/ssh` or `.history` — LIVE
- Custom spellbook: planned (AIO addon, replaces WoW spellbook for draft players)

## Save Points & Version Control

```bash
# Tag a working state
git tag -a "working-YYYY-MM-DD" -m "Description"
git push origin Playerbot
git push origin --tags

# Restore code to a tag
git checkout working-YYYY-MM-DD

# Restore DB from backup
for db in acore_auth acore_characters acore_world acore_playerbots acore_ale; do
  gunzip -c ~/SS/backups/YYYY-MM-DD/$db.sql.gz | mysql -u root $db
done
```

**Always backup DB + tag git before any destructive change.**

Current stable tag: `working-2026-03-24`

## Branch Divergence Policy

This branch (Playerbot) already diverges from upstream AzerothCore. Each core patch cherry-picked increases divergence. Strategy:
- Pull upstream playerbot branch updates periodically, resolve conflicts manually
- Keep a log of all core patches applied (cherry-picks, custom changes) so conflicts are predictable
- Prefer modules over core patches wherever possible

## Architecture

### Two server executables
- **authserver**: Authentication and realm selection
- **worldserver**: All gameplay

### Source layout
- `src/common/` — Networking, crypto, config, logging, threading
- `src/server/game/` — Core game logic (~52 subsystems)
- `src/server/scripts/` — Boss/spell/instance/command scripts
- `src/server/database/` — Database abstraction
- `src/server/shared/` — Shared between auth and world

### Playerbot system
- Bots are real player accounts spawned by the server
- Bot behavior driven by `PlayerbotAI` and strategy selectors
- Source: `modules/mod-playerbots/src/`

SQL updates: `data/sql/updates/pending_*/<db>/` until merged.

## Commit Message Format

```
Type(Scope): Short description
```
Types: feat, fix, refactor, style, docs, test, chore
Scopes: Core (C++), DB (SQL), Config, Module

## Code Style

- 4-space indentation for C++ (no tabs)
- 2-space indentation for JSON/YAML/shell
- UTF-8, LF line endings, max 80 chars/line
- No braces around single-line statements
- Use `{}` format specifiers, not `%u`/`%s`

## GM / Admin Cheat Sheet

All dot-commands work in-game chat. Console commands (no dot) work in the worldserver terminal.

### XP & Rates
| Command | Effect |
|---------|--------|
| `.xp 0` | Disable XP gain |
| `.xp 1` | Normal XP (×1) |
| `.xp 2` | Double XP (×2) |
| `.xp N` | Set XP multiplier to N |
| `.server rate XP.Kill N` | Change kill XP rate globally |

### Character / Level
| Command | Effect |
|---------|--------|
| `.level N` | Set your level to N |
| `.additem <id> [qty]` | Add item to your bag |
| `.learn <spellId>` | Learn a spell |
| `.unaura <spellId>` | Remove an aura/buff |
| `.cheat god on/off` | Toggle god mode |
| `.cheat explore on/off` | Reveal full map |
| `.cheat casttime on/off` | Instant casts |
| `.cheat cooldown on/off` | No cooldowns |
| `.cheat power on/off` | Infinite mana/energy/rage |
| `.modify speed N` | Set movement speed (1 = normal, 2 = double) |
| `.modify scale N` | Set model scale (1 = normal) |
| `.modify money N` | Add N copper coins |

### Teleport & Location
| Command | Effect |
|---------|--------|
| `.tele <name>` | Teleport to named location |
| `.tele list` | List teleport locations |
| `.goname <player>` | Teleport to a player |
| `.summon <player>` | Summon player to you |
| `.appear <player>` | Appear at player location |
| `.gps` | Show current coordinates |
| `.go xyz X Y Z [mapId]` | Teleport to exact coords |

### NPC & Object Inspection
| Command | Effect |
|---------|--------|
| `.npc info` | Show targeted NPC's ID, entry, faction |
| `.npc move` | Move NPC to your position |
| `.npc delete` | Delete targeted NPC |
| `.gobject info` | Show targeted game object info |
| `.lookup spell <name>` | Find spell ID by name |
| `.lookup item <name>` | Find item ID by name |
| `.lookup creature <name>` | Find creature ID |

### Draft / Spell History (Soul Survivors custom)
| Command | Effect |
|---------|--------|
| `/ssh` or `.history` in chat | Open Spell History popup |
| `SC_CHECK` whisper to self | Force-refresh draft status (debug) |

### Server Console Commands (in worldserver terminal)
| Command | Effect |
|---------|--------|
| `reload ale` | Hot-reload all Lua scripts (no restart) |
| `account create <user> <pass>` | Create a player account |
| `account set gmlevel <user> 3 -1` | Make account GM level 3 (all realms) |
| `server restart 5` | Restart worldserver in 5 seconds |
| `server shutdown 5` | Shutdown worldserver in 5 seconds |
| `ban account <user> <dur> <reason>` | Ban account |
| `send mail <player> "Subject" "Body"` | Send in-game mail to player |
