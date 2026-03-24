# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **mod-playerbots fork** of AzerothCore (`https://github.com/mod-playerbots/azerothcore-wotlk`), branch `Playerbot`. The Playerbot AI system is built directly into the core (not a separate module). The goal is a private WotLK 3.3.5a server with ~600 AI bots, transmog, auction house bots, autobalance, and NPC buffer — with friends able to join remotely.

- Authserver port: **3724**
- Worldserver port: **8085**
- Install prefix: `$HOME/azeroth-server`
- Working directory: `/home/sykx/SS/azerothcore-wotlk`

## Build Commands

**Skip building unless explicitly requested. All builds must run inside WSL Ubuntu, not Windows/MSYS2.**

```bash
# Open WSL terminal, then:
mkdir -p ~/SS/azerothcore-wotlk/build && cd ~/SS/azerothcore-wotlk/build
cmake .. \
  -DCMAKE_INSTALL_PREFIX=$HOME/azeroth-server \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DSCRIPTS=static \
  -DMODULES=static

make -j$(nproc)
make install
```

**Map extractor tools** (build separately after main build):
```bash
cd ~/SS/azerothcore-wotlk/build
cmake .. -DTOOLS_BUILD=maps-only
make -j$(nproc) map_extractor vmap4_extractor vmap4_assembler mmaps_generator
make install
# Tools install to ~/azeroth-server/bin/
```

**Map extraction** (run from WoW client dir, takes ~20-30 min total):
```bash
cd '/mnt/c/Program Files (x86)/WoW'
~/azeroth-server/bin/map_extractor
~/azeroth-server/bin/vmap4_extractor       # skip if Buildings/ folder already exists
~/azeroth-server/bin/vmap4_assembler Buildings vmaps
~/azeroth-server/bin/mmaps_generator
# Then move maps/ vmaps/ dbc/ Cameras/ mmaps/ to ~/azeroth-server/data/
```

Note: Tool target names use underscores: `map_extractor`, `vmap4_extractor`, `vmap4_assembler`, `mmaps_generator`.

### Key CMake options
- `SCRIPTS`: none, static, dynamic (default: static)
- `MODULES`: none, static, dynamic (default: static)
- `PLAYERBOTS`: 1 to enable if the flag exists in this fork
- `TOOLS_BUILD`: none, all, db-only, maps-only (default: none) — set to `maps-only` to build map extractors
- `USE_COREPCH` / `USE_SCRIPTPCH`: Precompiled headers (default: ON)

## Four Databases Required

This setup uses **4 databases** (not 3 like standard AzerothCore):
- `acore_auth` — Accounts, realm list
- `acore_characters` — Characters, inventories
- `acore_world` — Game content
- `acore_playerbots` — Bot AI state, travel nodes, guild/arena names (required by mod-playerbots)

**First-time DB setup:**
```bash
# Create acore_playerbots and grant access
mysql -u root -e "
  CREATE DATABASE IF NOT EXISTS acore_playerbots DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  GRANT ALL PRIVILEGES ON acore_playerbots.* TO 'acore'@'localhost';
  FLUSH PRIVILEGES;"

# Import playerbots base SQL (worldserver auto-updates after this)
cd ~/SS/azerothcore-wotlk/modules/mod-playerbots/data/sql/playerbots/base
for f in $(ls *.sql | sort); do mysql -u acore -pacore acore_playerbots < "$f" 2>/dev/null; done
```

**If world DB is missing tables** (e.g. after fresh clone with old DB): reimport base schema:
```bash
mysql -u acore -pacore -e 'DROP DATABASE acore_world; CREATE DATABASE acore_world DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
cd ~/SS/azerothcore-wotlk/data/sql/base/db_world
for f in $(ls *.sql | sort); do mysql -u acore -pacore acore_world < "$f" 2>/dev/null; done
# worldserver will auto-apply pending updates on next start
```

**worldserver restarts after applying DB updates** — this is normal. Run it again and it will load fully once all updates are applied. If an update fails mid-run, apply remaining SQL files manually and insert them into the `updates` table.

## Installed Modules (`modules/`)

These are cloned and will be compiled automatically when `MODULES=static`:

| Module | Purpose |
|--------|---------|
| `mod-ah-bot` | Populates the Auction House with bot listings |
| `mod-autobalance` | Scales dungeon/raid difficulty to party size |
| `mod-transmog` | Transmogrification NPC and system |
| `mod-npc-buffer` | NPC that casts buffs on players for free |

Each module has its own SQL files under `modules/<name>/data/sql/` that are auto-applied on first worldserver start.

## Architecture

### Two server executables
- **authserver** (`src/server/apps/authserver/`): Authentication and realm selection
- **worldserver** (`src/server/apps/worldserver/`): All gameplay

### Source layout
- `src/common/` — Networking, crypto, config, logging, threading, utilities
- `src/server/game/` — Core game logic (~52 subsystems): Entities, Spells, Maps, AI, Handlers, Scripting, Server
- `src/server/scripts/` — Boss/spell/instance/command content scripts
- `src/server/database/` — Database abstraction layer
- `src/server/shared/` — Shared between auth and world servers

### Playerbot system (built into core)
Playerbot source lives under `src/server/game/AI/PlayerAI/` or similar. Key concepts:
- Bots are real player accounts spawned by the server — they appear in the world as players
- Bot behavior is driven by `PlayerbotAI` and strategy selectors
- Config: `worldserver.conf` section `[Playerbot]` — controls bot count, login behavior, strategy
- Bots can be added via GM command: `.bot add <name>` or auto-spawned at startup

### Three databases
- `acore_auth` — Accounts, realm list
- `acore_characters` — Characters, inventories, progress
- `acore_world` — Game content (creatures, items, quests, loot)

SQL updates: `data/sql/updates/pending_*/<db>/` until merged, then `data/sql/updates/<db>/`.

### Scripting system
Scripts inherit from `SpellScript`, `CreatureScript`, `InstanceMapScript`, etc. Each script file implements `AddSC_*()` which is called from regional `*_script_loader.cpp` files.

## Configuration for 600 Bots

In `$HOME/azeroth-server/etc/worldserver.conf`, set:

```ini
# Playerbot settings
PlayerbotAI.enabled = 1
PlayerbotAI.maxNumBots = 600
PlayerbotAI.BotAutologin = 1
PlayerbotAI.numMinBots = 500
PlayerbotAI.RandomBotAccountPrefix = "rndbot"
PlayerbotAI.RandomBotAccountCount = 200
PlayerbotAI.RandomBotSpawnDelay = 1000

# Spread bots across races and classes
PlayerbotAI.RandomBotMapsAsString = "0 1 530 571"
```

Create bot accounts with the in-game GM command:
```
.playerbot random init
```
Or via the worldserver console after first start.

## AH Bot Configuration

In `$HOME/azeroth-server/etc/worldserver.conf` (added by mod-ah-bot):
```ini
AHBot.EnableSeller = 1
AHBot.EnableBuyer = 1
AHBot.Account = 1          # AH bot account ID (create a dedicated account)
AHBot.GUID = 1             # Character GUID of the AH bot character
AHBot.ItemsPerCycle = 200
```
After first start, use `.ahbot items` in-game to check status.

## Autobalance Configuration

In `worldserver.conf` (added by mod-autobalance):
```ini
AutoBalance.enable = 1
AutoBalance.LevelScaling = 1
AutoBalance.PlayerChangeNotify = 1
AutoBalance.DungeonScaleDownXP = 0
```

## Friends Joining (Network Setup)

For friends to connect from outside your LAN:
1. **Forward ports** on your router: TCP/UDP 3724 (auth) and TCP 8085 (world)
2. In `acore_auth` DB, update the realmlist: `UPDATE realmlist SET address = 'YOUR_PUBLIC_IP' WHERE id = 1;`
3. Friends set their `realmlist.wtf` to: `set realmlist YOUR_PUBLIC_IP`
4. Create accounts for friends with: `.account create <username> <password>` in worldserver console

For LAN-only play: use your LAN IP instead of public IP.

## Commit Message Format

```
Type(Scope): Short description
```
Types: feat, fix, refactor, style, docs, test, chore
Scopes: Core (C++), DB (SQL)

## Code Style

- 4-space indentation for C++ (no tabs)
- 2-space indentation for JSON/YAML/shell
- UTF-8, LF line endings, max 80 chars/line
- No braces around single-line statements
- Use `{}` format specifiers, not `%u`/`%s`
