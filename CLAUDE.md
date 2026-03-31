# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **mod-playerbots fork** of AzerothCore (`https://github.com/mod-playerbots/azerothcore-wotlk`), branch `Playerbot`. The Playerbot AI system is built directly into the core (not a separate module).

**Server name:** Soul Survivors
**Vision:** Dark Souls–style action RPG inside WoW 3.3.5. Bleach lore skin. Manual combat only — no auto-attack. Stats are purely player-allocated (no automatic DBC gains). Draft Mode classless system: random spells + 15 passive traits + free stat distribution on every level-up. Target: 500 AI bots + friends.

- Authserver port: **3724**
- Worldserver port: **8085**
- Install prefix: `$HOME/azeroth-server`
- Working directory: `/home/sykx/SS/azerothcore-wotlk`
- Tailscale IP: `100.107.146.103` (friends connect via Tailscale VPN)
- DB credentials: user `acore` / pass `acore` (local only, never exposed publicly)

## Network & Hosting

**Primary connection: Tailscale VPN** (no port forwarding needed)
- Friends install Tailscale, get approved on your network, set `realmlist.wtf` to `100.107.146.103`
- Realmlist DB: `UPDATE acore_auth.realmlist SET address='100.107.146.103' WHERE id=1;`
- Your local `realmlist.wtf`: `set realmlist 100.107.146.103`
- **ProtonVPN must be OFF** while server is running — it breaks Tailscale routing
- Never expose MySQL port 3306 publicly

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
| `mod-ale` | **LIVE** — ALE Lua engine (AzerothCore Lua Extension); hot-reload with `.reload ale` |

## HARD-LEARNED RULES — READ BEFORE EVERY SESSION

These are real mistakes that wasted days. Do not repeat them.

### Combat System: Auto-Attack
**Current live approach (Lua-only, no C++ rebuild needed):**
1. Spell 6603 (Attack toggle button) is stripped from players on login via `awakened_start.lua` — removes the button from the action bar
2. Weapon skills (Unarmed=162, Daggers=173, Staves=136, Wands=351) set to 0/0 via `SetSkill(id, 0, 0, 0)` — all auto-attacks miss
3. AIO client addon `AutoAttackSuppress.lua` calls `StopAttack()` when `PLAYER_REGEN_DISABLED` fires (combat entry) — belt-and-suspenders
4. **Result**: Right-click draws weapon (enters combat stance, weapon visible), but character never actually swings

**What NOT to do:**
- Do NOT set weapon skills to 1 — that lets auto-attacks land
- Do NOT try to prevent right-clicking a target — that breaks targeting entirely
- Do NOT suggest C++ as the only option for disabling auto-attack in early sessions. The AIO StopAttack() + 0 weapon skills approach works well enough to ship
- Do NOT strip spell 6603 and call it done. Right-click attack still fires without 6603. Need the weapon skill = 0 AND the AIO StopAttack on combat entry

**True fix (future, C++ required):** Hook `Unit::AttackStart()` and return early for player race=1. This completely prevents the attack timer from starting.

### Stats: No Auto-Distribution on Level-Up
**Current live approach:**
- `no_auto_stats.lua` resets all primary stats to base 5/5/5/5/5 (STR/AGI/STA/INT/SPI) on every level-up AND login — runs 50ms after InitStatsForLevel() so it's imperceptible
- HP = 50 + (STA × 10) — explicitly set after every stat reset
- `ApplyAllBoosts()` (global in stat_boost_system.lua) re-applies all stored draft boosts on top
- `prestige_stats.unspent_stat_points` tracks 5 free points per level — player distributes via the Draft window stat panel
- `StatDist.Commit` AIO handler applies allocations as `STR_FREE/AGI_FREE/STA_FREE/INT_FREE/SPI_FREE` in `character_stat_boosts`
- Server sends `StatDistUpdate` ("points:N,STR:X,...") to client after every level-up and login

**What NOT to do:**
- Do NOT set BASE stats to the DBC warlock values (20/20/21/23/26). The base is 5/5/5/5/5 forever
- Do NOT apply stats with a long delay (500ms+). 50ms is imperceptible. 3500ms+ is visible flicker
- Do NOT make `ApplyAllBoosts` local — it must be global so other scripts (no_auto_stats.lua) can call it
- Do NOT send `StatDistUpdate` at < 350ms delay after level-up — `no_auto_stats.lua` resets at 100ms; client receives DBC values (20/20/21/23/26) if the update fires before the reset settles. 350ms is safe.

### AIO Client Addons
**What NOT to do:**
- Do NOT use `AIO.AddAddon()` for `SpellChoice.lua`. It is distributed via the PrestigeSystem WoW addon folder, NOT AIO. Adding it to AIO causes a double-load crash (`SSCard1` duplicate frame name)
- Always guard `RegisterAddonMessagePrefix` with `if RegisterAddonMessagePrefix then ... end` — some WoW 3.3.5 clients don't have it, and calling nil crashes the entire file silently
- All `RegisterAddonMessagePrefix` prefix strings must be **≤ 16 characters** in WoW 3.3.5. "SpellChoiceBoosts" = 17 chars → crash. Use: SCBoosts, SCRarities, SCDrafts, SCRerolls, SCBans, SCStatus, SCBansLeft, SCBanDenied, SCBanAccepted

**Current short prefix mapping:**
| Long (broken) | Short (≤16, working) |
|---|---|
| SpellChoiceBoosts | SCBoosts |
| SpellChoiceRarities | SCRarities |
| SpellChoiceDrafts | SCDrafts |
| SpellChoiceRerolls | SCRerolls |
| SpellChoiceBans → SpellChoiceBans | SCBans |
| SpellChoiceStatus | SCStatus |
| SpellChoiceBansLeft | SCBansLeft |
| SpellChoiceBanDenied | SCBanDenied |
| SpellChoiceBanAccepted | SCBanAccepted |

### Lua 5.1 (WoW client)
- WoW 3.3.5 uses Lua 5.1. `goto` / `::label::` syntax is 5.2+. Use `if/else` instead
- Server-side uses Lua 5.1 via ALE — same restrictions apply

### Slash Commands: How to Verify They Work
When asked to add a slash command like `/lvl`:
1. The command is registered with `SLASH_CMDNAME1 = "/lvl"` and `SlashCmdList["CMDNAME"] = fn`
2. If the file has a pcall wrapper, a crash BEFORE the slash command registration silently stops everything
3. Test: if file has `[SS-A]`...`[SS-K]` checkpoints, ALL must appear in chat for the command to work
4. If user says command doesn't work, the FIRST question is "which checkpoint was the last one you saw?" — NOT "try reloading"
5. NEVER assume the command works just because the code is written. The file may have crashed before reaching that line

### Draft Window: /lvl
- `/lvl` opens `SSDraftWindow` (v6 SpellChoice.lua)
- Opens the window even when no draft is pending (shows a chat message, window still opens so player can confirm it works)
- ADVANCE button (`SSAdvanceBtn`) appears automatically when server sends `SpellChoice` message
- If user says window doesn't open: check ALL [SS-X] checkpoints appeared in chat first

## GM / Cheat Sheet Commands

### XP Rate (per-player)
Uses `ExpModifier_AIO_Server.lua` + `ExpModifier_AIO_Client.lua`:
- In-game: talk to the XP Rate NPC or use the AIO interface
- Worldserver console: `server set xprate <multiplier>` (affects all players)
- SQL per-account: `UPDATE acore_characters.characters SET xpRate = 2 WHERE guid = <guid>;`

### GM Commands (in-game, `.` prefix)
```
.account set gmlevel <account> <level> -1   -- set GM level (0-4)
.levelup <N>                                 -- level up N times
.modify xp rate <N>                          -- multiply XP rate
.modify money <amount>                       -- give gold
.additem <itemId> <count>                    -- give item
.learn <spellId>                             -- learn spell
.aura <spellId>                              -- apply aura
.cast <spellId>                              -- cast on self
.go xyz <x> <y> <z> <mapId>                 -- teleport
.lookup spell <name>                         -- find spell ID
.lookup item <name>                          -- find item ID
.npc info                                    -- info on targeted NPC
.reload all                                  -- reload SQL tables
```

### Server Console (worldserver terminal)
```
server restart 5                 -- restart in 5 seconds
server shutdown 5                -- shutdown in 5 seconds
.reload ale                      -- reload all Lua scripts (hot-reload)
account create <user> <pass>     -- create player account
account set gmlevel <user> 3 -1  -- make GM
```

### Lua Script Reload (after any .lua file change)
```
.reload ale                      -- in worldserver console
/reload                          -- in WoW client (for SpellChoice.lua and PrestigeSystem addon)
```

### Common Debug Queries
```sql
-- Check player stats
SELECT * FROM acore_characters.character_stat_boosts WHERE player_guid = <guid>;
SELECT * FROM acore_characters.prestige_stats WHERE player_id = <guid>;

-- Reset a player's draft
DELETE FROM acore_characters.prestige_stats WHERE player_id = <guid>;
DELETE FROM acore_characters.character_stat_boosts WHERE player_guid = <guid>;
DELETE FROM acore_characters.drafted_spells WHERE player_id = <guid>;

-- Find player GUID
SELECT guid, name, level FROM acore_characters.characters WHERE name = '<name>';
```

## Planned Additions (C++ — requires rebuild)

- **OnApplyWeaponDamage hook** (cherry-pick `87fbdb796`) — C++ core change, needed for FFXI-style TP system
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
5. Set `realmlist.wtf` to `100.107.146.103` (Tailscale — works for local host AND friends)
6. Launch WoW client

**Realmlist DB uses Tailscale IP:** `UPDATE acore_auth.realmlist SET address='100.107.146.103' WHERE id=1;`
**Everyone (local + friends) uses `realmlist.wtf`: `set realmlist 100.107.146.103`**

**ProtonVPN must be OFF during server operation.**

## Known Connection Issues & Fixes

**"Bounced back to realm selection" every session:** Almost always caused by one of:
1. Portproxy rules stale (WSL IP changed after reboot) → run `C:\Users\sykx\startup-wow-server.ps1` as Admin
2. Realmlist DB set to wrong IP → `UPDATE acore_auth.realmlist SET address='100.107.146.103' WHERE id=1;`
3. Worldserver overloaded (check CPU with `ps aux | grep worldserver`) → `server restart 5` in console
4. Tailscale not running → start Tailscale on Windows before launching WoW

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
| `spell_choice.lua` | Draft Mode — spell selection on level-up (server) |
| `prestige_chromie.lua` | Prestige/reset NPC gossip |
| `prestige_and_spell_choice_config.lua` | Draft config (start level, rerolls, pool size) |
| `stat_boost_system.lua` | 4 random stat boost choices per draft level |
| `awakened_start.lua` | Race restriction (Human only for players), Awakened welcome |
| `spell_history_server.lua` | Spell history popup server handler |
| `paragon_*.lua` | Paragon XP/stat system (requires `acore_ale` DB) |
| `_package_aliases.lua` | ALE require() compatibility shim |
| `classic.lua` | OOP library (sets `_G.Object` global for paragon) |
| `faction_npc.lua` | "The Weaver" — faction choice NPC (entry 9000001, 8 factions) |
| `no_auto_stats.lua` | Resets stats to 5/5/5/5/5 on every level-up and login; sets HP = 50 + STA×10 |
| `auto_attack_suppress.lua` | Registers AIO AutoAttackSuppress.lua client addon |
| `soul_reputation.lua` | 3-track karma system (Positive/Negative/Neutral); death destinations; `/soulrep` popup |
| `racial_skills.lua` | Race movement skills (Blink/Shadowstep per soul_state) granted at level 5 |
| `signature_item.lua` | Assigns one permanent signature item per character (Zanpakuto/Mask/Bow/etc.) |
| `awakened_skills.lua` | Grants starter combat spells + silently assigns random element at creation |
| `.client_addons/SpellChoice.lua` | Draft UI — spell cards + passive cards + stat panel + admin console; `/lvl` opens it |
| `.client_addons/spell_history_client.lua` | Spell history popup client |
| `.client_addons/TPBar.lua` | TP (Tactical Points) bar — blue HUD bar below stamina, fills on melee |
| `.client_addons/AutoAttackSuppress.lua` | Calls StopAttack() on PLAYER_REGEN_DISABLED; `/autoattack on/off` |

**AIO addon distribution:** Server uses `AIO.AddAddon(path)` to push client code to players on login.
Client addons in `.client_addons/` — ALE skips this dir (starts with `.`), AIO distributes contents.
`SpellChoice.lua` is loaded via the `PrestigeSystem` WoW addon (traditional), NOT AIO — use `/reload` in client, no relog needed.

**SpellChoice.lua location:** `/mnt/c/Program Files (x86)/WoW/Interface/AddOns/PrestigeSystem/SpellChoice.lua`
This is NOT the same as `lua_scripts/.client_addons/SpellChoice.lua` (that's the old AIO version — ignored).

**Draft system config** (`prestige_and_spell_choice_config.lua`):
- `DRAFT_START_LEVEL = 10` — skills/traits/rerolls only unlock at level 10; stat points flow from level 1
- `DRAFT_MODE_REROLLS = 0` — no rerolls at character creation; first reroll at level 10, then every 5 levels
- `STAT_BOOST_COUNT = 4` — 4 extra stat boost choices per draft level
- `DRAFT_MODE_SPELLS = 3` — spell choices per level (UI supports up to 8 buttons)
- `POOL_AMOUNT = 45` — spell pool size (increase carefully)

**Draft level gate rules:**
- Levels 1-9: stat points only (5 per level). Window opens but shows informational message.
- Level 10+: stat points + skill/trait sessions + rerolls unlock together.
- Session formula: `total_expected = max(current, newLevel - 9)` — prevents session debt when GM-leveling.
- Rerolls: first at level 10, then every 5 levels (+1 each time).
- All new Human Warlocks start with `draft_state=1` (prestiged) from creation.

**Draft UI key facts** (`.client_addons/SpellChoice.lua`):
- 8 spell buttons, all using `SpellChoiceButtonTemplate` (256×256 card, scaled to 0.5 by default)
- Spell name displays ABOVE the card in a dark backdrop frame (`btn.nameBg`, 32pt font)
- Rarity colored dot: `WHITE8x8` + `SetVertexColor` — no external texture files required
- `/timeline` or `/draftline` — opens scrollable Draft Timeline history panel
- `/draftscale <n>` — resize all spell buttons live (0.2–2.0)
- DraftTimeline panel is inlined directly in SpellChoice.lua
- Morpheus font (`Fonts\\MORPHEUS.TTF`) used for draft window title
- Title: `"Awakened  -  Choose Your Path"`

**Player slash commands** (SpellChoice.lua client addon):
- `/lvl` — opens Draft window (stat panel always; skills/traits from level 10+)
- `/timeline` / `/draftline` — scrollable draft history
- `/draftscale <n>` — resize spell buttons
- `/soulrep` / `/soul` — Soul Alignment popup (3 karma bars + form + destiny)
- `/sigitem` / `/zanpakuto` / `/mask` / `/bow` — reveal signature item (name + flavor)
- `/autoattack on/off` — toggle auto-attack suppression debug
- `/ssh` — spell history popup

**GM-only slash commands** (auto-shown when GM rank > 0 via SpellChoiceIsGM message):
- `/adminlog` / `/adm` — toggle Admin Console (scrolling bottom-right frame, red border)
- `/msglog N` — dump last N suppressed lines to chat

**Message suppression (SpellChoice.lua):**
- Suppressed: stat gain noise, HP/mana change notifications, "You are now level", login spam, WorldChat module messages
- NOT suppressed: "You have learned" and "added to your spellbook" — players should see every skill/trait earned
- All suppressed messages route to Admin Console instead (GM only)

**Custom DB tables** (acore_characters):
- `prestige_stats` — draft state, rerolls, bans per player
- `drafted_spells` — all spells chosen via draft
- `draft_level_history` — history of choices per level (powers spell history UI)
- `draft_bans` — spells a player has banned
- `character_stat_boosts` — cumulative stat boosts from draft choices
- `player_faction` — faction chosen via The Weaver NPC
- `character_soul` — soul_state (0-7), soul_positive, soul_negative, soul_neutral (karma)
- `character_signature` — player's permanent signature item (soul_type, sig_name, sig_flavor)

**Custom DB** (acore_ale): Paragon system tables (auto-created on first load)

## Soul State / Race System (LIVE)

Soul states are stored in `character_soul.soul_state` (tinyint). All gameplay race logic keys off this column.

| ID | Name | Home World | Notes |
|----|------|-----------|-------|
| 0 | Awakened (Human) | Earth | Former Warlock class; players only |
| 1 | Spirit | Earth (temporary) | Wandering dead, 24hr window |
| 2 | Hollow | Hueco Mundo | Consumed by hunger |
| 3 | Shinigami | Soul Society | Must remain in SS unless on mission |
| 4 | Quincy | Earth | Traverses Earth ↔ Wandenreich as power grows |
| 5 | Fullbringer | Earth | Manipulates soul of objects |
| 6 | Vizard | Earth | Requires prior Hollow + Shinigami reincarnations |
| 7 | Demon | Hell | Bleach Hellverse; future race |

**Class rename:** `LOCALIZED_CLASS_NAMES_MALE["WARLOCK"] = "Awakened"` in SpellChoice.lua — display-only, bots unaffected.

## Karma / Soul Reputation System (LIVE — soul_reputation.lua)

Three alignment tracks. All start at 0 (Neutral starts at 5000). Adding Positive or Negative drains Neutral.

| Track | Color | Effect |
|-------|-------|--------|
| Positive | Green | Light deeds → Soul Society / Wandenreich |
| Negative | Red | Dark deeds → Hell |
| Neutral | Grey | Drains as pos/neg grow; default state |

**Death destinations:**
- Awakened/Shinigami/Fullbringer/Vizard: Positive → Soul Society, Negative → Hell
- Quincy: Positive → Wandenreich, Negative → Hell
- Hollow (state=2): always Hueco Mundo
- Demon (state=7): always Hell
- Within 500 points of each other: 20% random flip

**API:** `SoulRep.Add(guid, "positive"/"negative"/"neutral", amount)`, `SoulRep.GetDestiny(guid)`, `SoulRep.SetState(guid, state)`

## Movement Skills / Racial Skills (LIVE — racial_skills.lua)

All races get a movement skill at level 5. All mapped to Blink (1953) currently — custom DBC IDs planned.

| Race | Lore Name | Spell ID |
|------|-----------|---------|
| Awakened/Spirit | Flash Step | 1953 Blink |
| Hollow | Sonído | 1953 Blink |
| Shinigami | Shunpo | 1953 Blink |
| Quincy | Hirenkyaku | 1953 Blink + Loose (19434) + Shoot (5019) |
| Fullbringer | Bringer Light | 1953 Blink |
| Vizard | Shunpo | 1953 Blink |
| Demon | Hell Step | 36554 Shadowstep |

All races also receive Shoot (5019) — universal ranged attack.
`RacialSkills_OnStateChange(player)` exported for soul_state change hooks.

**Protected spell IDs** (never stripped by SpellBlock): 1953, 36554, 19434, 5019 + all draft/core spells.

## Signature Items (LIVE — signature_item.lua)

Each character gets one permanent signature item assigned silently on first login.

| Race | Item Type | Examples |
|------|-----------|---------|
| Awakened Human | Sacred Talisman | "Sacred Seal", "The Iron Bell" |
| Hollow | Hollow Mask | "The Fractured Star", "The Bone Crescent" |
| Shinigami | Zanpakuto | "Senbonzakura", "Zangetsu", "Hyorinmaru" |
| Quincy | Spirit Bow | "Heilig Bogen", "Silberner Sturm" |
| Fullbringer | Fullbring Item | "The Silver Watch", "The Worn Photo" |
| Vizard | Sword + Mask | "Zangetsu / The Fractured Star" |
| Demon | Corrupted Horns | "Twin Perdition", "Void Crown" |

`SignatureItem.Assign(player, soulState)` — API for reincarnation. `SignatureItem.Reveal(player)` — prints name + flavor in gold/grey.

## Admin Console (LIVE — SpellChoice.lua)

- `SSAdminConsole` ScrollingMessageFrame — bottom-right, 420×220, red border
- Shows automatically when server confirms GM rank > 0 via `SpellChoiceIsGM` AIO message
- All suppressed chat messages route here instead of chat
- 500-line scrollable buffer; mousewheel to scroll
- Toggle: `/adminlog` or `/adm`; dump to chat: `/msglog N`

## Planned Features (Next Sessions)

### Human Race / Character Creation
- Players: Human only (enforced via `awakened_start.lua` server-side kick on creation)
- "Body type" subrace system: NPC after login lets player pick size variant (morph from human/gnome/elf/etc models) — all remain Human race ID for faction/bot compatibility
- Hollow race: separate DBC model + client patch via `build-mpq`
- Bots: keep all races — they need variety for world population

### Soul System (Next Sessions)
- **Spirit form death timer**: 1 hour respawn, 24 hr spirit window → hollow hunger messages → forced Hollow morph
- **Portal system**: Random portals open from Hell/SS/HuecoMundo/Wandenreich to Earth on timer; non-Earth races get 50% stat debuff on Earth
- **Territory system**: Each race protects a zone; killing all NPCs flips it hostile until respawn
- **Hollow rage mode**: Hollow on Earth loses control — needs `Unit::OnAIUpdate` C++ hook (Lua partial: periodic AttackStart)
- **Demon race**: DBC model + custom abilities (Chains of Hell, etc.) — future full implementation
- **Flash Step DBC**: Custom spell IDs per race so client shows correct name/tooltip/FX (currently all Blink 1953)
- **Quest reskin**: `UPDATE quest_template SET LogTitle=..., LogDescription=...` for Bleach lore per race tutorial questlines

### Dark Souls / FFXI Combat System
**LIVE (Lua):**
- Auto-attack disabled: spell 6603 stripped + weapon skills zeroed + AIO `StopAttack()` on combat entry
- Right-click draws weapon (combat stance) but no swing — exactly what was wanted
- `auto_attack_suppress.lua` + `.client_addons/AutoAttackSuppress.lua` — toggle with `/autoattack`
- Strike (Heroic Strike r1, spell 78), Charge, Dash, Defend granted at creation via `awakened_skills.lua`

**NEEDS C++ (future):**
- `Unit::AttackStart()` hook for player race=1 — prevents attack timer at the engine level; cleaner than Lua suppression
- Heroic Strike currently works off the auto-attack cycle. After C++ hook, Strike should be reworked as a true instant melee hit (custom spell, custom DBC)
- TP bar: new resource (0–1000), fills on melee hits — needs C++ new power type or persistent aura
- Skill Chains: 2-3 Strike sequence triggers elemental burst
- Running stamina drain: `OnMove` hook

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
Last known good lua_scripts commit: see `cd ~/azeroth-server/lua_scripts && git log --oneline -5`

**Session 2026-03-30 additions** (not yet tagged):
- soul_reputation.lua, racial_skills.lua, signature_item.lua (new files)
- spell_choice.lua: level gate (stat only 1-9, sessions at 10+), reroll timing, admin console, class rename, /soulrep window
- SpellChoice.lua (client): admin console, message suppression, stat panel layout, Morpheus font, /soulrep, /sigitem, /soul commands
- awakened_skills.lua: silent element assignment (no announcement)
- awakened_start.lua: player pointer bug fix

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
