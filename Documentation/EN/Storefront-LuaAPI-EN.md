# 🧊 IceBoxStorefront Plugin — `Storefront` Lua API

## Complete English Documentation

> The **IceBoxStorefront Plugin** integrates the **Steamworks SDK** into IceBoxEngine and exposes a
> single Lua table — **`Storefront`** — to your gameplay scripts (`.ice_class`, `.icemap`, `.ice_widget`).
>
> Through `Storefront` you get **achievements**, **stats**, **leaderboards**, **Steam Cloud**,
> **friends**, **rich presence**, the **Steam overlay**, **lobbies (matchmaking)**, **P2P networking**,
> **voice chat**, **DLC & store**, the **Steam Workshop** (create / update / subscribe), **Steam Input**
> (controllers, Steam Deck), **auth tickets**, **screenshots**, and a large set of platform utilities.
>
> This document describes **every** Lua function, enum, callback, and returned table the plugin exposes,
> with signatures, parameter tables, return values, and runnable examples.

> **Not a Valve product.** IceBoxStorefront Plugin is developed by IceBoxCrew Studio and is **not made by,
> affiliated with, endorsed by or sponsored by Valve Corporation**. Steam, Steamworks, Steam Deck, Steam
> Machine, Steam Frame, Steam Cloud, Steam Workshop, Steam Input, Big Picture and Proton are trademarks
> and/or registered trademarks of Valve Corporation; this document uses those names descriptively only.
> **The Steamworks SDK is not included with this plugin** — you download it from Valve yourself, under
> Valve's own [Steamworks SDK Access Agreement](https://partner.steamgames.com/doc/sdk). See
> [The Steamworks SDK](#the-steamworks-sdk), and `DISTRIBUTION.md` before you upload a build.

> **Free, but not open source.** The plugin ships as a compiled library with its manifest, node catalog and
> this documentation. There is no source code in the package and none is published. Use is free, commercial
> games included; the compiled plugin travels inside the game you release. See `LICENSE.txt`.

---

## 📑 Contents

1. [Overview](#1-overview)
2. [Installation & Setup](#2-installation--setup)
   - [Where the plugin lives](#where-the-plugin-lives)
   - [The Steamworks SDK](#the-steamworks-sdk)
   - [Configuring your AppId](#configuring-your-appid)
   - [`steam_appid.txt` for development](#steam_appidtxt-for-development)
   - [Restarting through Steam](#restarting-through-steam)
   - [What ships with your game, and what must not](#what-ships-with-your-game-and-what-must-not)
   - [Passing the plugin itself on — and why not to](#shipping-the-plugin-itself)
3. [Core Concepts](#3-core-concepts)
   - [The `Storefront` table](#the-storefront-table)
   - [The frame tick — `Storefront.Tick()`](#the-frame-tick--storefronttick)
   - [Result codes](#result-codes)
   - [Synchronous vs. asynchronous calls](#synchronous-vs-asynchronous-calls)
   - [The `UserHandle` table](#the-userhandle-table)
   - [Binary data as Lua strings](#binary-data-as-lua-strings)
   - [Backend availability and Steam-only calls](#backend-availability-and-steam-only-calls)
4. [Session & Identity](#4-session--identity)
5. [Achievements](#5-achievements)
6. [Stats](#6-stats)
7. [Leaderboards](#7-leaderboards)
8. [Steam Cloud](#8-steam-cloud)
9. [Friends, Rich Presence & Overlay](#9-friends-rich-presence--overlay)
10. [Lobbies (Matchmaking)](#10-lobbies-matchmaking)
11. [P2P Networking](#11-p2p-networking)
12. [Voice Chat](#12-voice-chat)
13. [DLC & Store](#13-dlc--store)
14. [Workshop (UGC)](#14-workshop-ugc)
15. [Steam Input & Steam Deck](#15-steam-input--steam-deck)
16. [Platform Utilities](#16-platform-utilities)
17. [Authentication Tickets](#17-authentication-tickets)
18. [App & Apps Information](#18-app--apps-information)
19. [Events & Callbacks](#19-events--callbacks)
20. [Steam Timeline & Game Recording](#20-steam-timeline--game-recording)
21. [Enumerations Reference](#21-enumerations-reference)
22. [Returned Tables Reference](#22-returned-tables-reference)
23. [Practical Examples](#23-practical-examples)
24. [Troubleshooting & FAQ](#24-troubleshooting--faq)
25. [Visual Scripting Nodes](#25-visual-scripting-nodes)

---

## 1. Overview

The Storefront plugin is a native C++ plugin (`IceBoxStorefront.dll` / `IceBoxStorefront.so` /
`IceBoxStorefront.dylib`) that links against the **Steamworks SDK**. When the engine loads it, the plugin:

1. Resolves your Steam **AppId** (see [Configuring your AppId](#configuring-your-appid)).
2. Initializes `SteamAPI` and verifies the local user is logged in to the Steam client.
3. Registers a global **`Storefront`** Lua table into every Lua state the engine creates.
4. Pumps Steam's callbacks **every frame** automatically and delivers the queued results to Lua
   (you call neither `SteamAPI_RunCallbacks` nor `Storefront.Tick()` yourself).

All gameplay code talks to Steam through the `Storefront` table. The table is **backend-agnostic** by design:
the core surface (achievements, stats, leaderboards, cloud, friends, lobbies, P2P, DLC) is modeled as a generic
"storefront", and a set of **Steam-specific** extensions (Steam Deck, Workshop, Steam Input, voice, auth tickets,
overlay invite dialogs) sit on top.

> **Platform:** Desktop only — **Windows**, **Linux**, **macOS**. The Steamworks SDK is desktop-only, so the
> plugin is skipped entirely on Android, iOS, and Web (Emscripten) builds. Always guard Steam code with
> [`Storefront.IsAvailable()`](#41-storefrontisavailable) so the same scripts run unchanged on other platforms.

> **Steam client required:** Steam features only work when the game is launched **through a running Steam client**
> by an account that **owns the AppId**. During development, see [`steam_appid.txt`](#steam_appidtxt-for-development).

---

## 2. Installation & Setup

### Where the plugin lives

Like every IceBox plugin, the Storefront plugin is a folder under your project's `Plugins/` directory. It is
named `IceBoxStorefront`, and the engine identifies it by that name:

```
Plugins/
└── IceBoxStorefront/
    ├── IceBoxStorefront.dll ← the compiled plugin (platform-specific, from the release)
    ├── steam_api64.dll      ← Steamworks runtime (you copy this here, from Valve)
    ├── plugin.json          ← plugin manifest
    ├── VisualScriptAPI.json ← node catalog (required at run time)
    ├── icon.png             ← plugin icon for the editor and launcher
    └── steam_config.json    ← your AppId (you create this)
```

The engine auto-discovers the folder, loads it when enabled, calls `OnUpdate` every frame, and binds the
`Storefront` table into Lua. You enable/disable plugins from the editor's **Plugins** panel.

> **The Steam runtime library.** It sits in the plugin folder, right next to `IceBoxStorefront.dll` /
> `IceBoxStorefront.so` / `IceBoxStorefront.dylib`, on **all three desktop platforms** — that is the one location
> every dynamic loader agrees on.
> On Windows the plugin loads it from there explicitly; on Linux the plugin carries an `$ORIGIN` RPATH; on macOS
> `libsteam_api.dylib` is built with the install name `@loader_path/libsteam_api.dylib` and *must* sit beside the
> library that links it. The engine picks a plugin's library by matching its name against the plugin, so the
> runtime sitting in the same folder is never mistaken for the plugin itself. You put it there once, copied out
> of your own Steamworks SDK download - `README.md` has the exact file per platform.

> ⚠️ **That folder is now self-contained for *you*, and it is not yours to hand on.** `steam_api64.dll` /
> `libsteam_api.so` / `libsteam_api.dylib` came out of Valve's SDK, and Valve licenses that SDK to you
> **nontransferably**. It may travel inside the game you release to players — that part Valve permits — but it
> must **not** be inside anything you give to another developer. Neither may the plugin itself: it is free, and
> republishing it is not permitted (Section 4.1 of `LICENSE.txt`). Send colleagues to the official download page
> instead — they get the same build in a minute, for free. `DISTRIBUTION.md` has the full list.

### The Steamworks SDK

The **Steamworks SDK is not bundled** with the plugin, and no part of it may be redistributed with the plugin —
each developer downloads it from the [Steamworks partner site](https://partner.steamgames.com/doc/sdk), accepting
Valve's *Steamworks SDK Access Agreement* on the way. Valve licenses the SDK to that developer directly;
IceBoxCrew Studio is not a party to it and sublicenses nothing.

You need **exactly one file** out of that download — the Steam runtime library — and you copy it into the plugin
folder once:

| Platform | File | From |
| --- | --- | --- |
| Windows x64 | `steam_api64.dll` | `redistributable_bin/win64/` |
| Linux x64 | `libsteam_api.so` | `redistributable_bin/linux64/` |
| macOS | `libsteam_api.dylib` | `redistributable_bin/osx/` |

The headers and the import library matter only to whoever compiles the plugin, and that is not you: the plugin
arrives already built. When **shipping your game**, the compiled `IceBoxStorefront` library and that runtime
travel inside your package — which Section 1.1 of your own agreement with Valve permits, because they go out
together with your application.

> **Minimum SDK version: v1.65.** The plugin uses `ISteamUtils::IsRunningOnSteamHardware()`,
> `GetSteamHardwareDefaultConfig()` and `IsRunningUnderProton()`, which arrived in v1.65 together with the removal
> of `IsRunningOnSteamDeck()`. Take the runtime library from a v1.65-or-newer SDK drop.

### Configuring your AppId

At load time the plugin resolves the Steam AppId from the first source that yields a non-zero value, in this order:

1. The **`ICEBOX_STEAM_APPID`** environment variable.
2. **`steam_config.json` in the plugin's own folder** — a JSON file with an `"AppId"` field. The engine tells the
   plugin where it was loaded from, so this works no matter what the folder is called or whether the plugin lives
   in the engine's or the project's `Plugins/`.
3. **`Plugins/IceBoxStorefront/steam_config.json`**, relative to the working directory.
4. **`steam_config.json`** in the working directory.
5. **`steam_appid.txt`** (a plain text file containing only the number), looked up in the same three places.

The recommended approach is `steam_config.json`:

```json
{
    "AppId": 480
}
```

> `480` is Valve's public **Spacewar** test AppId — useful for trying Steam features before your own app is
> provisioned. Replace it with your real AppId for production.

If **no** AppId can be resolved, the plugin writes a message to the engine log and **does not register a storefront
backend** — `Storefront.IsAvailable()` returns `false` and every call becomes a safe no-op. Your game still runs.
The same happens when `SteamAPI` itself fails to initialize (Steam not running, the account does not own the app,
…), so a missing or broken Steam setup can never take the game down with it.

### `steam_appid.txt` for development

When you run your game **from the editor or a raw executable** (not launched by Steam), the Steam client needs a
hint about which app you are. Place a `steam_appid.txt` containing just your AppId next to the executable during
development:

```
480
```

Remove it from shipping builds — the live Steam client provides the AppId when it launches your game.

### Restarting through Steam

A Steam build has to be **launched by Steam**, or the overlay, Cloud, DRM and the achievement toasts are all
missing. A player who makes a desktop shortcut straight to the `.exe` would otherwise get a silently degraded game.

**The shipped game handles this for you.** Before the engine starts — before the window, before any script — the
runtime runs each enabled plugin's early-boot hook. This plugin's hook resolves your AppId and calls
`SteamAPI_RestartAppIfNecessary`; if Steam has to relaunch the game, the process exits immediately and Steam
starts it again properly. There is nothing to call and nothing to wire up.

It deliberately does **not** run in the editor: relaunching through Steam there would kill your editor session.

Three ways to turn it off, in order of precedence:

| Escape hatch | Effect |
|--------------|--------|
| `steam_appid.txt` next to the executable | Steam treats the process as already launched, so the check passes through. This is the development workflow. |
| `ICEBOX_STEAM_NO_RESTART=1` in the environment | Skips the check for that run. |
| `"RestartIfNecessary": false` in `steam_config.json` | Skips the check permanently for this build — use it for a DRM-free build of the same game that must run standalone. |

```json
{
    "AppId": 480,
    "RestartIfNecessary": true
}
```

> **Using Valve's DRM wrapper?** Then the check is redundant — the wrapper already guarantees a proper launch.
> Leaving it on is harmless (it returns "no action needed"), but you may set `"RestartIfNecessary": false`.

> **What these switches are for, and what they are not.** All three exist for two legitimate jobs: running your
> own game from the editor or a raw executable while you develop it, and shipping a DRM-free build of your own
> game to a store that is not Steam. They are not a way to run somebody else's game without owning it — none of
> them defeats Valve's DRM wrapper, and a Steam build that is missing its licence simply fails to initialise.
> `steam_appid.txt` in particular belongs beside your build on your machine and nowhere else: it must not be in
> a package you upload to Steam, and it must not be in a plugin package you hand to another developer.

[`Storefront.RestartAppIfNecessary(appId)`](#185-storefrontrestartappifnecessary) is still exposed for the rare
case where you want to drive the check yourself, but calling it from a level script is far too late to be the
primary mechanism — Steam has already been initialized by then.

### What ships with your game, and what must not

The editor's **Tools → Build Game** stages your project's `Plugins/` folder into the game, then overlays the
compiled plugin binaries on top. A few things are worth knowing:

- **`Config/Plugins.json` decides what actually loads.** A plugin that is present but not listed as enabled there
  is discovered and ignored. Enable **IceBoxStorefront** once in the editor's Plugins panel — that writes the file,
  and the build copies it. Without it your shipped game has the plugin on disk and no Steam integration at all.
- **`steam_appid.txt` must not ship.** It tells Steam "assume this AppId, no launch check needed", which is exactly
  what you want on your machine and exactly what you do not want in a release. Keep it next to the built
  executable during testing, and delete it from the package you upload.
- **The Steam runtime library travels with the plugin.** `steam_api64.dll` / `libsteam_api.so` /
  `libsteam_api.dylib` is deployed into the plugin folder next to `IceBoxStorefront.dll`/`.so`/`.dylib` on every
  desktop platform, which is the only location all three dynamic loaders agree on. Do not move it.
- **Everything else in the plugin folder ships too**, including `Documentation/`. Nothing breaks if you leave it,
  but it is roughly 400 KB of markdown you probably do not want in a release — delete it from the staged output
  (or from your project's `Plugins/IceBoxStorefront/`) if package size matters. `LICENSE.txt`, `NOTICE.md` and
  `THIRD_PARTY_NOTICES.txt` stay: the last one is what satisfies the attribution sol2, Lua, nlohmann/json and fmt
  require, and it is a few kilobytes.

<a id="shipping-the-plugin-itself"></a>
### Passing the *plugin* on is a different question — and the answer is: don't

Everything above is about **your game**, going to **players**. Handing the plugin folder to another developer —
publishing it on itch.io, GitHub, a marketplace, an asset pack, or a zip in a chat — is a separate situation, and
the answer there is short: **no**.

| | Goes out with your **game** | Goes out to another **developer** |
|---|---|---|
| `IceBoxStorefront.dll` / `.so` / `.dylib` (the plugin) | ✅ required | 🚫 no — send them the download link |
| `plugin.json`, `VisualScriptAPI.json`, `icon.png` | ✅ required | 🚫 no |
| `LICENSE.txt`, `NOTICE.md`, `THIRD_PARTY_NOTICES.txt` | ✅ keep them in the folder | — |
| `Documentation/`, `README.md` | optional | — |
| `steam_api64.dll` / `libsteam_api.so` / `libsteam_api.dylib` | ✅ yes — Valve permits this | 🚫 **never** |
| `steam_config.json` | ✅ yes — your game reads its AppId from it | 🚫 never — it holds *your* AppId |
| `steam_appid.txt` | 🚫 never — delete it before you upload | 🚫 never |

Two separate reasons sit behind that column.

**The runtime library is Valve's.** Section 1.1 of the *Steamworks SDK Access Agreement* lets you distribute the
contents of the SDK's `redistributable_bin` folder **together with your own application** — that is what makes it
legitimate inside your released game. That permission is **nontransferable**: it does not travel to anyone you
hand a copy to. They get the SDK from Valve themselves, which every Steam developer does anyway.

**The plugin is ours.** It costs nothing and it is not open source. Republishing it — another site, a
marketplace, an asset pack, a modified engine distribution, with or without money — is not permitted; Section 4.1
of `LICENSE.txt` is the operative text. Inside your game it travels freely, and that is Section 2.2. Anyone who
wants the plugin gets it from the official download page in a minute, for free, in the build that matches their
engine version — which is also the only way they get one that actually loads.

`DISTRIBUTION.md` in the plugin root has the full list of what belongs in the build you upload and what must
never.

---

## 3. Core Concepts

### The `Storefront` table

Everything is reachable through one global table:

```lua
Storefront.UnlockAchievement("ACH_WIN_ONE_GAME")   -- top-level function
Storefront.Workshop.GetSubscribed()                 -- Workshop sub-table
Storefront.Input.GetControllers()                   -- Steam Input sub-table
Storefront.Result.Ok                                -- enum constant
```

- **Top-level functions** cover the backend-agnostic surface plus Steam extensions.
- **`Storefront.Workshop.*`** groups all Steam Workshop (UGC) functions.
- **`Storefront.Input.*`** groups all Steam Input (controller / Steam Deck) functions.
- **`Storefront.<EnumName>`** tables hold integer enum constants (e.g. `Storefront.Result`, `Storefront.LobbyType`).

### The frame tick — `Storefront.Tick()`

> **You do not have to call anything to keep the API alive.** The plugin pumps the native Steam callbacks **and**
> flushes the queued Lua callbacks once per frame from its own engine update, for as long as the runtime is running.

The results of async calls (`FindOrCreateLeaderboard`, `CreateLobby`, `Workshop.CreateItem`, …) and event handlers
(`OnP2PMessage`, `OnAchievementUnlocked`, …) are **queued** on the native side and delivered to Lua on the main
thread, in the engine's plugin-update step, before your level and entity `OnUpdate` run that frame.

`Storefront.Tick()` remains available as an **explicit, optional** flush — call it when you want the queue drained
at a specific point of your own frame (for example right before you read `Storefront.ReceiveP2P()` in a networked
loop), or in code that was written against an earlier version of this plugin:

```lua
-- Optional. In a level script (.icemap) or a long-lived manager entity:
function OnUpdate(dt)
    Storefront.Tick()
end
```

`Storefront.Tick()` is cheap, reentrant-safe, idempotent, and a no-op when no backend is active or when the queue
is already empty, so it is always safe to call — and equally safe to leave out.

### Result codes

State-changing functions return an **integer result code** from the `Storefront.Result` enum. `0` (`Ok`) means
success; positive `1` (`Pending`) means "accepted, still in progress"; **negative values are errors**.

```lua
local r = Storefront.UnlockAchievement("ACH_WIN_ONE_GAME")
if r == Storefront.Result.Ok then
    Print("Unlocked!")
else
    Print("Failed: " .. Storefront.ResultName(r))   -- e.g. "NotLoggedIn"
end
```

Use [`Storefront.ResultName(code)`](#412-storefrontresultname) to turn a code into a human-readable string for
logging. See the [full table of result codes](#result-enum).

### Synchronous vs. asynchronous calls

- **Synchronous** functions return immediately — a value, a table, or a `Result` code. Examples: `UnlockAchievement`,
  `GetFriends`, `CloudWrite`, `SetLobbyData`.
- **Asynchronous** functions take a **callback function** as their last argument and return nothing. The callback
  runs later (on the frame the result arrives, during the plugin's update or a manual `Storefront.Tick()`),
  receiving a `Result` code plus any data. Examples:
  `FindOrCreateLeaderboard`, `DownloadLeaderboardEntries`, `CreateLobby`, `JoinLobby`, `SearchLobbies`,
  `RequestNumberOfCurrentPlayers`, `RequestAuthSessionTicket`, and the whole `Workshop.*` set.

```lua
-- Asynchronous: result arrives in the callback, on a later frame.
Storefront.FindOrCreateLeaderboard("HighScores",
    Storefront.LeaderboardSort.Descending,
    Storefront.LeaderboardDisplay.Numeric,
    function(result, handle)
        if result == Storefront.Result.Ok then
            Print("Leaderboard handle: " .. handle)
        end
    end)
```

Callbacks are always invoked on the main thread, from the plugin's per-frame flush (or from a manual
`Storefront.Tick()`). It is safe to call other `Storefront` functions from within a callback.

### The `UserHandle` table

Steam users (the local player, friends, lobby members, leaderboard entry owners, P2P senders) are represented by a
small **`UserHandle`** table:

| Field | Type | Description |
|-------|------|-------------|
| `backend` | `int` | Backend id — `1` for Steam (see [`Storefront.Backend`](#backend-enum)). |
| `id` | `int` | The 64-bit SteamID as an integer. |
| `textId` | `string` | The same SteamID64 as a decimal string (safe for display / storage). |

Wherever a function needs to identify a user (e.g. `SendP2P`, `ActivateOverlayToUser`, `GetFriendAvatar`), pass a
table of this shape. You normally obtain these tables from `GetLocalUser()`, `GetFriends()`, lobby members, etc.

> **Tip:** Prefer `textId` when saving a SteamID to disk or sending it over the network — large 64-bit integers can
> lose precision in some contexts, but the string form never does.

### Binary data as Lua strings

Anywhere the API deals with **raw bytes** — Steam Cloud files, P2P payloads, lobby chat, voice data, auth tickets,
raw RGBA pixels — the bytes are passed to and from Lua as **Lua strings**. Lua strings are binary-safe (they may
contain `\0`), so this is lossless. You can also **pass** a byte array (a Lua table of integers `0–255`) where the
API accepts data to send/write; returned data is always a string.

```lua
-- Write arbitrary bytes to the cloud, read them back:
Storefront.CloudWrite("save.dat", "any\0binary\0bytes")
local data, r = Storefront.CloudRead("save.dat")
if r == Storefront.Result.Ok then
    Print("Read " .. #data .. " bytes")
end
```

### Backend availability and Steam-only calls

Every function on `Storefront` — without exception — is dispatched through the plugin's backend interface, never
through Steam directly. What differs is how much of that interface a given backend implements:

- The **portable core** (achievements, stats, leaderboards, cloud, friends, rich presence, overlay, lobbies, P2P,
  DLC) is required of every backend. If no backend is active at all, these return a safe default
  (`NotInitialized`, `false`, `nil`, or an empty table).
- The **capability groups** (Workshop, Input, Timeline, voice, auth tickets, device info, gamepad text, app info,
  avatars, screenshots) are optional. A backend that does not implement one returns the same safe defaults, or
  `Unsupported` where a result code is returned. Nothing throws and nothing crashes.

Ask before you branch, rather than assuming Steam:

```lua
if Storefront.Supports(Storefront.Capability.Workshop) then
    ShowModBrowser()
end
```

[`Storefront.Supports`](#414-storefrontsupports) is the supported way to find out what the active backend can do.
Today Steam is the only backend the plugin ships, and it reports every capability as available except
`Timeline`, which additionally depends on the Steam client being new enough.

Always gate your integration on availability and login:

```lua
function OnCreate()
    if not Storefront.IsAvailable() then return end   -- not a Steam build / no AppId
    if not Storefront.IsLoggedIn()  then return end   -- Steam client not logged in
    Print("Hello, " .. Storefront.GetPersonaName())
end
```

---

## 4. Session & Identity

Read-only information about the current Steam session and local user. All of these are synchronous.

### 4.1 Storefront.IsAvailable

```lua
Storefront.IsAvailable() -> bool
```

Returns `true` if a storefront backend is active (the plugin initialized successfully with a valid AppId and
a logged-in user). On non-Steam platforms, or when no AppId is configured, returns `false`. This is the first check
every integration should make.

```lua
if not Storefront.IsAvailable() then
    Print("Running without Steam — skipping Steam features")
    return
end
```

---

### 4.2 Storefront.GetBackend

```lua
Storefront.GetBackend() -> string
```

Returns the active backend name: `"Steam"` when Steam is active, otherwise `"None"`.

```lua
Print("Storefront backend: " .. Storefront.GetBackend())
```

---

### 4.3 Storefront.GetBackendId

```lua
Storefront.GetBackendId() -> int
```

Returns the active backend as an integer (see [`Storefront.Backend`](#backend-enum)): `1` for Steam, `0` for none.
The enum reserves further identifiers for backends the plugin may gain later; only `Steam` ships today.

---

### 4.4 Storefront.IsInitialized

```lua
Storefront.IsInitialized() -> bool
```

Returns `true` if the active backend reports it is initialized. In practice this tracks the same state as
`IsAvailable()` for the Steam backend.

---

### 4.5 Storefront.IsLoggedIn

```lua
Storefront.IsLoggedIn() -> bool
```

Returns `true` if the local user is logged on to Steam. Almost every feature requires this — most functions return
`NotInitialized` when it is `false`.

```lua
if Storefront.IsAvailable() and Storefront.IsLoggedIn() then
    -- safe to use achievements, cloud, friends, etc.
end
```

---

### 4.6 Storefront.GetAppId

```lua
Storefront.GetAppId() -> int
```

Returns the Steam AppId the plugin initialized with (`0` if no backend is active).

```lua
Print("AppId: " .. Storefront.GetAppId())
```

---

### 4.7 Storefront.GetLocalUser

```lua
Storefront.GetLocalUser() -> table | nil
```

Returns the local player's [`UserHandle`](#the-userhandle-table) table, or `nil` if not logged in.

```lua
local me = Storefront.GetLocalUser()
if me then
    Print("My SteamID: " .. me.textId)
end
```

---

### 4.8 Storefront.GetPersonaName

```lua
Storefront.GetPersonaName() -> string
```

Returns the local player's Steam display name (persona name). Empty string if not logged in.

```lua
Print("Welcome, " .. Storefront.GetPersonaName())
```

---

### 4.9 Storefront.GetCountryCode

```lua
Storefront.GetCountryCode() -> string
```

Returns the two-letter ISO country code reported by Steam for the user's IP (e.g. `"US"`, `"DE"`, `"UA"`). Empty
string if unavailable. Useful for region-specific defaults — **not** a substitute for a real geo-IP/legal check.

---

### 4.10 Storefront.GetLanguage

```lua
Storefront.GetLanguage() -> string
```

Returns the language the game is set to run in for this user on Steam (the API language string, e.g. `"english"`,
`"russian"`, `"german"`). This reflects the per-game language chosen in the Steam client. Empty string if
unavailable.

```lua
local lang = Storefront.GetLanguage()
if lang == "russian" then
    SetLanguage("ru")
end
```

---

### 4.11 Storefront.GetSteamUILanguage

```lua
Storefront.GetSteamUILanguage() -> string
```

Returns the language of the **Steam client UI itself** (which may differ from the per-game language). Steam-specific.

---

### 4.12 Storefront.ResultName

```lua
Storefront.ResultName(code) -> string
```

Converts a [`Result`](#result-enum) integer code into its name (e.g. `-2` → `"NotLoggedIn"`). Use it for logging and
debugging — never branch on the *string*; compare the numeric code against `Storefront.Result.*` constants instead.

| Parameter | Type | Description |
|-----------|------|-------------|
| `code` | `int` | A result code returned by another function |

```lua
local r = Storefront.CloudWrite("save.dat", data)
if r ~= Storefront.Result.Ok then
    Print("Cloud write failed: " .. Storefront.ResultName(r))
end
```

---

### 4.13 Storefront.GetProductId

```lua
Storefront.GetProductId() -> string
```

Returns the active backend's product identifier as a **string** — the same value as
[`Storefront.GetAppId()`](#46-storefrontgetappid) on Steam, where the identifier is numeric. Prefer this over
`GetAppId()` in code that only needs to log or transmit the identifier: backends other than Steam identify a
product with a non-numeric id, and this function keeps such code portable. Returns an empty string when no
backend is active.

```lua
Print("Product: " .. Storefront.GetProductId())
```

---

### 4.14 Storefront.Supports

```lua
Storefront.Supports(capability) -> bool
```

Reports whether the active backend implements a [capability group](#capability-enum). Returns `false` when no
backend is active, so a single check covers both "no storefront at all" and "this storefront cannot do that".

| Parameter | Type | Description |
|-----------|------|-------------|
| `capability` | `int` | A [`Storefront.Capability`](#capability-enum) constant |

```lua
if Storefront.Supports(Storefront.Capability.Voice) then
    EnableVoiceChatUI()
end

if not Storefront.Supports(Storefront.Capability.Timeline) then
    Print("Game Recording markers are unavailable on this client")
end
```

Calling into an unsupported group is still safe — it returns the group's neutral default. `Supports` exists so you
can hide UI and skip work instead of calling and discarding the result.

---

### 4.15 Storefront.CapabilityName

```lua
Storefront.CapabilityName(capability) -> string
```

Converts a [`Storefront.Capability`](#capability-enum) constant into its name (e.g. `12` → `"Workshop"`). For
logging and debugging only — branch on the numeric constants, never on the string.

| Parameter | Type | Description |
|-----------|------|-------------|
| `capability` | `int` | A [`Storefront.Capability`](#capability-enum) constant |

```lua
for _, cap in ipairs({ Storefront.Capability.Workshop, Storefront.Capability.Input }) do
    Print(Storefront.CapabilityName(cap) .. ": " .. tostring(Storefront.Supports(cap)))
end
```

---

## 5. Achievements

Achievements are identified by their **API Name** (the developer-facing id you set in the Steamworks partner site,
e.g. `ACH_WIN_ONE_GAME`), not their display name.

Achievement metadata is cached when the plugin initializes and refreshed when Steam delivers updated stats. Reads
(`GetAchievement`, `GetAllAchievements`) are served from that cache and are synchronous; writes go to Steam and are
persisted immediately.

### 5.1 Storefront.UnlockAchievement

```lua
Storefront.UnlockAchievement(id) -> int
```

Unlocks an achievement and immediately persists it to Steam (which shows the unlock toast). Idempotent — unlocking
an already-unlocked achievement is harmless.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |

**Returns:** a [`Result`](#result-enum) code — `Ok` on success.

```lua
if enemiesKilled >= 100 then
    Storefront.UnlockAchievement("ACH_KILL_100")
end
```

---

### 5.2 Storefront.ClearAchievement

```lua
Storefront.ClearAchievement(id) -> int
```

Re-locks (clears) an achievement and persists the change. Mostly for development and testing.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |

**Returns:** a [`Result`](#result-enum) code.

---

### 5.3 Storefront.IndicateAchievementProgress

```lua
Storefront.IndicateAchievementProgress(id, current, max) -> int
```

Shows Steam's **progress notification** toast (e.g. "37 / 100") for a progress-based achievement. This does **not**
unlock the achievement — when the underlying stat reaches its goal, Steam unlocks it. Use this purely to surface
progress to the player at meaningful milestones.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |
| `current` | `int` | Current progress value |
| `max` | `int` | Target progress value |

**Returns:** a [`Result`](#result-enum) code.

```lua
-- Every 10 kills, show progress toward the "100 kills" achievement:
if kills % 10 == 0 and kills < 100 then
    Storefront.IndicateAchievementProgress("ACH_KILL_100", kills, 100)
end
```

---

### 5.4 Storefront.GetAchievement

```lua
Storefront.GetAchievement(id) -> table | nil
```

Returns metadata about one achievement, or `nil` if the id is unknown.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |

**Returns:** an [`Achievement` table](#achievement-table):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | API name |
| `displayName` | `string` | Localized display name |
| `description` | `string` | Localized description |
| `hidden` | `bool` | `true` if the achievement is hidden until unlocked |
| `unlocked` | `bool` | Whether the local user has it |
| `progress` | `float` | Current progress (`0` if not a tracked progress achievement here) |
| `progressMax` | `float` | Maximum progress value (`0` if not a progress achievement) |
| `globalUnlockPercent` | `float` | Global unlock percentage across all players, or `-1` if not yet available |
| `unlockTime` | `int` | Unix timestamp (seconds) of the unlock, or `0` if locked |
| `iconLocked` | `string` | Reserved (empty) — use [`GetAchievementIcon`](#162-storefrontgetachievementicon) |
| `iconUnlocked` | `string` | Reserved (empty) — use [`GetAchievementIcon`](#162-storefrontgetachievementicon) |

```lua
local a = Storefront.GetAchievement("ACH_WIN_ONE_GAME")
if a then
    Print(a.displayName .. " — " .. (a.unlocked and "unlocked" or "locked"))
    if a.globalUnlockPercent >= 0 then
        Print(string.format("%.1f%% of players have this", a.globalUnlockPercent))
    end
end
```

> `globalUnlockPercent` is `-1` until Steam delivers global stats. The plugin requests them on init; you can also
> call [`RequestGlobalAchievementPercentages`](#164-storefrontrequestglobalachievementpercentages) and read the
> value a few frames later.

---

### 5.5 Storefront.GetAllAchievements

```lua
Storefront.GetAllAchievements() -> table
```

Returns a 1-indexed array of [`Achievement` tables](#achievement-table) for every achievement defined for the app.
Order is unspecified. Returns an empty table if not logged in or before the cache is populated.

```lua
for _, a in ipairs(Storefront.GetAllAchievements()) do
    Print(a.id .. ": " .. (a.unlocked and "✔" or "✘"))
end
```

---

## 6. Stats

Steam **stats** are named numeric values stored per-user (and optionally aggregated globally). Integer and float
stats are distinct types — read a stat with the same type you defined on the partner site.

> **Persistence:** `SetStatInt` / `SetStatFloat` change the value **locally**. Call
> [`StoreStats`](#65-storefrontstorestats) to upload them to Steam. (Unlocking/clearing an achievement and
> `ResetAllStats` flush stats for you, but explicit stat writes do not.)

### 6.1 Storefront.SetStatInt

```lua
Storefront.SetStatInt(id, value) -> int
```

Sets an integer stat locally.

> **Steam integer stats are 32-bit.** Lua hands you 64-bit integers, and anything outside the signed 32-bit range
> is truncated on the way to Steam without an error. For a value that can grow past ~2.1 billion (lifetime damage,
> total currency earned), use [`SetStatFloat`](#62-storefrontsetstatfloat) or split it across two stats.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Stat API name |
| `value` | `int` | New value |

**Returns:** a [`Result`](#result-enum) code.

---

### 6.2 Storefront.SetStatFloat

```lua
Storefront.SetStatFloat(id, value) -> int
```

Sets a floating-point stat locally.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Stat API name |
| `value` | `number` | New value |

**Returns:** a [`Result`](#result-enum) code.

---

### 6.3 Storefront.GetStatInt

```lua
Storefront.GetStatInt(id) -> (int | nil, int)
```

Reads an integer stat. Returns **two values**: the stat value (or `nil` on failure) and a [`Result`](#result-enum)
code.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Stat API name |

```lua
local value, r = Storefront.GetStatInt("total_kills")
if r == Storefront.Result.Ok then
    Print("Kills: " .. value)
end
```

---

### 6.4 Storefront.GetStatFloat

```lua
Storefront.GetStatFloat(id) -> (number | nil, int)
```

Reads a float stat. Returns the value (or `nil`) and a [`Result`](#result-enum) code.

```lua
local dist, r = Storefront.GetStatFloat("distance_traveled")
if r == Storefront.Result.Ok then
    Print(string.format("Travelled %.1f m", dist))
end
```

---

### 6.5 Storefront.StoreStats

```lua
Storefront.StoreStats() -> int
```

Uploads all locally-changed stats (and any newly-unlocked achievements) to Steam. Call this after a batch of
`SetStat*` calls, e.g. at the end of a level or run.

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.SetStatInt("total_kills", totalKills)
Storefront.SetStatFloat("distance_traveled", distance)
Storefront.StoreStats()   -- persist both
```

---

### 6.6 Storefront.ResetAllStats

```lua
Storefront.ResetAllStats(alsoAchievements) -> int
```

Resets **all** stats to their default values. If `alsoAchievements` is `true`, also clears every achievement. This
persists immediately and rebuilds the achievement cache. Intended for development/testing.

| Parameter | Type | Description |
|-----------|------|-------------|
| `alsoAchievements` | `bool` | Also clear all achievements |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.ResetAllStats(true)   -- wipe stats AND achievements (dev only!)
```

---

### 6.7 Storefront.UpdateAvgRateStat

```lua
Storefront.UpdateAvgRateStat(id, countThisSession, sessionLength) -> int
```

Updates an **average-rate** ("AVGRATE") stat — Steam's helper for values like "average points per hour". You feed it
the count accumulated this session and the session length (in the same time unit, e.g. seconds or hours), and Steam
maintains the sliding average. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | AVGRATE stat API name |
| `countThisSession` | `number` | Amount accumulated during this session |
| `sessionLength` | `number` | Length of this session (consistent time unit) |

**Returns:** a [`Result`](#result-enum) code. Call `StoreStats()` afterward to persist.

```lua
-- Player earned `points` over `dtHours` hours of play:
Storefront.UpdateAvgRateStat("AvgPointsPerHour", points, dtHours)
Storefront.StoreStats()
```

---

### 6.8 Storefront.RequestNumberOfCurrentPlayers

```lua
Storefront.RequestNumberOfCurrentPlayers(callback)
```

Asynchronously asks Steam how many users are currently playing the app right now. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `callback` | `function(result, count)` | `result` is a [`Result`](#result-enum) code; `count` is the live player count |

```lua
Storefront.RequestNumberOfCurrentPlayers(function(r, count)
    if r == Storefront.Result.Ok then
        Print(count .. " players online now")
    end
end)
```

---

### 6.9 Storefront.AreStatsReady

```lua
Storefront.AreStatsReady() -> bool
```

Returns `true` once Steam has delivered this user's stats and achievements to the running process. Steam-specific.

This matters because **stats and achievements are not available the instant Steam initializes** — the client
fetches them from its backend and hands them over a moment later, typically within the first frames. Anything you
write before then is rejected: `SetStatInt`, `SetStatFloat` and `UnlockAchievement` return
[`BackendError`](#result-enum), and `GetAllAchievements()` returns an empty table. The plugin does not retry for
you, so an achievement unlocked in the very first `OnCreate` of the very first level can be lost silently.

Guard early writes, or simply defer them:

```lua
local pendingUnlocks = {}

function Unlock(id)
    if Storefront.IsAvailable() and Storefront.AreStatsReady() then
        Storefront.UnlockAchievement(id)
    else
        pendingUnlocks[#pendingUnlocks + 1] = id
    end
end

function OnUpdate(dt)
    if #pendingUnlocks > 0 and Storefront.IsAvailable() and Storefront.AreStatsReady() then
        for _, id in ipairs(pendingUnlocks) do Storefront.UnlockAchievement(id) end
        pendingUnlocks = {}
    end
end
```

Unlocking at the end of a run, on a level transition, or from any menu is already far past this window — the guard
only matters for writes on the first frames of the session.

---

## 7. Leaderboards

Leaderboards are referenced by an opaque **handle** (an integer). First resolve the handle with
`FindOrCreateLeaderboard`, then upload scores and download entries against it. All leaderboard functions are
asynchronous.

### 7.1 Storefront.FindOrCreateLeaderboard

```lua
Storefront.FindOrCreateLeaderboard(name, sort, display, callback)
```

Finds a leaderboard by name (creating it if it does not yet exist and your app settings allow it), then delivers its
handle to the callback.

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Leaderboard name (as configured on the partner site) |
| `sort` | `int` | A [`LeaderboardSort`](#leaderboardsort-enum) value (`Ascending` / `Descending`) |
| `display` | `int` | A [`LeaderboardDisplay`](#leaderboarddisplay-enum) value (`Numeric` / `TimeSeconds` / `TimeMilliSeconds`) |
| `callback` | `function(result, handle)` | `handle` is the leaderboard handle (integer); `0` on failure |

```lua
local boardHandle = nil
Storefront.FindOrCreateLeaderboard("HighScores",
    Storefront.LeaderboardSort.Descending,
    Storefront.LeaderboardDisplay.Numeric,
    function(r, handle)
        if r == Storefront.Result.Ok then
            boardHandle = handle
        end
    end)
```

> **Tip:** Resolve the handle once (e.g. on level start) and cache it; reuse it for all subsequent uploads and
> downloads.

---

### 7.2 Storefront.UploadLeaderboardScore

```lua
Storefront.UploadLeaderboardScore(handle, score, details, method, callback)
```

Uploads a score to a leaderboard.

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `int` | Leaderboard handle from `FindOrCreateLeaderboard` |
| `score` | `int` | The score to submit. Steam stores leaderboard scores as **signed 32-bit** integers, so the value must lie between `-2147483648` and `2147483647`. A score outside that range is rejected with `InvalidArgument` and a log line, rather than silently wrapping around — scale or clamp it in your game first. |
| `details` | `table` | Optional 1-indexed array of integers stored alongside the score (e.g. metadata); pass `{}` for none |
| `method` | `int` | A [`LeaderboardUpload`](#leaderboardupload-enum) value — `KeepBest` (only replace if better) or `ForceUpdate` (always replace) |
| `callback` | `function(result)` | Delivers the [`Result`](#result-enum) code |

```lua
Storefront.UploadLeaderboardScore(boardHandle, finalScore, {},
    Storefront.LeaderboardUpload.KeepBest,
    function(r)
        Print("Upload: " .. Storefront.ResultName(r))
    end)
```

> **Details array:** Steam stores up to **64** integer details per entry. They are returned verbatim when you
> download entries — handy for storing things like a replay seed, character id, or split times.

---

### 7.3 Storefront.DownloadLeaderboardEntries

```lua
Storefront.DownloadLeaderboardEntries(handle, range, rangeStart, rangeEnd, callback)
```

Downloads a set of leaderboard entries.

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `int` | Leaderboard handle |
| `range` | `int` | A [`LeaderboardRange`](#leaderboardrange-enum) value — `Global`, `AroundUser`, or `Friends` |
| `rangeStart` | `int` | Start index (meaning depends on `range`, see below) |
| `rangeEnd` | `int` | End index |
| `callback` | `function(result, entries)` | `entries` is an array of [`LeaderboardEntry` tables](#leaderboardentry-table) |

**Range semantics:**

- **`Global`** — `rangeStart`/`rangeEnd` are absolute ranks (1-based). `(1, 10)` = top 10.
- **`AroundUser`** — relative to the local user's rank. `(-4, 5)` = 4 above through 5 below (10 entries centered on
  the player).
- **`Friends`** — only the user's friends; the range is ignored by Steam (pass `(0, 0)`).

Each [`LeaderboardEntry`](#leaderboardentry-table) contains:

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | The entry owner's [`UserHandle`](#the-userhandle-table) |
| `displayName` | `string` | The owner's Steam persona name |
| `rank` | `int` | Global rank |
| `score` | `int` | The score |
| `details` | `table` | 1-indexed array of the integer details that were uploaded |

```lua
-- Top 10:
Storefront.DownloadLeaderboardEntries(boardHandle,
    Storefront.LeaderboardRange.Global, 1, 10,
    function(r, entries)
        if r ~= Storefront.Result.Ok then return end
        for _, e in ipairs(entries) do
            Print(e.rank .. ". " .. e.displayName .. " — " .. e.score)
        end
    end)

-- Entries around the player:
Storefront.DownloadLeaderboardEntries(boardHandle,
    Storefront.LeaderboardRange.AroundUser, -4, 5,
    function(r, entries) --[[ ... ]] end)
```

---

## 8. Steam Cloud

Steam Cloud (Remote Storage) stores per-user files that sync across the player's machines. Files are addressed by
name; contents are raw bytes (Lua strings). All cloud functions are synchronous.

> **Enabling cloud:** Steam Cloud must be enabled for your app on the partner site **and** by the user. Check
> [`CloudIsEnabled`](#86-storefrontcloudisenabled) before relying on it. Writes still succeed locally and sync when
> cloud is enabled.

### 8.1 Storefront.CloudWrite

```lua
Storefront.CloudWrite(name, data) -> int
```

Writes a file to the cloud, overwriting any existing file with the same name.

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | File name (e.g. `"slot1.sav"`) |
| `data` | `string` \| `table` | Raw bytes — a binary-safe Lua string, or a 1-indexed array of integers `0–255` |

**Returns:** a [`Result`](#result-enum) code. `QuotaExceeded` if the file is larger than ~2 GB.

```lua
Storefront.CloudWrite("slot1.sav", SaveStateToString())
```

---

### 8.2 Storefront.CloudRead

```lua
Storefront.CloudRead(name) -> (string | nil, int)
```

Reads a cloud file. Returns the file contents as a binary-safe string (or `nil`) and a [`Result`](#result-enum) code
(`NotFound` if the file does not exist).

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | File name |

```lua
local data, r = Storefront.CloudRead("slot1.sav")
if r == Storefront.Result.Ok then
    LoadStateFromString(data)
elseif r == Storefront.Result.NotFound then
    Print("No cloud save yet")
end
```

---

### 8.3 Storefront.CloudDelete

```lua
Storefront.CloudDelete(name) -> int
```

Deletes a cloud file. Returns `NotFound` if it does not exist.

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | File name |

**Returns:** a [`Result`](#result-enum) code.

---

### 8.4 Storefront.CloudList

```lua
Storefront.CloudList() -> table
```

Returns a 1-indexed array of [`CloudFile` tables](#cloudfile-table) describing every file currently in the cloud for
this user/app.

Each [`CloudFile`](#cloudfile-table):

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | File name |
| `size` | `int` | Size in bytes |
| `timestamp` | `int` | Last-modified Unix timestamp (seconds) |

```lua
for _, f in ipairs(Storefront.CloudList()) do
    Print(f.name .. " — " .. f.size .. " bytes")
end
```

---

### 8.5 Storefront.CloudQuotaTotal / CloudQuotaAvailable

```lua
Storefront.CloudQuotaTotal()     -> int   -- total bytes granted
Storefront.CloudQuotaAvailable() -> int   -- bytes still free
```

Return the user's total and remaining cloud storage quota, in bytes.

```lua
local used = Storefront.CloudQuotaTotal() - Storefront.CloudQuotaAvailable()
Print(string.format("Cloud: %d / %d KB used",
    used // 1024, Storefront.CloudQuotaTotal() // 1024))
```

---

### 8.6 Storefront.CloudIsEnabled

```lua
Storefront.CloudIsEnabled() -> bool
```

Returns `true` only if cloud is enabled **both** for the app (partner settings) and for the account (user setting).

```lua
if Storefront.CloudIsEnabled() then
    Storefront.CloudWrite("slot1.sav", data)
else
    SaveLocally(data)   -- fall back to local disk
end
```

---

## 9. Friends, Rich Presence & Overlay

### 9.1 Storefront.GetFriends

```lua
Storefront.GetFriends() -> table
```

Returns a 1-indexed array of [`Friend` tables](#friend-table) for the local user's immediate friends.

Each [`Friend`](#friend-table):

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | The friend's [`UserHandle`](#the-userhandle-table) |
| `personaName` | `string` | Steam display name |
| `statusText` | `string` | The friend's rich-presence `status` string (empty if none) |
| `online` | `bool` | `true` if not offline |
| `playingOurGame` | `bool` | `true` if currently playing *this* app |
| `inLobby` | `bool` | `true` if in a lobby for this app |
| `lobby` | `int` | The friend's lobby id (`0` if none) — pass to `JoinLobby` |

```lua
for _, f in ipairs(Storefront.GetFriends()) do
    if f.playingOurGame then
        Print(f.personaName .. " is playing now — " .. f.statusText)
    end
end
```

---

### 9.2 Storefront.SetRichPresence

```lua
Storefront.SetRichPresence(presence) -> int
```

Publishes rich-presence key/values for the local user. Friends see your `statusText` next to your name, and the
`connectString` powers Steam's **"Join Game"** button.

`presence` is a table with these optional fields:

| Field | Type | Description |
|-------|------|-------------|
| `statusText` | `string` | Human-readable status (stored under Steam's `status` key) |
| `connectString` | `string` | A launch/connect string for "Join Game" (stored under Steam's `connect` key) |
| `tokens` | `table` | Optional extra `string → string` key/value pairs (custom rich-presence keys) |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.SetRichPresence({
    statusText    = "In the Dark Forest (Wave 7)",
    connectString = "+connect_lobby " .. myLobbyId,
    tokens        = { level = "DarkForest", mode = "coop" }
})
```

> **Steam limits:** up to 20 keys; keys ≤ 64 bytes; values ≤ 256 bytes. For a localized status, set `status` to a
> token like `#StatusInGame` defined in your rich-presence localization file on the partner site.

---

### 9.3 Storefront.ClearRichPresence

```lua
Storefront.ClearRichPresence() -> int
```

Clears all rich-presence keys for the local user.

**Returns:** a [`Result`](#result-enum) code.

---

### 9.4 Storefront.ActivateOverlay

```lua
Storefront.ActivateOverlay(dialog) -> int
```

Opens the Steam overlay to a named dialog.

| Parameter | Type | Description |
|-----------|------|-------------|
| `dialog` | `string` | One of: `"Friends"`, `"Community"`, `"Players"`, `"Settings"`, `"OfficialGameGroup"`, `"Stats"`, `"Achievements"`. Empty string defaults to `"Friends"`. |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.ActivateOverlay("Achievements")
```

> The overlay only appears if Steam's in-game overlay is enabled and the game was launched through Steam.

---

### 9.5 Storefront.ActivateOverlayToUser

```lua
Storefront.ActivateOverlayToUser(dialog, user) -> int
```

Opens the overlay to a dialog targeting a specific user.

| Parameter | Type | Description |
|-----------|------|-------------|
| `dialog` | `string` | One of: `"steamid"` (profile), `"chat"`, `"jointrade"`, `"stats"`, `"achievements"`, `"friendadd"`, `"friendremove"`, `"friendrequestaccept"`, `"friendrequestignore"`. Empty defaults to `"steamid"`. |
| `user` | `table` | Target [`UserHandle`](#the-userhandle-table) |

**Returns:** a [`Result`](#result-enum) code.

```lua
local friend = Storefront.GetFriends()[1]
if friend then
    Storefront.ActivateOverlayToUser("steamid", friend.user)
end
```

---

### 9.6 Storefront.ActivateOverlayToWebPage

```lua
Storefront.ActivateOverlayToWebPage(url) -> int
```

Opens a URL in the Steam overlay's web browser.

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | `string` | Absolute URL (`https://…`) |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.ActivateOverlayToWebPage("https://yourgame.com/patch-notes")
```

---

### 9.7 Storefront.ActivateOverlayToStore

```lua
Storefront.ActivateOverlayToStore(appId) -> int
```

Opens an app's store page in the overlay. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `appId` | `string` | The target app's id as a string. Empty / `"0"` opens **this** game's store page. |

**Returns:** a [`Result`](#result-enum) code.

---

### 9.8 Storefront.GetFriendAvatar

```lua
Storefront.GetFriendAvatar(user, size) -> int
```

Returns a Steam **image handle** for a user's avatar, or `0` if not available yet. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | Target [`UserHandle`](#the-userhandle-table) |
| `size` | `int` | `0` = small (32×32), `1` = medium (64×64), `2` = large (184×184) |

Convert the handle to pixels with [`GetImageRGBA`](#165-storefrontgetimagergba), or use the one-step
[`GetFriendAvatarRGBA`](#99-storefrontgetfriendavatarrgba).

```lua
local me = Storefront.GetLocalUser()
local handle = Storefront.GetFriendAvatar(me, 1)
```

---

### 9.9 Storefront.GetFriendAvatarRGBA

```lua
Storefront.GetFriendAvatarRGBA(user, size) -> (string | nil, int, int)
```

Convenience wrapper that returns the avatar **pixels** directly. Returns `(rgba, width, height)` where `rgba` is a
binary string of `width × height × 4` bytes (RGBA8), or `(nil, 0, 0)` if the avatar is not loaded yet. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | Target [`UserHandle`](#the-userhandle-table) |
| `size` | `int` | `0` small, `1` medium, `2` large |

```lua
local rgba, w, h = Storefront.GetFriendAvatarRGBA(Storefront.GetLocalUser(), 2)
if rgba then
    -- upload `rgba` (w×h RGBA8) to a texture for display
end
```

> Avatars load asynchronously. If you get `nil`, try again on a later frame (Steam fetches the image in the
> background after the first request).

---

## 10. Lobbies (Matchmaking)

Lobbies are Steam's matchmaking primitive — a shared, server-listed room with an owner, members, key/value
metadata, and a chat channel. Lobbies are identified by an integer **lobby id**.

Creating, joining, and searching are asynchronous; everything else is synchronous. To receive chat and
member-change events, register [`OnLobbyChat`](#192-storefrontonlobbychat) and
[`OnLobbyMemberChanged`](#193-storefrontonlobbymemberchanged).

### 10.1 Storefront.CreateLobby

```lua
Storefront.CreateLobby(type, maxMembers, callback)
```

Creates a new lobby owned by the local user.

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | `int` | A [`LobbyType`](#lobbytype-enum) value (`Private` / `FriendsOnly` / `Public` / `Invisible`) |
| `maxMembers` | `int` | Maximum members (including the owner) |
| `callback` | `function(result, lobby)` | `lobby` is a [`LobbyInfo` table](#lobbyinfo-table) |

```lua
Storefront.CreateLobby(Storefront.LobbyType.FriendsOnly, 4, function(r, lobby)
    if r == Storefront.Result.Ok then
        myLobbyId = lobby.id
        Storefront.SetLobbyData(lobby.id, "map", "DarkForest")
    end
end)
```

---

### 10.2 Storefront.JoinLobby

```lua
Storefront.JoinLobby(id, callback)
```

Joins an existing lobby by id. Get ids from `SearchLobbies`, a friend's `lobby` field, or an invite/connect string.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `int` | Lobby id |
| `callback` | `function(result, lobby)` | `lobby` is the joined [`LobbyInfo` table](#lobbyinfo-table) |

```lua
Storefront.JoinLobby(targetLobbyId, function(r, lobby)
    if r == Storefront.Result.Ok then
        Print("Joined lobby of " .. lobby.memberCount .. " players")
    end
end)
```

---

### 10.3 Storefront.LeaveLobby

```lua
Storefront.LeaveLobby(id) -> int
```

Leaves a lobby the local user is in.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `int` | Lobby id |

**Returns:** a [`Result`](#result-enum) code.

---

### 10.4 Storefront.SearchLobbies

```lua
Storefront.SearchLobbies(filter, callback)
```

Searches the public lobby list, optionally filtered.

`filter` is a table with these optional fields:

| Field | Type | Description |
|-------|------|-------------|
| `stringEquals` | `table` | `string → string` map; only lobbies whose metadata matches all entries |
| `intEquals` | `table` | `string → int` map; numeric metadata equality filters |
| `minSlotsAvailable` | `int` | Only lobbies with at least this many free slots |
| `resultLimit` | `int` | Cap the number of results returned |

| Parameter | Type | Description |
|-----------|------|-------------|
| `filter` | `table` | Filter table (may be empty `{}` for "all") |
| `callback` | `function(result, lobbies)` | `lobbies` is an array of [`LobbyInfo` tables](#lobbyinfo-table) |

```lua
Storefront.SearchLobbies({
    stringEquals      = { map = "DarkForest" },
    minSlotsAvailable = 1,
    resultLimit       = 20,
}, function(r, lobbies)
    if r ~= Storefront.Result.Ok then return end
    for _, l in ipairs(lobbies) do
        Print("Lobby " .. l.id .. " — " .. l.memberCount .. "/" .. l.memberLimit)
    end
end)
```

---

### 10.5 Storefront.SetLobbyData / GetLobbyData

```lua
Storefront.SetLobbyData(id, key, value) -> int
Storefront.GetLobbyData(id, key)        -> string
```

Set or read a lobby's metadata key. Lobby data is replicated to all members and to lobby searches. Only the lobby
**owner** can usefully set data that others should see. `GetLobbyData` returns an empty string if the key is unset.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `int` | Lobby id |
| `key` | `string` | Metadata key |
| `value` | `string` | Metadata value (set only) |

```lua
Storefront.SetLobbyData(myLobbyId, "state", "in_progress")
local map = Storefront.GetLobbyData(myLobbyId, "map")
```

---

### 10.6 Storefront.GetLobbyMembers

```lua
Storefront.GetLobbyMembers(id) -> table
```

Returns a 1-indexed array of [`LobbyMember` tables](#lobbymember-table) currently in the lobby.

Each [`LobbyMember`](#lobbymember-table):

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | Member's [`UserHandle`](#the-userhandle-table) |
| `personaName` | `string` | Steam display name |
| `data` | `table` | `string → string` map of that member's per-member lobby data |

```lua
for _, m in ipairs(Storefront.GetLobbyMembers(myLobbyId)) do
    Print(m.personaName)
end
```

---

### 10.7 Storefront.SendLobbyChat

```lua
Storefront.SendLobbyChat(id, data) -> int
```

Broadcasts a chat message to all lobby members. Deliver received messages via
[`OnLobbyChat`](#192-storefrontonlobbychat).

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `int` | Lobby id |
| `data` | `string` \| `table` | Payload bytes (max **4096** bytes); a string or array of `0–255` |

**Returns:** a [`Result`](#result-enum) code (`InvalidArgument` if over 4096 bytes).

```lua
Storefront.SendLobbyChat(myLobbyId, "ready")
```

---

### 10.8 Storefront.InviteFriendToLobby

```lua
Storefront.InviteFriendToLobby(lobby, user) -> int
```

Sends a friend a Steam invite to join the given lobby.

| Parameter | Type | Description |
|-----------|------|-------------|
| `lobby` | `int` | Lobby id |
| `user` | `table` | The friend's [`UserHandle`](#the-userhandle-table) |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.InviteFriendToLobby(myLobbyId, friend.user)
```

---

### 10.9 Storefront.ActivateOverlayInviteDialog

```lua
Storefront.ActivateOverlayInviteDialog(lobby) -> int
```

Opens the Steam overlay's **invite friends** dialog targeting a lobby — the simplest way to let players invite
others. Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `lobby` | `int` | Lobby id |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.ActivateOverlayInviteDialog(myLobbyId)
```

---

### 10.10 Storefront.ActivateOverlayInviteDialogConnectString

```lua
Storefront.ActivateOverlayInviteDialogConnectString(connectString) -> int
```

Opens the overlay invite dialog using a **connect string** instead of a lobby (for games that join via a custom
launch/connect string rather than a Steam lobby). Steam-specific.

| Parameter | Type | Description |
|-----------|------|-------------|
| `connectString` | `string` | The connect string friends will use to join |

**Returns:** a [`Result`](#result-enum) code.

---

### 10.11 Storefront.ActivateRemotePlayTogetherInvite

```lua
Storefront.ActivateRemotePlayTogetherInvite() -> int
```

Opens the **Remote Play Together** invite dialog, letting friends join your local-multiplayer session by streaming.
Steam-specific.

**Returns:** a [`Result`](#result-enum) code.

---

## 11. P2P Networking

Peer-to-peer messaging is built on Steam's relay-backed networking (`SteamNetworkingMessages`), so it works through
NATs and firewalls without you running a server. Messages are addressed to a user, tagged with a **channel**, and
sent with a **send type** (reliability). Incoming sessions are **accepted automatically**.

There are two ways to receive messages — pick one:

- **Polling:** call [`ReceiveP2P`](#112-storefrontreceivep2p) each frame to pull queued messages.
- **Callback:** register [`OnP2PMessage`](#191-storefrontonp2pmessage). **If a handler is registered, messages are
  delivered to it and are *not* queued for polling.**

Only channels `0–3` (the [`P2PChannel`](#p2pchannel-enum) values) are received.

### 11.1 Storefront.SendP2P

```lua
Storefront.SendP2P(user, data, channel, sendType) -> int
```

Sends a message to another user.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | Target [`UserHandle`](#the-userhandle-table) |
| `data` | `string` \| `table` | Payload bytes — a string or array of `0–255` |
| `channel` | `int` | A [`P2PChannel`](#p2pchannel-enum) value (`Default` / `GameState` / `Voice` / `FileTransfer`) |
| `sendType` | `int` | A [`P2PSendType`](#p2psendtype-enum) value (reliability — see below) |

**Returns:** a [`Result`](#result-enum) code — `Ok`, `NetworkFailure` (no connection), or `BackendError`.

**Send types:**

- **`Unreliable`** — may be dropped or arrive out of order; lowest latency. Good for frequent state snapshots.
- **`UnreliableNoDelay`** — unreliable and never buffered/Nagle-delayed; lowest possible latency.
- **`Reliable`** — guaranteed, ordered delivery (like TCP). Good for commands, chat, important events.
- **`ReliableWithBuffering`** — reliable, but allows the network layer to coalesce small sends for throughput.

```lua
local payload = string.pack("<fff", x, y, angle)   -- pack a small state snapshot
Storefront.SendP2P(peer, payload,
    Storefront.P2PChannel.GameState,
    Storefront.P2PSendType.Unreliable)
```

---

### 11.2 Storefront.ReceiveP2P

```lua
Storefront.ReceiveP2P() -> table | nil
```

Pops the next queued incoming message, or `nil` if the queue is empty. Drain it in a loop each frame. **Has no
effect if you registered `OnP2PMessage`** (those messages go to the callback instead).

**Returns:** a [`P2PMessage` table](#p2pmessage-table):

| Field | Type | Description |
|-------|------|-------------|
| `sender` | `table` | Sender's [`UserHandle`](#the-userhandle-table) |
| `channel` | `int` | The [`P2PChannel`](#p2pchannel-enum) the message arrived on |
| `payload` | `string` | The raw bytes |

```lua
function OnUpdate(dt)
    Storefront.Tick()
    while true do
        local msg = Storefront.ReceiveP2P()
        if not msg then break end
        HandlePacket(msg.sender, msg.channel, msg.payload)
    end
end
```

> The incoming queue is bounded (the oldest messages are dropped beyond ~4096 pending). Drain it every frame.

---

### 11.3 Storefront.CloseP2PSession

```lua
Storefront.CloseP2PSession(user) -> int
```

Tears down the P2P session with a user, freeing its resources. Call it when a peer disconnects or leaves.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | The peer's [`UserHandle`](#the-userhandle-table) |

**Returns:** a [`Result`](#result-enum) code (`NotFound` if there was no session).

---

## 12. Voice Chat

Capture the local microphone through Steam's voice system (which handles capture and a built-in codec), transmit the
compressed bytes yourself (e.g. over [P2P](#11-p2p-networking)), and decode received bytes to PCM for playback. All
voice functions are Steam-specific.

### 12.1 Storefront.StartVoiceRecording / StopVoiceRecording

```lua
Storefront.StartVoiceRecording() -> bool
Storefront.StopVoiceRecording()  -> bool
```

Begin / end microphone capture. Return `true` if the call was issued. While recording, poll
[`GetVoice`](#122-storefrontgetvoice) each frame to retrieve compressed audio.

```lua
function OnPushToTalkDown() Storefront.StartVoiceRecording() end
function OnPushToTalkUp()   Storefront.StopVoiceRecording()  end
```

---

### 12.2 Storefront.GetVoice

```lua
Storefront.GetVoice() -> (int, string | nil)
```

Pulls the next chunk of **compressed** captured voice. Returns a [`Result`](#result-enum) code and the compressed
bytes (or `nil` when there is nothing available yet). Poll it every frame while recording.

- `Ok` with a string → you have compressed audio to send.
- `Ok` with `nil` → no data available right now (recording but silent).
- `Pending` → capture is initializing; try again next frame.

```lua
function OnUpdate(dt)
    Storefront.Tick()
    local r, voice = Storefront.GetVoice()
    if r == Storefront.Result.Ok and voice then
        -- broadcast compressed voice to peers on the Voice channel
        for _, peer in ipairs(peers) do
            Storefront.SendP2P(peer, voice,
                Storefront.P2PChannel.Voice,
                Storefront.P2PSendType.UnreliableNoDelay)
        end
    end
end
```

---

### 12.3 Storefront.DecompressVoice

```lua
Storefront.DecompressVoice(compressed) -> (int, string | nil, int)
```

Decodes compressed voice bytes (received from a peer) into raw **16-bit signed PCM** (mono). Returns a
[`Result`](#result-enum) code, the PCM bytes (or `nil`), and the **sample rate** in Hz.

| Parameter | Type | Description |
|-----------|------|-------------|
| `compressed` | `string` | Compressed voice bytes received from a peer's `GetVoice` |

```lua
-- When a Voice-channel P2P message arrives:
local r, pcm, sampleRate = Storefront.DecompressVoice(msg.payload)
if r == Storefront.Result.Ok and pcm then
    -- `pcm` is 16-bit signed mono at `sampleRate` Hz — feed it to your audio playback
    PlayPCM(pcm, sampleRate)
end
```

> PCM format: signed 16-bit little-endian, mono. Each sample is 2 bytes, so `#pcm / 2` samples.

---

## 13. DLC & Store

Query and manage Steam DLC owned by the player, and open store pages. The generic DLC functions work on any
storefront; `InstallDLC` / `UninstallDLC` are Steam-specific.

> DLC is identified by its **AppId**, passed as a **string** (e.g. `"431960"`).

### 13.1 Storefront.GetDLCs

```lua
Storefront.GetDLCs() -> table
```

Returns a 1-indexed array of [`DLC` tables](#dlc-table) for every DLC registered for the app.

Each [`DLC`](#dlc-table):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | DLC AppId as a string |
| `name` | `string` | DLC name (from Steam) |
| `appId` | `int` | DLC AppId as an integer |
| `owned` | `bool` | Whether the user owns/subscribes to the DLC |
| `installed` | `bool` | Whether the DLC is installed locally |
| `bytesDownloaded` | `int` | Download progress (bytes) if downloading |
| `bytesTotal` | `int` | Total download size (bytes) if downloading |

```lua
for _, dlc in ipairs(Storefront.GetDLCs()) do
    Print(dlc.name .. ": " .. (dlc.owned and "owned" or "not owned"))
end
```

---

### 13.2 Storefront.IsDLCInstalled

```lua
Storefront.IsDLCInstalled(id) -> bool
```

Returns `true` if the given DLC (by AppId string) is installed.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | DLC AppId as a string |

```lua
if Storefront.IsDLCInstalled("431960") then
    EnableBonusContent()
end
```

---

### 13.3 Storefront.InstallDLC / UninstallDLC

```lua
Storefront.InstallDLC(id)   -> int
Storefront.UninstallDLC(id) -> int
```

Asks Steam to install or uninstall an owned DLC. Steam-specific. The actual download happens in the background; poll
[`GetDLCs`](#131-storefrontgetdlcs) for progress, or listen for completion in your own logic.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | DLC AppId as a string |

**Returns:** a [`Result`](#result-enum) code (`InvalidArgument` if the id is not a valid number).

```lua
Storefront.InstallDLC("431960")
```

---

### 13.4 Storefront.RequestPurchase

```lua
Storefront.RequestPurchase(productId) -> int
```

Opens the Steam **store page** for a product in the overlay so the player can buy it. `productId` is an app/DLC id;
the plugin opens `https://store.steampowered.com/app/<productId>` in the overlay browser.

| Parameter | Type | Description |
|-----------|------|-------------|
| `productId` | `string` | App/DLC id |

**Returns:** a [`Result`](#result-enum) code.

```lua
Storefront.RequestPurchase("431960")   -- prompt the player to buy this DLC
```

---

### 13.5 Storefront.OpenStorePage

```lua
Storefront.OpenStorePage(productId) -> int
```

Opens a product's store page in the overlay. If `productId` is a valid app id it uses the native store overlay;
otherwise it falls back to the web store page (same as `RequestPurchase`).

| Parameter | Type | Description |
|-----------|------|-------------|
| `productId` | `string` | App/DLC id |

**Returns:** a [`Result`](#result-enum) code.

---

## 14. Workshop (UGC)

The Steam Workshop lets players publish, browse, subscribe to, and download user-generated content (mods, maps,
skins). All Workshop functions live under **`Storefront.Workshop.*`** and are Steam-specific.

Items are identified by a 64-bit **item id** (the published file id). Subscribed items are downloaded by Steam into
a per-item folder you can locate with [`GetInstallInfo`](#144-storefrontworkshopgetinstallinfo).

> **Lifecycle of a published item:** `CreateItem` → set fields & upload via the `args` table → Steam assigns an item
> id → later `UpdateItem` to push changes. Players use `Subscribe` / `Download` / `Unsubscribe`.

### 14.1 Storefront.Workshop.GetSubscribed

```lua
Storefront.Workshop.GetSubscribed(includeDisabled) -> table
```

`includeDisabled` is optional and defaults to `false`; pass `true` to also get items the player has disabled in
Steam's own UI (see [`SetItemsDisabledLocally`](#1415-storefrontworkshopsetitemsdisabledlocally)).

> Only **local** state is filled in here — `installed`, `installFolder`, `needsUpdate`, `disabledLocally` and so
> on. `title`, `description`, `tags` and `metadata` come from the Steam backend: fetch them with
> [`QueryItemsByIds`](#1411-storefrontworkshopqueryitemsbyids).

Returns a 1-indexed array of [`WorkshopItem` tables](#workshopitem-table) for every item the user is subscribed to,
with local state (installed / downloading / needs-update) filled in.

```lua
for _, item in ipairs(Storefront.Workshop.GetSubscribed()) do
    if item.installed then
        MountMod(item.installFolder)
    end
end
```

---

### 14.2 Storefront.Workshop.GetItemInfo

```lua
Storefront.Workshop.GetItemInfo(itemId) -> table | nil
```

Returns a [`WorkshopItem` table](#workshopitem-table) with the **local** state for one item (subscription, install,
download progress, install folder), or `nil` if Workshop is unavailable. Note that fields describing the published
listing (title, description, votes, tags) are only filled in by [`QueryUserItems`](#146-storefrontworkshopqueryuseritems)
— `GetItemInfo` returns the locally-known state.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |

```lua
local info = Storefront.Workshop.GetItemInfo(itemId)
if info and info.needsUpdate then
    Storefront.Workshop.Download(itemId, true, function(r) end)
end
```

---

### 14.3 Storefront.Workshop.GetItemState

```lua
Storefront.Workshop.GetItemState(itemId) -> int
```

Returns Steam's raw item-state **bitmask** for an item. For most uses prefer the decoded boolean fields on the
[`WorkshopItem` table](#workshopitem-table) (`subscribed`, `installed`, `needsUpdate`, `downloading`) from
`GetItemInfo`/`GetSubscribed`. The bitmask bits are: `1` = subscribed, `2` = legacy, `4` = installed, `8` = needs
update, `16` = downloading, `32` = download pending.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |

```lua
local state = Storefront.Workshop.GetItemState(itemId)
local installed = (state & 4) ~= 0
```

---

### 14.4 Storefront.Workshop.GetInstallInfo

```lua
Storefront.Workshop.GetInstallInfo(itemId) -> table | nil
```

Returns where an installed item lives on disk, or `nil` if it is not installed.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |

**Returns:** a table:

| Field | Type | Description |
|-------|------|-------------|
| `folder` | `string` | Absolute path to the item's content folder |
| `size` | `int` | Total size on disk (bytes) |

```lua
local install = Storefront.Workshop.GetInstallInfo(itemId)
if install then
    LoadModFrom(install.folder)
end
```

---

### 14.5 Storefront.Workshop.Subscribe / Unsubscribe

```lua
Storefront.Workshop.Subscribe(itemId, callback)
Storefront.Workshop.Unsubscribe(itemId, callback)
```

Subscribe or unsubscribe the local user to an item. Subscribing queues a download (Steam manages it). Asynchronous.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |
| `callback` | `function(result)` | Delivers the [`Result`](#result-enum) code |

```lua
Storefront.Workshop.Subscribe(itemId, function(r)
    Print("Subscribe: " .. Storefront.ResultName(r))
end)
```

---

### 14.6 Storefront.Workshop.QueryUserItems

```lua
Storefront.Workshop.QueryUserItems(page, callback)
```

Queries the items **published by the local user**, one page at a time (Steam returns up to 50 per page). Results
include full listing metadata (title, description, votes, tags, preview URL). Asynchronous.

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | `int` | 1-based page number (`0` is treated as page 1) |
| `callback` | `function(result, items)` | `items` is an array of [`WorkshopItem` tables](#workshopitem-table) |

```lua
Storefront.Workshop.QueryUserItems(1, function(r, items)
    if r ~= Storefront.Result.Ok then return end
    for _, it in ipairs(items) do
        Print(it.title .. " — 👍 " .. it.votesUp .. " 👎 " .. it.votesDown)
    end
end)
```

---

### 14.7 Storefront.Workshop.CreateItem

```lua
Storefront.Workshop.CreateItem(args, callback)
```

Creates a **new** Workshop item and uploads its initial content in one step. Steam assigns a new item id, then the
plugin applies your fields and submits the upload.

`args` is a table with these fields (all optional except as noted):

| Field | Type | Description |
|-------|------|-------------|
| `title` | `string` | Item title |
| `description` | `string` | Item description |
| `contentFolder` | `string` | Absolute path to the folder whose contents become the item payload |
| `previewImagePath` | `string` | Absolute path to a preview image (JPG/PNG, ≤ 1 MB) |
| `visibility` | `int` | A [`WorkshopVisibility`](#workshopvisibility-enum) value (defaults to `Public`; omit the field to leave Steam's own default alone on an update) |
| `tags` | `table` | 1-indexed array of tag strings |
| `metadata` | `string` | Developer metadata, up to 10 000 characters — your own machine-readable blob (mod format version, entry script, required engine API), invisible to players and returned by [`QueryItemsByIds`](#1411-storefrontworkshopqueryitemsbyids) |
| `changeNote` | `string` | Change note for the initial upload (defaults to `"Initial upload"`) |

| Parameter | Type | Description |
|-----------|------|-------------|
| `args` | `table` | The fields above |
| `callback` | `function(result, itemId, needsLegalAgreement)` | On success, `itemId` is the new item's id; `needsLegalAgreement` is `true` when the item stays hidden until the author accepts the Workshop agreement |

```lua
Storefront.Workshop.CreateItem({
    title            = "My Custom Map",
    description      = "A spooky forest level.",
    contentFolder    = ModExportPath() .. "/MyMap",
    previewImagePath = ModExportPath() .. "/MyMap/preview.png",
    visibility       = Storefront.WorkshopVisibility.Public,
    tags             = { "Map", "Singleplayer" },
    metadata         = '{"modFormat":2,"entry":"main.lua"}',
}, function(r, itemId, needsLegalAgreement)
    if r ~= Storefront.Result.Ok then
        Print("Publish failed: " .. Storefront.ResultName(r))
        return
    end
    Print("Published item " .. itemId)
    SetString("my_workshop_item", tostring(itemId))

    if needsLegalAgreement then
        ShowDialog("Accept the Steam Workshop agreement so your item becomes visible.")
        Storefront.Workshop.OpenLegalAgreement()
    end
end)
```

> **Always act on `needsLegalAgreement`.** Until the author accepts it, the item is invisible to everyone —
> including the author — with no error anywhere. See
> [`OpenLegalAgreement`](#1413-storefrontworkshopopenlegalagreement).

> **Tags must already exist in your app's Workshop configuration** on the Steamworks partner site. Steam rejects
> tags it does not know about, and the upload fails with a generic backend error.

---

### 14.8 Storefront.Workshop.UpdateItem

```lua
Storefront.Workshop.UpdateItem(itemId, args, callback)
```

Updates an existing item you own. Only the fields you provide in `args` are changed (empty fields are left as-is).

`args` accepts the same fields as [`CreateItem`](#147-storefrontworkshopcreateitem). Only what you provide is
changed — an omitted or empty field leaves that property as it was, and omitting `visibility` entirely leaves the
item's current visibility alone.

| Field | Type | Description |
|-------|------|-------------|
| `changeNote` | `string` | A patch note describing this update (shown in the item's change history) |

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | The item to update |
| `args` | `table` | Fields to change |
| `callback` | `function(result, itemId, needsLegalAgreement)` | Echoes the item id on success; `needsLegalAgreement` is `true` when the item stays hidden until the author accepts the Workshop agreement |

```lua
Storefront.Workshop.UpdateItem(itemId, {
    description   = "Updated for v1.2 — fixed spawn points.",
    contentFolder = ModExportPath() .. "/MyMap",
    changeNote    = "Bugfix patch",
    metadata      = '{"modFormat":2,"entry":"main.lua"}',
}, function(r, id, needsLegalAgreement)
    Print("Update: " .. Storefront.ResultName(r))
    if needsLegalAgreement then Storefront.Workshop.OpenLegalAgreement() end
end)
```

Poll [`GetUpdateProgress`](#1414-storefrontworkshopgetupdateprogress) while this runs to show a progress bar.

---

### 14.9 Storefront.Workshop.Download

```lua
Storefront.Workshop.Download(itemId, highPriority, callback)
```

Asks Steam to (re)download an item's content immediately, even if not subscribed. Useful to force an update or
pre-fetch. The callback fires when the download has been **requested** (not when it finishes — poll `GetItemInfo`
for completion).

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |
| `highPriority` | `bool` | `true` to suspend other downloads and fetch this first |
| `callback` | `function(result)` | Delivers the [`Result`](#result-enum) code |

```lua
Storefront.Workshop.Download(itemId, true, function(r) end)
```

---

### 14.10 Storefront.Workshop.SetUserItemVote

```lua
Storefront.Workshop.SetUserItemVote(itemId, voteUp, callback)
```

Casts the local user's up/down vote on an item. Asynchronous.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |
| `voteUp` | `bool` | `true` for a thumbs-up, `false` for thumbs-down |
| `callback` | `function(result)` | Delivers the [`Result`](#result-enum) code |

```lua
Storefront.Workshop.SetUserItemVote(itemId, true, function(r) end)
```

---

### 14.11 Storefront.Workshop.QueryItemsByIds

```lua
Storefront.Workshop.QueryItemsByIds(itemIds, callback)
```

Fetches the **full details** — title, description, tags, metadata, vote counts, preview URL — for a specific list
of items. Asynchronous.

This is the companion to [`GetSubscribed`](#141-storefrontworkshopgetsubscribed), which only reports local state
and leaves `title` and `description` empty. Subscribe list first, details second:

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemIds` | `table` | Array of Workshop item ids; **at most 50** per call |
| `callback` | `function(result, items)` | `items` is an array of [Workshop item](#workshopitem-table) tables |

```lua
local subscribed = Storefront.Workshop.GetSubscribed()
local ids = {}
for _, item in ipairs(subscribed) do ids[#ids + 1] = item.itemId end

Storefront.Workshop.QueryItemsByIds(ids, function(r, items)
    if r ~= Storefront.Result.Ok then return end
    for _, item in ipairs(items) do
        Print(item.title .. " by " .. item.owner.textId .. " — " .. item.installFolder)
    end
end)
```

Passing more than 50 ids returns `InvalidArgument` without calling Steam; page your list yourself if a player is
subscribed to more.

---

### 14.12 Storefront.Workshop.SetFavorite

```lua
Storefront.Workshop.SetFavorite(itemId, favorite, callback)
```

Adds the item to, or removes it from, the user's favourites. Asynchronous.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemId` | `int` | Workshop item id |
| `favorite` | `bool` | `true` to add, `false` to remove |
| `callback` | `function(result)` | Delivers the [`Result`](#result-enum) code |

---

### 14.13 Storefront.Workshop.OpenLegalAgreement

```lua
Storefront.Workshop.OpenLegalAgreement() -> int
```

Opens the Steam Workshop legal agreement in the overlay.

**You must call this when a publish reports `needsLegalAgreement`.** Until the author accepts that agreement, the
item they just uploaded exists but is **invisible to everyone, including them** — no error, no warning, it simply
never appears. It is the single most common "my mod uploaded but nobody can see it" support ticket.

```lua
Storefront.Workshop.CreateItem(args, function(r, itemId, needsLegalAgreement)
    if r ~= Storefront.Result.Ok then return end
    if needsLegalAgreement then
        ShowDialog("One more step: accept the Steam Workshop agreement so your item becomes visible.")
        Storefront.Workshop.OpenLegalAgreement()
    end
end)
```

---

### 14.14 Storefront.Workshop.GetUpdateProgress

```lua
Storefront.Workshop.GetUpdateProgress() -> table
```

Progress of the upload currently in flight. Poll it every frame while a `CreateItem` / `UpdateItem` is running to
drive a progress bar — uploading a mod is the one Workshop operation slow enough that silence looks like a hang.

| Field | Type | Description |
|-------|------|-------------|
| `status` | `int` | A [`WorkshopUpdateStatus`](#workshopupdatestatus-enum) value; `Invalid` when nothing is uploading |
| `bytesProcessed` | `int` | Bytes done so far |
| `bytesTotal` | `int` | Total bytes for the current stage; `0` until Steam knows |

```lua
function OnUpdate(dt)
    local p = Storefront.Workshop.GetUpdateProgress()
    if p.status ~= Storefront.WorkshopUpdateStatus.Invalid and p.bytesTotal > 0 then
        SetProgressBar(p.bytesProcessed / p.bytesTotal)
    end
end
```

---

### 14.15 Storefront.Workshop.SetItemsDisabledLocally

```lua
Storefront.Workshop.SetItemsDisabledLocally(itemIds, disabled) -> int
```

Marks subscribed items as locally disabled (or re-enables them) **in Steam's own bookkeeping**. A disabled item
stays subscribed and stays on disk, but drops out of `GetSubscribed()` unless you ask for it, and reports
`disabledLocally = true`.

Use this to back your in-game mod toggles with Steam's state instead of a private list of your own: the player
sees the same on/off in the Steam client, and it survives a reinstall of your game.

| Parameter | Type | Description |
|-----------|------|-------------|
| `itemIds` | `table` | Array of Workshop item ids |
| `disabled` | `bool` | `true` to disable, `false` to re-enable |

```lua
Storefront.Workshop.SetItemsDisabledLocally({ itemId }, true)
local all = Storefront.Workshop.GetSubscribed(true)   -- pass true to still see disabled ones
```

---

### 14.16 Storefront.Workshop.SetSubscriptionsLoadOrder

```lua
Storefront.Workshop.SetSubscriptionsLoadOrder(itemIds) -> int
```

Stores the order in which subscribed items should load, so a reordering the player makes in your mod menu is
remembered by Steam and shown consistently in the Steam client. Pass the **complete** ordered list.

```lua
Storefront.Workshop.SetSubscriptionsLoadOrder({ topMod, middleMod, bottomMod })
```

Note that the engine's own mod loading is ordered by `LoadOrder` in each `mod.json`; this call records the
player's preferred order on Steam's side, and it is up to your mod menu to reconcile the two.

---

### 14.17 Storefront.Workshop.MarkDownloadedItemAsUnused

```lua
Storefront.Workshop.MarkDownloadedItemAsUnused(itemId) -> int
```

Tells Steam your game is done with the item's content, so it may be evicted when disk space is reclaimed. Call it
for items you downloaded on demand and no longer load — never for something the player is subscribed to and
actively using.

---

### 14.18 Turning Workshop items into engine mods

The Workshop API tells you **where** a subscribed item is installed; the engine's mod system is what actually
**runs** it. They meet at one call: `Mods.AddSearchPath()` registers an arbitrary directory as a place to scan for
mods, so a Workshop folder becomes a mod without copying a single file.

Steam installs every subscribed item into its own folder under `steamapps/workshop/content/<appid>/<itemid>/`, and
`item.installFolder` is that path. Give the mod system those folders and rescan:

```lua
function MountWorkshopMods()
    if not Storefront.IsAvailable() then return end

    Mods.ClearSearchPaths()

    for _, item in ipairs(Storefront.Workshop.GetSubscribed()) do
        if item.installed and not item.needsUpdate and item.installFolder ~= "" then
            Mods.AddSearchPath(item.installFolder)
        end
    end

    Mods.Refresh()
end
```

The contract for a mod-shaped Workshop item is simply the engine's own mod contract — the uploaded **content
folder** is the mod folder:

```
MyWorkshopMod/          <- the folder you pass to Workshop.CreateItem as contentFolder
├── mod.json            <- Name, Version, Author, EntryScript, LoadOrder, Dependencies
├── main.lua            <- EntryScript; gets MOD_NAME / MOD_VERSION / MOD_DIR and ModRequire()
└── ...                 <- anything else the mod loads through MOD_DIR
```

Practical notes for a Workshop-driven mod pipeline:

- **Search paths are per session.** They are not saved anywhere; re-register them on every launch, before the
  scan. Call `Mods.ClearSearchPaths()` first so items the player just unsubscribed from drop out.
- **Newly discovered mods start disabled**, exactly like any mod missing from `Config/Mods.json`. Subscribing is
  not the same as enabling — call `Mods.SetEnabled(name, true)` for the ones the player turns on. That loads the
  mod immediately while a scene is running and persists the choice.
- **Names must be unique.** A Workshop item whose `mod.json` `Name` collides with a mod already found under
  `Mods/` is skipped with a warning. Tell your modders to namespace it.
- **Mount before the mods are needed.** `Mods.Refresh()` unloads and reloads everything, so run it from an early
  startup level (a boot or main-menu level), not in the middle of gameplay.
- **Respect item state.** Skip items where `installed` is `false` or `needsUpdate` is `true` — Steam is still
  downloading them. Use [`Storefront.Workshop.Download`](#149-storefrontworkshopdownload) to prioritize one, then
  mount it on a later frame once `GetItemInfo(itemId).installed` flips to `true`.
- **Tag your Workshop items** (`tags` in `CreateItem` / `UpdateItem`) so the Workshop page can filter mod kinds,
  and set matching tags in your Steamworks app configuration first — Steam rejects tags it does not know.

---

## 15. Steam Input & Steam Deck

Steam Input gives you Steam's unified, **action-based** controller API: you define *actions* (like `"Jump"` or
`"Move"`) and *action sets* in a manifest, and Steam maps them to whatever device the player uses — Xbox, PlayStation,
Switch Pro, Steam Controller, or the Steam Deck's built-in controls — including the player's own custom bindings.
You then ask "is the Jump action pressed?" instead of polling raw buttons. All functions are under
**`Storefront.Input.*`** and are Steam-specific.

> **Auto-initialization:** the plugin already calls Steam Input's `Init` on startup and `RunFrame` every tick, so in
> most cases you can skip `Input.Init` / `Input.RunFrame` and go straight to reading controllers and actions. Call
> `Input.Init(manifestPath)` only if you need to point Steam at a specific action manifest at runtime.

**Handles:** controllers, action sets, action-set layers, and actions are all referenced by integer **handles** you
resolve by name. Resolve them once and cache them.

### 15.1 Storefront.Input.Init

```lua
Storefront.Input.Init(manifestPath?) -> bool
```

Ensures Steam Input is initialized, optionally setting the absolute path to your action manifest (a `.vdf` file
describing your actions, action sets, and default bindings). Returns `true` if Steam Input is ready.

| Parameter | Type | Description |
|-----------|------|-------------|
| `manifestPath` | `string` | *(optional)* Absolute path to the action manifest file |

```lua
Storefront.Input.Init("C:/Game/steam_input_manifest.vdf")
```

---

### 15.2 Storefront.Input.Shutdown / RunFrame

```lua
Storefront.Input.Shutdown()
Storefront.Input.RunFrame()
```

`Shutdown` releases Steam Input; `RunFrame` pumps it once. Both are optional — the plugin manages Steam Input's
lifecycle for you. Only call `RunFrame` manually if you have disabled the automatic tick.

---

### 15.3 Storefront.Input.SetActionManifestFilePath

```lua
Storefront.Input.SetActionManifestFilePath(absPath) -> bool
```

Points Steam Input at an action manifest file at runtime. Returns `true` on success.

| Parameter | Type | Description |
|-----------|------|-------------|
| `absPath` | `string` | Absolute path to the manifest |

---

### 15.4 Storefront.Input.GetControllers

```lua
Storefront.Input.GetControllers() -> table
```

Returns a 1-indexed array of [`Controller` tables](#controller-table) for every connected, Steam-Input-handled
controller.

Each [`Controller`](#controller-table):

| Field | Type | Description |
|-------|------|-------------|
| `handle` | `int` | Controller handle (pass to the other `Input.*` functions) |
| `inputType` | `int` | A [`InputType`](#inputtype-enum) value (Xbox360, PS5, SteamDeck, …) |
| `isSteamDeck` | `bool` | `true` if this is the Steam Deck's built-in controller |
| `gamepadIndex` | `int` | The emulated Xbox gamepad index, or `-1` |

```lua
for _, c in ipairs(Storefront.Input.GetControllers()) do
    Print("Controller " .. c.handle .. " type=" .. c.inputType)
end
```

---

### 15.5 Controller / gamepad index mapping

```lua
Storefront.Input.GetControllerForGamepadIndex(index) -> int
Storefront.Input.GetGamepadIndexForController(handle) -> int
Storefront.Input.GetInputTypeForHandle(handle)        -> int
Storefront.Input.IsSteamDeckController(handle)         -> bool
```

Map between Steam Input controller **handles** and the emulated Xbox **gamepad index** the rest of the engine sees,
query a controller's [`InputType`](#inputtype-enum), and test whether a handle is the Steam Deck.

| Function | Returns |
|----------|---------|
| `GetControllerForGamepadIndex(index)` | The controller handle for emulated gamepad slot `index`, or `0` |
| `GetGamepadIndexForController(handle)` | The emulated gamepad index for a handle, or `-1` |
| `GetInputTypeForHandle(handle)` | A [`InputType`](#inputtype-enum) value |
| `IsSteamDeckController(handle)` | `true` if the handle is a Steam Deck controller |

---

### 15.6 Action sets and layers

```lua
Storefront.Input.GetActionSetHandle(name)               -> int
Storefront.Input.ActivateActionSet(handle, actionSet)
Storefront.Input.GetCurrentActionSet(handle)            -> int
Storefront.Input.ActivateActionSetLayer(handle, layer)
Storefront.Input.DeactivateActionSetLayer(handle, layer)
Storefront.Input.DeactivateAllActionSetLayers(handle)
```

An **action set** is a named group of bindings (e.g. `"InGameControls"`, `"MenuControls"`). Switch the active set
when context changes (gameplay ↔ menu ↔ vehicle). **Layers** stack temporary binding overrides on top of a set
(e.g. a `"Scoped"` layer while aiming).

| Function | Description |
|----------|-------------|
| `GetActionSetHandle(name)` | Resolve an action-set handle by its manifest name |
| `ActivateActionSet(handle, actionSet)` | Make `actionSet` the controller's active set |
| `GetCurrentActionSet(handle)` | The controller's currently-active action-set handle |
| `ActivateActionSetLayer(handle, layer)` | Push an action-set layer |
| `DeactivateActionSetLayer(handle, layer)` | Pop a specific layer |
| `DeactivateAllActionSetLayers(handle)` | Pop all layers |

```lua
local inGame = Storefront.Input.GetActionSetHandle("InGameControls")
for _, c in ipairs(Storefront.Input.GetControllers()) do
    Storefront.Input.ActivateActionSet(c.handle, inGame)
end
```

---

### 15.7 Digital actions

```lua
Storefront.Input.GetDigitalActionHandle(name)          -> int
Storefront.Input.GetDigitalActionData(handle, action)  -> table
```

A **digital action** is an on/off action (Jump, Fire, Interact). Resolve the handle once by name, then read its state
each frame for a given controller.

`GetDigitalActionData` returns a table:

| Field | Type | Description |
|-------|------|-------------|
| `state` | `bool` | `true` while the action is pressed |
| `active` | `bool` | `true` if the action is bound/available in the current action set |

```lua
local jump = Storefront.Input.GetDigitalActionHandle("Jump")

function OnUpdate(dt)
    Storefront.Tick()
    for _, c in ipairs(Storefront.Input.GetControllers()) do
        local d = Storefront.Input.GetDigitalActionData(c.handle, jump)
        if d.active and d.state then
            DoJump()
        end
    end
end
```

---

### 15.8 Analog actions

```lua
Storefront.Input.GetAnalogActionHandle(name)          -> int
Storefront.Input.GetAnalogActionData(handle, action)  -> table
Storefront.Input.StopAnalogActionMomentum(handle, action)
```

An **analog action** is a 1- or 2-axis action (Move, Look, Throttle). `GetAnalogActionData` returns:

| Field | Type | Description |
|-------|------|-------------|
| `x` | `float` | Horizontal axis (typically `-1`…`1`) |
| `y` | `float` | Vertical axis (typically `-1`…`1`) |
| `mode` | `int` | The source mode (joystick, trackpad, mouse, …) as reported by Steam |
| `active` | `bool` | Whether the action is currently bound/active |

`StopAnalogActionMomentum` cancels residual trackpad "flick" momentum for an action.

```lua
local move = Storefront.Input.GetAnalogActionHandle("Move")
local m = Storefront.Input.GetAnalogActionData(controller, move)
if m.active then
    MovePlayer(m.x, m.y)
end
```

---

### 15.9 Action origins & glyphs

> Two glyph flavours are available: `GetGlyphSVGForActionOrigin(origin)` returns a **path to an SVG file**, and
> `GetGlyphPNGForActionOrigin(origin, size)` returns a **path to a PNG file** at one of three fixed sizes. Both
> return a filesystem path, not image bytes — load it with your normal texture pipeline. Reach for the PNG unless
> your UI renderer actually rasterizes SVG.
>
> ```lua
> local png = Storefront.Input.GetGlyphPNGForActionOrigin(origin, Storefront.GlyphSize.Medium)
> if png ~= "" then ShowButtonHint(png) end
> ```
>
> `size` is optional and defaults to `Medium`; see [`GlyphSize`](#glyphsize-enum).

```lua
Storefront.Input.GetDigitalActionOrigins(handle, actionSet, action) -> table
Storefront.Input.GetAnalogActionOrigins(handle, actionSet, action)  -> table
Storefront.Input.GetGlyphSVGForActionOrigin(origin)                 -> string
Storefront.Input.GetStringForActionOrigin(origin)                   -> string
```

An **origin** is the concrete physical input (e.g. "A button", "Right trigger") an action is currently bound to on a
specific device. Use origins to draw correct on-screen prompts ("Press Ⓐ to jump") that match the player's actual
controller and bindings.

- `GetDigitalActionOrigins` / `GetAnalogActionOrigins` return a 1-indexed array of integer origin ids for the given
  controller + action set + action.
- `GetGlyphSVGForActionOrigin(origin)` returns an absolute path to an **SVG glyph** file for that origin (for
  rendering the button icon).
- `GetStringForActionOrigin(origin)` returns a human-readable name for the origin (e.g. `"A"`, `"Right Trigger"`).

```lua
local jump = Storefront.Input.GetDigitalActionHandle("Jump")
local set  = Storefront.Input.GetActionSetHandle("InGameControls")
local origins = Storefront.Input.GetDigitalActionOrigins(controller, set, jump)
if #origins > 0 then
    local label = Storefront.Input.GetStringForActionOrigin(origins[1])
    local glyph = Storefront.Input.GetGlyphSVGForActionOrigin(origins[1])
    ShowPrompt("Press " .. label .. " to jump", glyph)
end
```

---

### 15.10 Haptics & LED

```lua
Storefront.Input.TriggerVibration(handle, left, right)
Storefront.Input.TriggerVibrationExtended(handle, left, right, leftTrigger, rightTrigger)
Storefront.Input.SetLEDColor(handle, r, g, b)
```

- `TriggerVibration` — rumble the two main motors. `left`/`right` are intensities `0–65535`.
- `TriggerVibrationExtended` — also drives the trigger motors (DualSense / supported pads).
- `SetLEDColor` — set the controller light bar color (DualShock/DualSense). `r`,`g`,`b` are `0–255`.

```lua
-- Damage feedback:
Storefront.Input.TriggerVibration(controller, 40000, 40000)
Storefront.Input.SetLEDColor(controller, 255, 0, 0)
```

---

### 15.11 Storefront.Input.ShowBindingPanel

```lua
Storefront.Input.ShowBindingPanel(handle) -> bool
```

Opens Steam's controller **configurator** overlay for the given controller, where the player can remap bindings.
Returns `true` if the panel was shown (requires Big Picture / the overlay).

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `int` | Controller handle |

```lua
function OnConfigureControlsButton()
    local c = Storefront.Input.GetControllers()[1]
    if c then Storefront.Input.ShowBindingPanel(c.handle) end
end
```

---

## 16. Platform Utilities

Device, environment, image, and on-screen-keyboard helpers. These are Steam-specific.

### 16.1 Device & environment queries

```lua
Storefront.IsSteamDeck()       -> bool   -- running on a Steam Deck
Storefront.IsSteamMachine()    -> bool   -- running on a Steam Machine
Storefront.IsSteamFrame()      -> bool   -- running on a Steam Frame
Storefront.GetSteamHardware()  -> int    -- Storefront.SteamHardware.* (0 when not Steam hardware)
Storefront.GetHardwareDefaultConfig() -> int  -- Storefront.HardwareConfig.*
Storefront.IsRunningUnderProton()     -> bool  -- running through Proton on Linux
Storefront.IsBigPictureMode()  -> bool   -- Steam Big Picture mode is active
Storefront.IsRunningInVR()     -> bool   -- launched in Steam VR mode
Storefront.GetBatteryPower()   -> int    -- battery % (0–100), or 255 on AC / unknown
```

Use these to adapt UI scale, default to controller input, or show battery warnings on handhelds.

```lua
if Storefront.IsSteamDeck() then
    UseControllerUI()
    UseLargerFonts()
end

local bat = Storefront.GetBatteryPower()
if bat ~= 255 and bat <= 15 then
    ShowToast("Low battery: " .. bat .. "%")
end
```

**Choosing graphics defaults on Steam hardware.** `GetSteamHardware()` identifies the exact device and is meant
for analytics, support, and diagnostics. For an actual *decision* — which graphics preset to start with — use
`GetHardwareDefaultConfig()` instead: Valve maps it per device (and lets you re-map it later from the Steamworks
partner site, without shipping a patch), and it also fires for third-party handhelds with similar performance
characteristics. That is what makes a build survive Steam hardware that did not exist when it shipped.

```lua
-- Run this once, the first time the game starts on a new machine.
local cfg = Storefront.GetHardwareDefaultConfig()
local HC  = Storefront.HardwareConfig

if     cfg == HC.SteamDeck    then ApplyPreset("deck")
elseif cfg == HC.SteamMachine then ApplyPreset("machine")
elseif cfg == HC.SteamFrame   then ApplyPreset("frame")
elseif cfg == HC.Low          then ApplyPreset("low")
elseif cfg == HC.Medium       then ApplyPreset("medium")
elseif cfg == HC.High         then ApplyPreset("high")
elseif cfg == HC.Max          then ApplyPreset("ultra")
else                               ApplyPresetFromOwnHeuristics()
end
```

> **Steam Deck, Steam Machine and Steam Frame.** `Storefront.IsSteamDeck()` still works exactly as before, but it
> is now derived from the device query that replaced the removed `ISteamUtils::IsRunningOnSteamDeck()` in
> Steamworks SDK **v1.65**, so it also knows about Steam Machine and Steam Frame. Prefer capability checks
> (`GetHardwareDefaultConfig()`, `IsBigPictureMode()`, `Storefront.Input.GetControllers()`, `GetBatteryPower()`)
> over device checks whenever the decision is about a *feature* rather than about reporting.

---

### 16.2 Storefront.GetAchievementIcon

```lua
Storefront.GetAchievementIcon(id) -> int
```

Returns a Steam **image handle** for an achievement's icon (the icon for its current locked/unlocked state), or `0`
if not loaded yet. Convert to pixels with [`GetImageRGBA`](#165-storefrontgetimagergba), or use
[`GetAchievementIconRGBA`](#163-storefrontgetachievementiconrgba).

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |

---

### 16.3 Storefront.GetAchievementIconRGBA

```lua
Storefront.GetAchievementIconRGBA(id) -> (string | nil, int, int)
```

Returns an achievement icon as pixels: `(rgba, width, height)` where `rgba` is `width × height × 4` bytes (RGBA8),
or `(nil, 0, 0)` if the icon is not available yet.

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `string` | Achievement API name |

```lua
local rgba, w, h = Storefront.GetAchievementIconRGBA("ACH_WIN_ONE_GAME")
if rgba then UploadIconTexture(rgba, w, h) end
```

---

### 16.4 Storefront.RequestGlobalAchievementPercentages

```lua
Storefront.RequestGlobalAchievementPercentages()
```

Asks Steam to (re)fetch the global unlock percentages for all achievements. After it completes (a few frames later),
the `globalUnlockPercent` field on [`Achievement` tables](#achievement-table) becomes valid. The plugin already
requests these on init; call this to refresh.

---

### 16.5 Storefront.GetImageRGBA

```lua
Storefront.GetImageRGBA(handle) -> (string | nil, int, int)
```

Converts any Steam **image handle** (from `GetAchievementIcon`, `GetFriendAvatar`, etc.) into raw pixels. Returns
`(rgba, width, height)` with `rgba` a binary string of `width × height × 4` bytes (RGBA8), or `(nil, 0, 0)` on
failure.

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `int` | A Steam image handle |

```lua
local handle = Storefront.GetFriendAvatar(Storefront.GetLocalUser(), 2)
local rgba, w, h = Storefront.GetImageRGBA(handle)
```

---

### 16.6 On-screen keyboard (gamepad text input)

```lua
Storefront.ShowGamepadTextInput(description, existingText, maxChars) -> bool
Storefront.GetEnteredGamepadText()                                  -> string | nil
```

`ShowGamepadTextInput` opens Steam's **modal** on-screen keyboard (used in Big Picture and on the Steam Deck). It
returns `true` if the keyboard was shown. When the user confirms, retrieve the result with `GetEnteredGamepadText`
(which returns the entered string, or `nil` if nothing was entered / it was dismissed).

| Parameter | Type | Description |
|-----------|------|-------------|
| `description` | `string` | Prompt shown above the keyboard |
| `existingText` | `string` | Pre-filled text |
| `maxChars` | `int` | Maximum input length |

```lua
if Storefront.ShowGamepadTextInput("Enter your name", currentName, 32) then
    -- later, after the user confirms (poll or check on a UI event):
    local text = Storefront.GetEnteredGamepadText()
    if text then SetPlayerName(text) end
end
```

---

### 16.7 Floating keyboard (Steam Deck)

```lua
Storefront.ShowFloatingGamepadTextInput(mode, x, y, width, height) -> bool
Storefront.DismissFloatingGamepadTextInput()                       -> bool
Storefront.DismissGamepadTextInput()                               -> bool
```

The **floating** keyboard is the non-modal Steam Deck keyboard you position next to a text field yourself.

| Parameter | Type | Description |
|-----------|------|-------------|
| `mode` | `int` | A [`FloatingKeyboardMode`](#floatingkeyboardmode-enum) value (`SingleLine` / `MultipleLines` / `Email` / `Numeric`) |
| `x`, `y` | `int` | Top-left position (screen pixels) of the text field the keyboard should sit beside |
| `width`, `height` | `int` | Size of that text field |

- `ShowFloatingGamepadTextInput` returns `true` if shown.
- `DismissFloatingGamepadTextInput` closes the floating keyboard.
- `DismissGamepadTextInput` closes the **modal** keyboard (from `ShowGamepadTextInput`).

```lua
Storefront.ShowFloatingGamepadTextInput(
    Storefront.FloatingKeyboardMode.SingleLine, fieldX, fieldY, fieldW, fieldH)
```

---

### 16.8 Storefront.TriggerScreenshot

```lua
Storefront.TriggerScreenshot() -> bool
```

Asks Steam to take a screenshot (exactly as if the player pressed the Steam screenshot hotkey). Steam captures,
saves, and adds it to the user's screenshot library. Returns `true` if the request was issued.

```lua
function OnScreenshotButton()
    Storefront.TriggerScreenshot()
end
```

### 16.9 Overlay notification placement

```lua
Storefront.SetOverlayNotificationPosition(position)
Storefront.SetOverlayNotificationInset(horizontal, vertical)
```

Moves Steam's own toast — the achievement pop-up, the "friend came online" notice — to a corner that does not sit
on top of your HUD, and nudges it inward by a pixel margin.

| Parameter | Type | Description |
|-----------|------|-------------|
| `position` | `int` | An [`OverlayNotificationPosition`](#overlaynotificationposition-enum) value |
| `horizontal` | `int` | Inset from the left/right edge, in pixels |
| `vertical` | `int` | Inset from the top/bottom edge, in pixels |

Set it once at startup, before anything can unlock. It is a small thing that shows up in every screenshot and
every review video, so pick the corner your HUD leaves free.

```lua
function OnCreate()
    if not Storefront.IsAvailable() then return end
    Storefront.SetOverlayNotificationPosition(Storefront.OverlayNotificationPosition.TopRight)
    Storefront.SetOverlayNotificationInset(16, 16)
end
```

---

### 16.10 Reporting your graphics settings to Steam

```lua
Storefront.SetGamePerformanceSetting(setting)
Storefront.SetGameRenderResolution(width, height)
```

Tells Steam which preset the game is **currently running at** and at what internal resolution. This is the mirror
image of [`GetHardwareDefaultConfig()`](#161-device--environment-queries): that one asks Steam what to start with,
these two report back what you settled on.

| Parameter | Type | Description |
|-----------|------|-------------|
| `setting` | `int` | A [`PerformanceSetting`](#performancesetting-enum) value |
| `width`, `height` | `int` | Internal render resolution in pixels (not the window size, if you render at a different scale) |

Steam attaches this to the anonymous framerate data players may opt into sharing, and Valve uses it to tune what
`GetHardwareDefaultConfig()` returns for your game on future hardware. Reporting it costs you nothing and makes
your game behave better on Steam devices that do not exist yet.

```lua
function ApplyPreset(name, w, h)
    -- ... apply your own settings ...
    if not Storefront.IsAvailable() then return end
    local PS = Storefront.PerformanceSetting
    local map = { low = PS.Low, medium = PS.Medium, high = PS.High, ultra = PS.Ultra }
    Storefront.SetGamePerformanceSetting(map[name] or PS.Custom)
    Storefront.SetGameRenderResolution(w, h)
end
```

---

## 17. Authentication Tickets

Auth tickets let your backend verify a player's Steam identity and ownership. Both requests are asynchronous and
deliver the ticket bytes to a callback. Steam-specific.

> Tickets are binary blobs returned as Lua strings. Send them to your server, which validates them with Steam's Web
> API. Do not trust client-side claims of identity without a validated ticket.

### 17.1 Storefront.RequestAuthSessionTicket

```lua
Storefront.RequestAuthSessionTicket(callback)
```

Requests a **session ticket** proving the local user's Steam identity (used to authenticate to a game server or
backend).

| Parameter | Type | Description |
|-----------|------|-------------|
| `callback` | `function(result, ticket, handle)` | `ticket` is the ticket bytes (string), or `nil` on failure; `handle` is the ticket handle to pass to [`CancelAuthSessionTicket`](#173-storefrontcancelauthsessionticket) |

```lua
Storefront.RequestAuthSessionTicket(function(r, ticket, handle)
    if r == Storefront.Result.Ok and ticket then
        myTicketHandle = handle
        SendToServerForValidation(ticket)
    end
end)
```

> The callback only fires once **Steam has validated the ticket** with its servers — a ticket handed out earlier
> would be rejected by `BeginAuthSession` on the receiving end. Expect it a few frames (or, on a slow connection,
> a few seconds) after the call. If validation fails the callback still runs, with an error `result` and no ticket.

---

### 17.2 Storefront.RequestEncryptedAppTicket

```lua
Storefront.RequestEncryptedAppTicket(userData, callback)
```

Requests an **encrypted app ticket** — a tamper-proof token (decryptable only with your app's secret key on your
server) carrying ownership info plus your own `userData`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `userData` | `string` \| `table` | Arbitrary bytes to embed in the ticket (a string or array of `0–255`) |
| `callback` | `function(result, ticket)` | `ticket` is the encrypted bytes (string), or `nil` on failure |

```lua
Storefront.RequestEncryptedAppTicket("session:" .. mySessionId, function(r, ticket)
    if r == Storefront.Result.Ok and ticket then
        SendToServer(ticket)
    end
end)
```

> Steam rate-limits encrypted app ticket requests (about once every few seconds) — request one per session and
> cache it.

---

### 17.3 Storefront.CancelAuthSessionTicket

```lua
Storefront.CancelAuthSessionTicket(handle)
```

Tells Steam to invalidate a previously-issued auth session ticket (for example when the player disconnects from your
server). `handle` is the third value the
[`RequestAuthSessionTicket`](#171-storefrontrequestauthsessionticket) callback received.

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `int` | The auth-ticket handle to cancel |

```lua
if myTicketHandle then
    Storefront.CancelAuthSessionTicket(myTicketHandle)
    myTicketHandle = nil
end
```

### 17.4 Storefront.RequestWebApiAuthTicket

```lua
Storefront.RequestWebApiAuthTicket(identity, callback)
```

Asks Steam for a ticket meant for **your own web backend**, which verifies it by calling Steam's
`ISteamUserAuth/AuthenticateUserTicket` web API. Asynchronous.

> **This is the one to use for a server.** The ticket from
> [`RequestAuthSessionTicket`](#171-storefrontrequestauthsessionticket) is for peer-to-peer
> [`BeginAuthSession`](#175-storefrontbeginauthsession) only — Valve's own header states it will *fail* against
> the web API. If you are proving "this player really is this SteamID" to a backend you control, you want this
> function.

| Parameter | Type | Description |
|-----------|------|-------------|
| `identity` | `string` | Optional. A string identifying the service the ticket is for; pass the same value your backend uses when it verifies |
| `callback` | `function(result, ticket, handle)` | `ticket` is a binary string, `handle` cancels it later |

```lua
Storefront.RequestWebApiAuthTicket("my-game-backend", function(r, ticket, handle)
    if r ~= Storefront.Result.Ok then return end
    myTicketHandle = handle
    -- Hex-encode and send to your backend, which calls AuthenticateUserTicket.
    local hex = (ticket:gsub(".", function(c) return string.format("%02X", c:byte()) end))
    SendLoginRequest(hex)
end)
```

Cancel it with [`CancelAuthSessionTicket`](#173-storefrontcancelauthsessionticket) when the session ends.

---

### 17.5 Storefront.BeginAuthSession

```lua
Storefront.BeginAuthSession(ticket, user) -> int
```

Starts verifying a ticket that a **peer sent you** — the host side of a P2P session. Returns an
[`AuthSessionResult`](#authsessionresult-enum) **synchronously**, which only says whether Steam accepted the ticket
for checking. The real verdict arrives later in
[`OnAuthSessionValidated`](#197-storefrontonauthsessionvalidated).

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticket` | `string` | The binary ticket the peer sent, exactly as received |
| `user` | `table` | The [`UserHandle`](#userhandle-table-ref) of the peer who sent it |

```lua
function OnPeerHello(peer, ticket)
    local r = Storefront.BeginAuthSession(ticket, peer.user)
    if r ~= Storefront.AuthSessionResult.Ok then
        Disconnect(peer, "bad ticket")
    end
    -- Now wait for OnAuthSessionValidated before trusting them.
end

Storefront.OnAuthSessionValidated(function(user, owner, response)
    if response ~= Storefront.AuthSessionResponse.Ok then
        Disconnect(FindPeer(user), Storefront.ResultName(response))
        return
    end
    -- `owner` differs from `user` when the game is Family Shared.
    if Storefront.UserHasLicenseForApp(user, Storefront.GetAppId())
            ~= Storefront.LicenseResult.HasLicense then
        Disconnect(FindPeer(user), "does not own the game")
        return
    end
    Admit(FindPeer(user))
end)
```

Always pair it with [`EndAuthSession`](#176-storefrontendauthsession) — Steam keeps tracking the user until you do.

---

### 17.6 Storefront.EndAuthSession

```lua
Storefront.EndAuthSession(user)
```

Stops the tracking started by [`BeginAuthSession`](#175-storefrontbeginauthsession). Call it the moment a peer
leaves, including on an abnormal disconnect — otherwise Steam holds the session open and the peer's next
`BeginAuthSession` comes back as `DuplicateRequest`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | The [`UserHandle`](#userhandle-table-ref) of the peer |

---

### 17.7 Storefront.UserHasLicenseForApp

```lua
Storefront.UserHasLicenseForApp(user, appId) -> int
```

After a successful `BeginAuthSession`, asks whether that user owns a given app or DLC. Returns a
[`LicenseResult`](#licenseresult-enum).

| Parameter | Type | Description |
|-----------|------|-------------|
| `user` | `table` | The [`UserHandle`](#userhandle-table-ref) of an authenticated peer |
| `appId` | `int` | The app or DLC id to check |

Returns `NoAuth` if you never called `BeginAuthSession` for that user, which is the usual reason this looks broken.

---

## 18. App & Apps Information

Read-only information about the app installation, ownership, beta branch, and launch parameters. These are
Steam-specific.

### 18.1 Ownership & subscription

```lua
Storefront.IsSubscribed()                  -> bool   -- user owns/licenses this app
Storefront.IsSubscribedFromFreeWeekend()   -> bool   -- access via a Free Weekend
Storefront.IsSubscribedFromFamilySharing() -> bool   -- access via Family Sharing
Storefront.IsLowViolence()                 -> bool   -- the low-violence build variant
Storefront.IsCybercafe()                   -> bool   -- the account is a cybercafe/internet-cafe account
Storefront.IsVACBanned()                   -> bool   -- the user is VAC-banned for this app
```

```lua
if Storefront.IsSubscribedFromFreeWeekend() then
    ShowBuyPrompt()   -- gently nudge free-weekend players
end
```

---

### 18.2 Build, beta & install

```lua
Storefront.GetBuildId()         -> int      -- the installed build id
Storefront.GetCurrentBetaName() -> string   -- active beta branch name, or "" for default
Storefront.GetAppInstallDir()   -> string   -- absolute install folder of this app
```

```lua
Print("Build " .. Storefront.GetBuildId())
local beta = Storefront.GetCurrentBetaName()
if beta ~= "" then Print("On beta branch: " .. beta) end
```

---

### 18.3 Launch parameters

```lua
Storefront.GetLaunchCommandLine()    -> string          -- full launch command line
Storefront.GetLaunchQueryParam(key)  -> string          -- a single launch query parameter
```

When a player launches the game via a Steam URL (e.g. `steam://run/<appid>//?param=value`) or accepts an invite,
these expose the parameters. Combine with [`OnConnectInvite`](#196-storefrontonconnectinvite) for join-game flows.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Query-parameter name (for `GetLaunchQueryParam`) |

```lua
local connect = Storefront.GetLaunchQueryParam("connect")
if connect ~= "" then
    JoinServer(connect)
end
```

---

### 18.4 Ownership & time

```lua
Storefront.GetAppOwner()             -> table | nil   -- UserHandle of the app's owner
Storefront.GetServerRealTime()       -> int           -- Steam server time (Unix seconds)
Storefront.GetSecondsSinceAppActive() -> int          -- seconds since the app gained focus
```

`GetAppOwner` differs from `GetLocalUser` when the app is being played through **Family Sharing** (the owner is the
lender). `GetServerRealTime` is a trustworthy clock (immune to local clock tampering) for time-gated events.

```lua
local me    = Storefront.GetLocalUser()
local owner = Storefront.GetAppOwner()
if owner and me and owner.id ~= me.id then
    Print("Playing via Family Sharing, owned by " .. owner.textId)
end
```

---

### 18.5 Storefront.RestartAppIfNecessary

```lua
Storefront.RestartAppIfNecessary(appId) -> bool
```

If the game was **not** launched through Steam, asks Steam to relaunch it under the given AppId and returns `true`
(you should then quit). Returns `false` if the game is already running correctly under Steam. This is a **static**
call — safe to use at the very start of the program, before anything else is initialized. See
[Restarting through Steam](#restarting-through-steam).

| Parameter | Type | Description |
|-----------|------|-------------|
| `appId` | `int` | Your Steam AppId |

```lua
if Storefront.RestartAppIfNecessary(480) then
    Quit()
end
```

---

## 19. Events & Callbacks

Some Steam activity is **pushed** to you rather than polled: an incoming P2P message, a lobby chat line, a friend
accepting an invite. Register a handler function for each event you care about. **Handlers fire on the main
thread, during the plugin's [per-frame flush](#the-frame-tick--storefronttick)** — you do not have to tick for them.

Registering a handler replaces any previous one for that event. You can register at any time (the plugin re-installs
handlers when a backend becomes active). Pass `nil`-free functions; to stop receiving, register an empty function.

| Registration | Handler signature |
|--------------|-------------------|
| `Storefront.OnP2PMessage(fn)` | `fn(message)` |
| `Storefront.OnLobbyChat(fn)` | `fn(lobbyId, sender, payload)` |
| `Storefront.OnLobbyMemberChanged(fn)` | `fn(lobbyId, user, joined)` |
| `Storefront.OnAchievementUnlocked(fn)` | `fn(achievement)` |
| `Storefront.OnOverlayActivated(fn)` | `fn(active)` |
| `Storefront.OnConnectInvite(fn)` | `fn(connectString)` |
| `Storefront.SetLogSink(fn)` | `fn(level, message)` |

### 19.1 Storefront.OnP2PMessage

```lua
Storefront.OnP2PMessage(function(message) ... end)
```

Called for each incoming [P2P message](#p2pmessage-table). **While a handler is registered, messages are delivered
here and not queued for [`ReceiveP2P`](#112-storefrontreceivep2p).** `message` is a [`P2PMessage` table](#p2pmessage-table)
(`sender`, `channel`, `payload`).

```lua
Storefront.OnP2PMessage(function(msg)
    HandlePacket(msg.sender, msg.channel, msg.payload)
end)
```

---

### 19.2 Storefront.OnLobbyChat

```lua
Storefront.OnLobbyChat(function(lobbyId, sender, payload) ... end)
```

Called when a lobby chat message is received (sent via [`SendLobbyChat`](#107-storefrontsendlobbychat)).

| Argument | Type | Description |
|----------|------|-------------|
| `lobbyId` | `int` | The lobby the message was sent in |
| `sender` | `table` | Sender's [`UserHandle`](#the-userhandle-table) |
| `payload` | `string` | The message bytes |

```lua
Storefront.OnLobbyChat(function(lobbyId, sender, text)
    Print(sender.textId .. ": " .. text)
end)
```

---

### 19.3 Storefront.OnLobbyMemberChanged

```lua
Storefront.OnLobbyMemberChanged(function(lobbyId, user, joined) ... end)
```

Called when a member joins or leaves a lobby you are in.

| Argument | Type | Description |
|----------|------|-------------|
| `lobbyId` | `int` | The lobby |
| `user` | `table` | The member's [`UserHandle`](#the-userhandle-table) |
| `joined` | `bool` | `true` if they joined, `false` if they left/disconnected/were kicked |

```lua
Storefront.OnLobbyMemberChanged(function(lobbyId, user, joined)
    Print(user.textId .. (joined and " joined" or " left"))
    RefreshLobbyRoster(lobbyId)
end)
```

---

### 19.4 Storefront.OnAchievementUnlocked

```lua
Storefront.OnAchievementUnlocked(function(achievement) ... end)
```

Called whenever an achievement is unlocked (including from `UnlockAchievement` or when Steam unlocks a progress
achievement). `achievement` is an [`Achievement` table](#achievement-table). Handy for your own in-game celebration
UI on top of Steam's toast.

```lua
Storefront.OnAchievementUnlocked(function(a)
    PlaySound("fanfare")
    ShowBanner("Achievement: " .. a.displayName)
end)
```

---

### 19.5 Storefront.OnOverlayActivated

```lua
Storefront.OnOverlayActivated(function(active) ... end)
```

Called when the Steam overlay opens or closes. Pause the game while the overlay is up.

| Argument | Type | Description |
|----------|------|-------------|
| `active` | `bool` | `true` when the overlay opened, `false` when it closed |

```lua
Storefront.OnOverlayActivated(function(active)
    SetPaused(active)
end)
```

---

### 19.6 Storefront.OnConnectInvite

```lua
Storefront.OnConnectInvite(function(connectString) ... end)
```

Called when the player accepts a Steam **invite** or launches the game via a **+connect** parameter / rich-presence
"Join Game". This is the entry point for *join-my-friend's-game* flows. The `connectString` is whatever the inviter
set (e.g. `"+connect_lobby <lobbyId>"` from a lobby invite, or your own custom connect string).

```lua
Storefront.OnConnectInvite(function(connect)
    local lobbyId = Storefront.ParseConnectLobby(connect)
    if lobbyId then
        Storefront.JoinLobby(lobbyId, function(r, lobby) end)
    end
end)
```

#### The two halves of "Join Game"

An invite reaches your game through **one of two paths**, and a game that only implements one of them looks broken
half the time:

| The player clicks Join… | …and the game is | The invite arrives as |
|---|---|---|
| in the friends list or overlay | **already running** | this handler |
| in the friends list or overlay | **not running** | the process command line — Steam launches your game with `+connect_lobby <id>` appended |

The plugin handles both and gives you one shape either way. On startup it parses its own command line (and Steam's
launch parameters) and keeps whatever it found; drain it once with `GetPendingConnect()`:

```lua
Storefront.GetPendingConnect()  -> string | nil   -- returns it once, then forgets it
Storefront.PeekPendingConnect() -> string | nil   -- same value, without consuming
Storefront.ParseConnectLobby(connectString) -> int | nil
```

If **no** `OnConnectInvite` handler is registered when an invite arrives while the game is running, it is stored as
a pending connect instead — so a game that only ever polls still works. Register a handler and the pending slot is
never used for live invites. You never get the same invite twice.

```lua
local function Join(connect)
    local lobbyId = Storefront.ParseConnectLobby(connect)
    if lobbyId then Storefront.JoinLobby(lobbyId, function(r, lobby) end) end
end

function OnCreate()
    if not Storefront.IsAvailable() then return end
    Storefront.OnConnectInvite(Join)          -- warm path: game already running
    local pending = Storefront.GetPendingConnect()
    if pending then Join(pending) end         -- cold path: launched from the invite
end
```

`ParseConnectLobby` pulls the lobby id out of a `+connect_lobby <id>` string and returns `nil` for anything else —
a rich-presence `connect` value of your own design comes through untouched for you to parse.

---

### 19.7 Storefront.OnAuthSessionValidated

```lua
Storefront.OnAuthSessionValidated(function(user, owner, response) ... end)
```

The verdict for a ticket you submitted with [`BeginAuthSession`](#175-storefrontbeginauthsession). Steam-specific.

| Argument | Type | Description |
|----------|------|-------------|
| `user` | `table` | [`UserHandle`](#userhandle-table-ref) of the user whose ticket was checked |
| `owner` | `table` | [`UserHandle`](#userhandle-table-ref) of the account that **owns** the game — different from `user` under Family Sharing |
| `response` | `int` | An [`AuthSessionResponse`](#authsessionresponse-enum) value; anything but `Ok` means reject |

This can fire **again** for a user you already admitted — Steam re-validates when something changes, for example
when the ticket is cancelled or the account logs in elsewhere. Handle it as a state change, not a one-shot: kick a
player whose `response` turns non-`Ok` mid-session.

---

### 19.8 Storefront.SetLogSink

```lua
Storefront.SetLogSink(function(level, message) ... end)
```

Routes the plugin's internal diagnostic log lines to your function — useful for surfacing Steam init/runtime issues
in your own console.

| Argument | Type | Description |
|----------|------|-------------|
| `level` | `int` | `0` = info, `1` = warning, `2` = error |
| `message` | `string` | The log text |

```lua
Storefront.SetLogSink(function(level, msg)
    Print("[Steam:" .. level .. "] " .. msg)
end)
```

> This is **additive**: the same lines always go to the engine log as well, so you never lose Steam diagnostics by
> not installing a sink — and installing one does not silence the engine log. Unlike every other handler, the sink
> is called immediately on the frame the message is produced rather than being queued for `Storefront.Tick()`.

---

## 20. Steam Timeline & Game Recording

Steam Timeline is what turns a recorded session into something a player can navigate: the strip under Steam's
Game Recording timeline gets **coloured by game mode**, **marked with your events**, and **split into phases**.
Everything here is optional decoration on top of a recording the player may or may not be making — the calls are
free no-ops when the feature is unavailable, so you never have to branch on it.

All of it lives under **`Storefront.Timeline`**.

```lua
Storefront.Timeline.IsAvailable() -> bool
```

`true` when the running Steam client exposes the Timeline interface. Everything below is safe to call regardless;
this is only for hiding a "Steam clips" toggle in your own UI.

---

### 20.1 Game mode — colouring the timeline

```lua
Storefront.Timeline.SetGameMode(mode)
```

Tells Steam what kind of activity is on screen right now, which colours that stretch of the recording. Set it on
every state change and leave it alone in between — it is a mode, not an event.

| Parameter | Type | Description |
|-----------|------|-------------|
| `mode` | `int` | A [`TimelineGameMode`](#timelinegamemode-enum) value |

```lua
function OnMainMenuEnter()  Storefront.Timeline.SetGameMode(Storefront.TimelineGameMode.Menus) end
function OnLevelLoadBegin() Storefront.Timeline.SetGameMode(Storefront.TimelineGameMode.LoadingScreen) end
function OnLevelStart()     Storefront.Timeline.SetGameMode(Storefront.TimelineGameMode.Playing) end
function OnInventoryOpen()  Storefront.Timeline.SetGameMode(Storefront.TimelineGameMode.Staging) end
```

---

### 20.2 Tooltips

```lua
Storefront.Timeline.SetTooltip(description, timeDelta)   -- timeDelta optional, default 0
Storefront.Timeline.ClearTooltip(timeDelta)              -- timeDelta optional, default 0
```

Sets the text Steam shows when the player hovers the current point of the timeline. `timeDelta` shifts the moment
the change applies to, in seconds **into the past** — use it when you notice a state change a frame or two late.

```lua
Storefront.Timeline.SetTooltip("Boss: The Hollow King — phase 2")
```

---

### 20.3 Events

An **instantaneous** event is a single marker; a **range** event covers a stretch of time. Both accept an
`iconPriority` (0–1000, higher wins when markers collide) and a `clipPriority` telling Steam how strongly to
suggest the moment as a clip.

```lua
Storefront.Timeline.AddInstantaneousEvent(args) -> int   -- event handle, 0 on failure
Storefront.Timeline.AddRangeEvent(args)         -> int
Storefront.Timeline.StartRangeEvent(args)       -> int
Storefront.Timeline.UpdateRangeEvent(event, args)
Storefront.Timeline.EndRangeEvent(event, endOffsetSeconds)   -- offset optional, default 0
Storefront.Timeline.RemoveEvent(event)
```

`args` is a table:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `title` | `string` | `""` | Short label shown on the timeline |
| `description` | `string` | `""` | Longer text shown on hover |
| `icon` | `string` | `""` | A Steam-provided icon name, or one of your app's uploaded timeline icons |
| `iconPriority` | `int` | `0` | 0–1000; higher survives when Steam has to thin out markers |
| `startOffsetSeconds` | `float` | `0` | Seconds **into the past** the event actually happened |
| `duration` | `float` | `0` | `AddRangeEvent` only; capped at 600 seconds by Steam |
| `clipPriority` | `int` | `None` | A [`TimelineClipPriority`](#timelineclippriority-enum) value |

Use `AddRangeEvent` when you already know the length, and `StartRangeEvent` / `EndRangeEvent` when you do not.
Keep the handle from `StartRangeEvent` — it is what `UpdateRangeEvent`, `EndRangeEvent` and `RemoveEvent` take.

```lua
-- A single marker for a kill.
Storefront.Timeline.AddInstantaneousEvent({
    title = "Boss defeated", description = "The Hollow King",
    icon = "steam_achievement", iconPriority = 900,
    clipPriority = Storefront.TimelineClipPriority.Featured,
})

-- A range that lasts as long as the fight does.
local fight = nil
function OnBossFightStart()
    fight = Storefront.Timeline.StartRangeEvent({
        title = "Boss fight", icon = "steam_combat", iconPriority = 800,
        clipPriority = Storefront.TimelineClipPriority.Standard,
    })
end
function OnBossFightEnd()
    if fight then Storefront.Timeline.EndRangeEvent(fight); fight = nil end
end
```

---

### 20.4 Game phases

A **phase** is a larger chunk than an event — a run, a match, a dungeon. Steam groups a recording by phase and
lets the player jump between them, and phases carry **tags** and **attributes** that show up as searchable
metadata on the recording.

```lua
Storefront.Timeline.StartGamePhase()
Storefront.Timeline.EndGamePhase()
Storefront.Timeline.SetGamePhaseId(phaseId)                     -- your own id, max 64 chars
Storefront.Timeline.AddGamePhaseTag(tagName, tagIcon, tagGroup, priority)
Storefront.Timeline.SetGamePhaseAttribute(group, value, priority)
Storefront.Timeline.OpenOverlayToGamePhase(phaseId)
Storefront.Timeline.OpenOverlayToEvent(event)
```

Give every phase an id you can reconstruct later (a run seed, a match id) — that is the handle you pass to
`OpenOverlayToGamePhase` to take the player straight to that recording from your own UI.

```lua
function OnRunStart(seed)
    Storefront.Timeline.StartGamePhase()
    Storefront.Timeline.SetGamePhaseId("run-" .. seed)
    Storefront.Timeline.AddGamePhaseTag("Nightmare", "steam_difficulty", "Difficulty", 100)
end

function OnRunEnd(kills)
    Storefront.Timeline.SetGamePhaseAttribute("Kills", tostring(kills), 100)
    Storefront.Timeline.EndGamePhase()
end
```

> **Nothing here requires the player to be recording.** If Game Recording is off, or the Steam client predates
> the Timeline interface, every call above is a no-op that costs nothing. Instrument freely.

---

## 21. Enumerations Reference

All enum values are plain integers exposed as fields of a table on `Storefront`. Compare against the named constants
rather than hard-coding numbers.

<a id="backend-enum"></a>
### Storefront.Backend

| Constant | Value | Meaning |
|----------|-------|---------|
| `None` | `0` | No backend active |
| `Steam` | `1` | Steam backend |
| `Epic` | `2` | Reserved for an Epic backend |
| `GOG` | `3` | Reserved for a GOG Galaxy backend |
| `Microsoft` | `4` | Reserved for a Microsoft Store backend |
| `Itch` | `5` | Reserved for an itch.io backend |
| `Discord` | `6` | Reserved for a Discord backend |
| `Custom` | `7` | Reserved for a backend of your own |

Only `Steam` ships today. The remaining identifiers are fixed in advance so that a script written now keeps
comparing correctly if the plugin ever gains another backend.

<a id="capability-enum"></a>
### Storefront.Capability

Passed to [`Storefront.Supports`](#414-storefrontsupports) and [`Storefront.CapabilityName`](#415-storefrontcapabilityname).

| Constant | Value | Covers |
|----------|-------|--------|
| `Achievements` | `0` | Unlock, clear, progress, icons, global percentages |
| `Stats` | `1` | Integer/float stats, averages, storing |
| `Leaderboards` | `2` | Find-or-create, upload, download |
| `Cloud` | `3` | Cloud read/write/delete/list and quota |
| `Friends` | `4` | The friends list |
| `RichPresence` | `5` | Rich presence and the connect string |
| `Overlay` | `6` | Overlay dialogs and web pages |
| `Lobbies` | `7` | Matchmaking lobbies and lobby chat |
| `P2P` | `8` | Peer-to-peer messaging |
| `Voice` | `9` | Voice capture, compression and decoding |
| `DLC` | `10` | The DLC list, install state and store pages |
| `AuthTickets` | `11` | Session, Web API and encrypted app tickets |
| `Workshop` | `12` | `Storefront.Workshop.*` — user-generated content |
| `Input` | `13` | `Storefront.Input.*` — controllers, actions, glyphs |
| `Timeline` | `14` | `Storefront.Timeline.*` — Game Recording markers |
| `Screenshots` | `15` | Triggering screenshots |
| `AppInfo` | `16` | Ownership, build id, beta, install dir, launch parameters |
| `Device` | `17` | Steam Deck / Machine / Frame, Proton, VR, battery, performance |
| `GamepadText` | `18` | On-screen keyboard input |
| `Avatars` | `19` | Friend avatars and image handles |

With Steam active every capability reports `true` except `Timeline`, which also requires a Steam client new enough
to expose the Timeline API.

<a id="result-enum"></a>
### Storefront.Result

| Constant | Value | Meaning |
|----------|-------|---------|
| `Ok` | `0` | Success |
| `Pending` | `1` | Accepted; still in progress |
| `NotInitialized` | `-1` | No backend / not logged in |
| `NotLoggedIn` | `-2` | User not logged in to Steam |
| `NotFound` | `-3` | The requested item/file/key does not exist |
| `AlreadyExists` | `-4` | Already exists |
| `InvalidArgument` | `-5` | A parameter was invalid (e.g. payload too large) |
| `PermissionDenied` | `-6` | Not permitted |
| `NetworkFailure` | `-7` | Network/connection failure |
| `QuotaExceeded` | `-8` | Storage/size quota exceeded |
| `Timeout` | `-9` | Operation timed out |
| `Cancelled` | `-10` | Operation was cancelled |
| `Unsupported` | `-11` | The active backend does not support this |
| `DuplicateRequest` | `-12` | A duplicate request is already in flight |
| `BackendError` | `-100` | A generic Steam/backend error |

Convert any code to its name with [`Storefront.ResultName(code)`](#412-storefrontresultname).

<a id="lobbytype-enum"></a>
### Storefront.LobbyType

| Constant | Value | Meaning |
|----------|-------|---------|
| `Private` | `0` | Invite-only; not listed |
| `FriendsOnly` | `1` | Joinable by friends; not publicly listed |
| `Public` | `2` | Listed in lobby searches |
| `Invisible` | `3` | Joinable but hidden from the user's friends list view |

<a id="leaderboardsort-enum"></a>
### Storefront.LeaderboardSort

| Constant | Value | Meaning |
|----------|-------|---------|
| `Ascending` | `0` | Lower scores rank higher (e.g. lap times) |
| `Descending` | `1` | Higher scores rank higher (e.g. points) |

<a id="leaderboarddisplay-enum"></a>
### Storefront.LeaderboardDisplay

| Constant | Value | Meaning |
|----------|-------|---------|
| `Numeric` | `0` | Plain number |
| `TimeSeconds` | `1` | Display as seconds |
| `TimeMilliSeconds` | `2` | Display as milliseconds |

<a id="leaderboardupload-enum"></a>
### Storefront.LeaderboardUpload

| Constant | Value | Meaning |
|----------|-------|---------|
| `KeepBest` | `0` | Only replace the user's entry if the new score is better |
| `ForceUpdate` | `1` | Always replace the user's entry |

<a id="leaderboardrange-enum"></a>
### Storefront.LeaderboardRange

| Constant | Value | Meaning |
|----------|-------|---------|
| `Global` | `0` | Absolute ranks |
| `AroundUser` | `1` | Relative to the local user's rank |
| `Friends` | `2` | The user's friends only |

<a id="p2pchannel-enum"></a>
### Storefront.P2PChannel

| Constant | Value | Suggested use |
|----------|-------|---------------|
| `Default` | `0` | General messages |
| `GameState` | `1` | Gameplay state snapshots |
| `Voice` | `2` | Voice data |
| `FileTransfer` | `3` | Bulk transfers |

<a id="p2psendtype-enum"></a>
### Storefront.P2PSendType

| Constant | Value | Meaning |
|----------|-------|---------|
| `Unreliable` | `0` | May drop / reorder; lowest latency |
| `UnreliableNoDelay` | `1` | Unreliable and never buffered |
| `Reliable` | `2` | Guaranteed, ordered (TCP-like) |
| `ReliableWithBuffering` | `3` | Reliable; may coalesce small sends |

<a id="workshopvisibility-enum"></a>
### Storefront.WorkshopVisibility

| Constant | Value | Meaning |
|----------|-------|---------|
| `Public` | `0` | Visible to everyone |
| `FriendsOnly` | `1` | Visible to the author's friends |
| `Private` | `2` | Visible only to the author |
| `Unlisted` | `3` | Accessible by direct link, not listed |

<a id="inputtype-enum"></a>
### Storefront.InputType

| Constant | Value | Device |
|----------|-------|--------|
| `Unknown` | `0` | Unknown |
| `SteamController` | `1` | Steam Controller |
| `Xbox360` | `2` | Xbox 360 |
| `XboxOne` | `3` | Xbox One |
| `GenericGamepad` | `4` | Generic gamepad |
| `PS4` | `5` | DualShock 4 |
| `SwitchProController` | `10` | Switch Pro |
| `PS5` | `13` | DualSense |
| `SteamDeck` | `14` | Steam Deck built-in |

<a id="itemstate-enum"></a>
### Storefront.ItemState

A **bit mask**, returned by [`Storefront.Workshop.GetItemState`](#143-storefrontworkshopgetitemstate). Test with
`state & Storefront.ItemState.Installed ~= 0`. The booleans on a Workshop item table cover the same ground more
conveniently.

| Constant | Value | Meaning |
|----------|-------|---------|
| `None` | `0` | Item not tracked on this client |
| `Subscribed` | `1` | The user is subscribed to it |
| `LegacyItem` | `2` | Created with the old ISteamRemoteStorage API |
| `Installed` | `4` | Content is on disk and usable |
| `NeedsUpdate` | `8` | Not downloaded yet, or the author published a newer version |
| `Downloading` | `16` | Currently downloading |
| `DownloadPending` | `32` | Queued for download |
| `DisabledLocally` | `64` | The user disabled it in Steam's own UI — do not load it |

<a id="workshopupdatestatus-enum"></a>
### Storefront.WorkshopUpdateStatus

The `status` field of [`Storefront.Workshop.GetUpdateProgress`](#1414-storefrontworkshopgetupdateprogress).

| Constant | Value | Meaning |
|----------|-------|---------|
| `Invalid` | `0` | No upload in flight (or it just finished) |
| `PreparingConfig` | `1` | Processing the item's configuration |
| `PreparingContent` | `2` | Reading and processing content files |
| `UploadingContent` | `3` | Uploading content — this is the long one |
| `UploadingPreviewFile` | `4` | Uploading the preview image |
| `CommittingChanges` | `5` | Committing |

<a id="authsessionresult-enum"></a>
### Storefront.AuthSessionResult

Returned **synchronously** by [`Storefront.BeginAuthSession`](#175-storefrontbeginauthsession). `Ok` only means
the ticket was accepted for checking — the verdict arrives in
[`OnAuthSessionValidated`](#197-storefrontonauthsessionvalidated).

| Constant | Value | Meaning |
|----------|-------|---------|
| `Ok` | `0` | Ticket accepted; wait for the validation callback |
| `InvalidTicket` | `1` | Not a valid ticket |
| `DuplicateRequest` | `2` | A ticket for this user is already being checked |
| `InvalidVersion` | `3` | Ticket from an incompatible interface version |
| `GameMismatch` | `4` | Ticket is for a different game |
| `ExpiredTicket` | `5` | Ticket has expired |

<a id="authsessionresponse-enum"></a>
### Storefront.AuthSessionResponse

The verdict delivered to [`OnAuthSessionValidated`](#197-storefrontonauthsessionvalidated). Anything other than
`Ok` means you should drop the connection.

| Constant | Value | Meaning |
|----------|-------|---------|
| `Ok` | `0` | Verified: online, ticket valid, not reused |
| `UserNotConnectedToSteam` | `1` | The user is not connected to Steam |
| `NoLicenseOrExpired` | `2` | No license, or it expired |
| `VACBanned` | `3` | VAC banned for this game |
| `LoggedInElseWhere` | `4` | The account logged in somewhere else |
| `VACCheckTimedOut` | `5` | VAC could not run its checks |
| `AuthTicketCanceled` | `6` | The issuer cancelled the ticket |
| `AuthTicketInvalidAlreadyUsed` | `7` | Ticket already used |
| `AuthTicketInvalid` | `8` | Ticket is not from a connected user instance |
| `PublisherIssuedBan` | `9` | Banned by the publisher through the web API |
| `NetworkIdentityFailure` | `10` | The identity in the ticket does not match the authenticator |

<a id="licenseresult-enum"></a>
### Storefront.LicenseResult

Returned by [`Storefront.UserHasLicenseForApp`](#177-storefrontuserhaslicenseforapp).

| Constant | Value | Meaning |
|----------|-------|---------|
| `HasLicense` | `0` | The user owns the app/DLC |
| `DoesNotHaveLicense` | `1` | The user does not own it |
| `NoAuth` | `2` | The user has not been authenticated — call `BeginAuthSession` first |

<a id="overlaynotificationposition-enum"></a>
### Storefront.OverlayNotificationPosition

Corner used by [`Storefront.SetOverlayNotificationPosition`](#169-overlay-notification-placement).

| Constant | Value |
|----------|-------|
| `TopLeft` | `0` |
| `TopRight` | `1` |
| `BottomLeft` | `2` |
| `BottomRight` | `3` |

<a id="performancesetting-enum"></a>
### Storefront.PerformanceSetting

Passed to [`Storefront.SetGamePerformanceSetting`](#1610-reporting-your-graphics-settings-to-steam). This is what
your game *is currently running at*, reported to Steam — not a recommendation coming back.

| Constant | Value |
|----------|-------|
| `NotSet` | `0` |
| `Low` | `1` |
| `Medium` | `2` |
| `High` | `3` |
| `Ultra` | `4` |
| `Custom` | `5` |

<a id="glyphsize-enum"></a>
### Storefront.GlyphSize

Passed to [`Storefront.Input.GetGlyphPNGForActionOrigin`](#159-action-origins--glyphs).

| Constant | Value | Pixels |
|----------|-------|--------|
| `Small` | `0` | 32×32 |
| `Medium` | `1` | 128×128 |
| `Large` | `2` | 256×256 |

<a id="timelinegamemode-enum"></a>
### Storefront.TimelineGameMode

Passed to [`Storefront.Timeline.SetGameMode`](#201-game-mode--colouring-the-timeline).

| Constant | Value | Meaning |
|----------|-------|---------|
| `Invalid` | `0` | Unset |
| `Playing` | `1` | Actual gameplay |
| `Staging` | `2` | Between-action screens: inventory, shop, lobby |
| `Menus` | `3` | Main menu and settings |
| `LoadingScreen` | `4` | Loading |

<a id="timelineclippriority-enum"></a>
### Storefront.TimelineClipPriority

How strongly Steam should offer a timeline event as a clip.

| Constant | Value | Meaning |
|----------|-------|---------|
| `Invalid` | `0` | Unset |
| `None` | `1` | Mark the moment, never suggest a clip |
| `Standard` | `2` | Offer it as a clip |
| `Featured` | `3` | Offer it ahead of standard events |

<a id="steamhardware-enum"></a>
### Storefront.SteamHardware

Returned by [`Storefront.GetSteamHardware()`](#161-device--environment-queries). Intended for analytics, support
and diagnostics; for feature decisions prefer `Storefront.HardwareConfig` and the capability queries.

| Constant | Value | Device |
|----------|-------|--------|
| `None` | `0` | Not Steam hardware |
| `SteamDeck` | `1` | Steam Deck |
| `SteamMachine` | `2` | Steam Machine |
| `SteamFrame` | `3` | Steam Frame |

<a id="hardwareconfig-enum"></a>
### Storefront.HardwareConfig

Returned by [`Storefront.GetHardwareDefaultConfig()`](#161-device--environment-queries). Map the device-specific
values to a tuned configuration and the general values to one of your own graphics presets. Values can also appear
for third-party hardware with comparable performance, and Valve can re-map them per device from the Steamworks
partner site without a game patch.

| Constant | Value | Meaning |
|----------|-------|---------|
| `None` | `0` | No recommendation; use your own heuristics |
| `Low` | `1` | Your lowest preset |
| `Medium` | `2` | Your medium preset |
| `High` | `3` | Your high preset |
| `Max` | `4` | Your highest preset |
| `SteamDeck` | `5` | Your Steam Deck tuned configuration |
| `SteamMachine` | `6` | Your Steam Machine tuned configuration |
| `SteamFrame` | `7` | Your Steam Frame tuned configuration |

<a id="floatingkeyboardmode-enum"></a>
### Storefront.FloatingKeyboardMode

| Constant | Value | Meaning |
|----------|-------|---------|
| `SingleLine` | `0` | Single line of text |
| `MultipleLines` | `1` | Multi-line text |
| `Email` | `2` | Email-optimized layout |
| `Numeric` | `3` | Numeric layout |

---

## 22. Returned Tables Reference

A consolidated reference of every table the API returns. All arrays are **1-indexed**. Timestamps are **Unix
seconds**. Byte fields are **binary-safe Lua strings**.

<a id="userhandle-table-ref"></a>
### UserHandle

Identifies a Steam user. See [The `UserHandle` table](#the-userhandle-table).

| Field | Type | Description |
|-------|------|-------------|
| `backend` | `int` | `1` for Steam |
| `id` | `int` | 64-bit SteamID as an integer |
| `textId` | `string` | SteamID64 as a decimal string |

<a id="achievement-table"></a>
### Achievement

Returned by [`GetAchievement`](#54-storefrontgetachievement), [`GetAllAchievements`](#55-storefrontgetallachievements),
and [`OnAchievementUnlocked`](#194-storefrontonachievementunlocked).

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | API name |
| `displayName` | `string` | Localized name |
| `description` | `string` | Localized description |
| `hidden` | `bool` | Hidden until unlocked |
| `unlocked` | `bool` | Owned by the local user |
| `progress` | `float` | Current progress (`0` if not tracked here) |
| `progressMax` | `float` | Max progress (`0` if not a progress achievement) |
| `globalUnlockPercent` | `float` | Global unlock %, or `-1` if unavailable |
| `unlockTime` | `int` | Unlock Unix time, or `0` |
| `iconLocked` | `string` | Reserved (empty) |
| `iconUnlocked` | `string` | Reserved (empty) |

<a id="leaderboardentry-table"></a>
### LeaderboardEntry

Returned by [`DownloadLeaderboardEntries`](#73-storefrontdownloadleaderboardentries).

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | Owner's [UserHandle](#userhandle-table-ref) |
| `displayName` | `string` | Owner's persona name |
| `rank` | `int` | Global rank |
| `score` | `int` | Score |
| `details` | `table` | Array of integer details |

<a id="cloudfile-table"></a>
### CloudFile

Returned by [`CloudList`](#84-storefrontcloudlist).

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | File name |
| `size` | `int` | Size in bytes |
| `timestamp` | `int` | Last-modified Unix time |

<a id="friend-table"></a>
### Friend

Returned by [`GetFriends`](#91-storefrontgetfriends).

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | Friend's [UserHandle](#userhandle-table-ref) |
| `personaName` | `string` | Display name |
| `statusText` | `string` | Rich-presence status |
| `online` | `bool` | Online |
| `playingOurGame` | `bool` | Playing this app |
| `inLobby` | `bool` | In a lobby for this app |
| `lobby` | `int` | Lobby id, or `0` |

<a id="lobbyinfo-table"></a>
### LobbyInfo

Returned by [`CreateLobby`](#101-storefrontcreatelobby), [`JoinLobby`](#102-storefrontjoinlobby), and
[`SearchLobbies`](#104-storefrontsearchlobbies).

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Lobby id |
| `owner` | `table` | Owner's [UserHandle](#userhandle-table-ref) |
| `memberCount` | `int` | Current members |
| `memberLimit` | `int` | Maximum members |
| `data` | `table` | `string → string` metadata map |

<a id="lobbymember-table"></a>
### LobbyMember

Returned by [`GetLobbyMembers`](#106-storefrontgetlobbymembers).

| Field | Type | Description |
|-------|------|-------------|
| `user` | `table` | Member's [UserHandle](#userhandle-table-ref) |
| `personaName` | `string` | Display name |
| `data` | `table` | `string → string` per-member data |

<a id="p2pmessage-table"></a>
### P2PMessage

Returned by [`ReceiveP2P`](#112-storefrontreceivep2p) and [`OnP2PMessage`](#191-storefrontonp2pmessage).

| Field | Type | Description |
|-------|------|-------------|
| `sender` | `table` | Sender's [UserHandle](#userhandle-table-ref) |
| `channel` | `int` | [P2PChannel](#p2pchannel-enum) it arrived on |
| `payload` | `string` | Raw bytes |

<a id="dlc-table"></a>
### DLC

Returned by [`GetDLCs`](#131-storefrontgetdlcs).

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | DLC AppId as a string |
| `name` | `string` | DLC name |
| `appId` | `int` | DLC AppId |
| `owned` | `bool` | Owned/subscribed |
| `installed` | `bool` | Installed |
| `bytesDownloaded` | `int` | Download progress |
| `bytesTotal` | `int` | Download size |

<a id="workshopitem-table"></a>
### WorkshopItem

Returned by [`Workshop.GetSubscribed`](#141-storefrontworkshopgetsubscribed),
[`Workshop.GetItemInfo`](#142-storefrontworkshopgetiteminfo),
[`Workshop.QueryUserItems`](#146-storefrontworkshopqueryuseritems) and
[`Workshop.QueryItemsByIds`](#1411-storefrontworkshopqueryitemsbyids). Listing fields (title, votes, tags, metadata,
…) are only populated by the two **query** functions; the others are local state that is always filled in.

| Field | Type | Description |
|-------|------|-------------|
| `itemId` | `int` | Published file id |
| `title` | `string` | Title |
| `description` | `string` | Description |
| `owner` | `table` | Author's [UserHandle](#userhandle-table-ref) |
| `previewUrl` | `string` | Preview image URL |
| `metadata` | `string` | Developer metadata |
| `tags` | `table` | Array of tag strings |
| `fileSize` | `int` | Content size in bytes |
| `subscriptions` | `int` | Subscription count |
| `favorites` | `int` | Favorite count |
| `votesUp` | `int` | Up votes |
| `votesDown` | `int` | Down votes |
| `score` | `float` | Steam's computed score |
| `timeCreated` | `int` | Creation Unix time |
| `timeUpdated` | `int` | Last-update Unix time |
| `subscribed` | `bool` | Locally subscribed |
| `installed` | `bool` | Installed locally |
| `needsUpdate` | `bool` | An update is available |
| `downloading` | `bool` | Currently downloading |
| `disabledLocally` | `bool` | The player disabled it in Steam's own UI — do not load it |
| `banned` | `bool` | Banned on the Workshop (query results only) |
| `bytesDownloaded` | `int` | Download progress |
| `bytesTotal` | `int` | Download size |
| `installFolder` | `string` | Local content folder |
| `visibility` | `int` | A [WorkshopVisibility](#workshopvisibility-enum) value |

<a id="controller-table"></a>
### Controller

Returned by [`Input.GetControllers`](#154-storefrontinputgetcontrollers).

| Field | Type | Description |
|-------|------|-------------|
| `handle` | `int` | Controller handle |
| `inputType` | `int` | An [InputType](#inputtype-enum) value |
| `isSteamDeck` | `bool` | Steam Deck built-in controller |
| `gamepadIndex` | `int` | Emulated Xbox gamepad index, or `-1` |

---

## 23. Practical Examples

### 22.1 Minimal setup — guard, identify, tick

```lua
-- Level script (.icemap) that owns the Steam lifecycle for the session.
function OnCreate()
    if not Storefront.IsAvailable() or not Storefront.IsLoggedIn() then
        Print("Steam not available — continuing without it")
        return
    end
    Print("Steam ready for " .. Storefront.GetPersonaName()
          .. " (" .. Storefront.GetLocalUser().textId .. ")")
end

function OnUpdate(dt)
    Storefront.Tick()    -- optional: the plugin already flushes callbacks/events every frame
end
```

---

### 22.2 Achievements + stats at end of a run

```lua
function OnRunComplete(score, kills, distance)
    if not Storefront.IsAvailable() then return end

    -- Stats
    local total = select(1, Storefront.GetStatInt("total_kills")) or 0
    Storefront.SetStatInt("total_kills", total + kills)
    Storefront.SetStatFloat("best_distance", distance)
    Storefront.StoreStats()

    -- Achievements
    if kills >= 100 then Storefront.UnlockAchievement("ACH_KILL_100") end
    if score >= 10000 then Storefront.UnlockAchievement("ACH_SCORE_10K") end

    -- Progress toast toward a long-term goal
    Storefront.IndicateAchievementProgress("ACH_KILL_1000",
        math.min(total + kills, 1000), 1000)
end
```

---

### 22.3 Leaderboards — submit and show top 10

```lua
local board = nil

function OnCreate()
    Storefront.FindOrCreateLeaderboard("HighScores",
        Storefront.LeaderboardSort.Descending,
        Storefront.LeaderboardDisplay.Numeric,
        function(r, handle)
            if r == Storefront.Result.Ok then board = handle end
        end)
end

function SubmitScore(score)
    if not board then return end
    Storefront.UploadLeaderboardScore(board, score, {},
        Storefront.LeaderboardUpload.KeepBest, function(r)
            -- after upload, refresh the top 10
            Storefront.DownloadLeaderboardEntries(board,
                Storefront.LeaderboardRange.Global, 1, 10,
                function(r2, entries)
                    if r2 ~= Storefront.Result.Ok then return end
                    for _, e in ipairs(entries) do
                        Print(e.rank .. ". " .. e.displayName .. " — " .. e.score)
                    end
                end)
        end)
end
```

---

### 22.4 Cloud saves with local fallback

```lua
function SaveGame(slot, bytes)
    local name = "slot" .. slot .. ".sav"
    if Storefront.IsAvailable() and Storefront.CloudIsEnabled() then
        local r = Storefront.CloudWrite(name, bytes)
        if r == Storefront.Result.Ok then return true end
    end
    return SaveToDisk(name, bytes)   -- your local fallback
end

function LoadGame(slot)
    local name = "slot" .. slot .. ".sav"
    if Storefront.IsAvailable() and Storefront.CloudIsEnabled() then
        local data, r = Storefront.CloudRead(name)
        if r == Storefront.Result.Ok then return data end
    end
    return LoadFromDisk(name)
end
```

---

### 22.5 Co-op lobby with invites and join-game

```lua
local myLobby = nil

function HostCoop()
    Storefront.CreateLobby(Storefront.LobbyType.FriendsOnly, 4, function(r, lobby)
        if r ~= Storefront.Result.Ok then return end
        myLobby = lobby.id
        Storefront.SetLobbyData(myLobby, "map", CurrentMap())
        Storefront.SetRichPresence({
            statusText    = "Hosting co-op",
            connectString = "+connect_lobby " .. myLobby,
        })
    end)
end

function InviteFriends()
    if myLobby then Storefront.ActivateOverlayInviteDialog(myLobby) end
end

function OnCreate()
    -- Accepting an invite (or launching via +connect) routes here:
    Storefront.OnConnectInvite(function(connect)
        local id = connect:match("connect_lobby%s+(%d+)")
        if id then
            Storefront.JoinLobby(math.tointeger(id), function(r, lobby)
                if r == Storefront.Result.Ok then EnterCoopMap(lobby) end
            end)
        end
    end)

    Storefront.OnLobbyMemberChanged(function(lobbyId, user, joined)
        Print(user.textId .. (joined and " joined" or " left"))
    end)
end

function OnUpdate(dt) Storefront.Tick() end
```

---

### 22.6 P2P state sync (callback style)

```lua
local peers = {}

function OnCreate()
    Storefront.OnP2PMessage(function(msg)
        if msg.channel == Storefront.P2PChannel.GameState then
            local x, y, a = string.unpack("<fff", msg.payload)
            ApplyRemoteState(msg.sender.textId, x, y, a)
        end
    end)
end

function OnUpdate(dt)
    Storefront.Tick()
    local snapshot = string.pack("<fff", myX, myY, myAngle)
    for _, peer in ipairs(peers) do
        Storefront.SendP2P(peer, snapshot,
            Storefront.P2PChannel.GameState,
            Storefront.P2PSendType.Unreliable)
    end
end
```

---

### 22.7 Steam Deck adaptation

```lua
function OnCreate()
    if Storefront.IsSteamDeck() then
        Settings.SetUIScale(1.25)
        Settings.SetDefaultInput("controller")
    end
    if Storefront.IsBigPictureMode() then
        EnableGamepadNavigation()
    end
end

function OnUpdate(dt)
    Storefront.Tick()
    local bat = Storefront.GetBatteryPower()
    if bat ~= 255 and bat <= 10 then
        ShowLowBatteryWarning(bat)
    end
end
```

---

### 22.8 Publishing a Workshop item

```lua
function PublishMap(folder, previewPng)
    Storefront.Workshop.CreateItem({
        title            = "Forsaken Keep",
        description      = "A 4-player survival map.",
        contentFolder    = folder,
        previewImagePath = previewPng,
        visibility       = Storefront.WorkshopVisibility.Public,
        tags             = { "Map", "Co-op" },
    }, function(r, itemId)
        if r == Storefront.Result.Ok then
            Print("Published! Item id: " .. itemId)
            SetString("workshop_item_id", tostring(itemId))
        else
            Print("Publish failed: " .. Storefront.ResultName(r))
        end
    end)
end
```

---

## 24. Troubleshooting & FAQ

**`Storefront.IsAvailable()` returns `false`.**
The plugin did not initialize a backend. Common causes: no AppId configured (create `steam_config.json` with
`{"AppId": <id>}` in the plugin folder, or set `ICEBOX_STEAM_APPID`); the Steam client is not running; the signed-in
account does not own the AppId; or this is a non-desktop build (Android/iOS/Web — the plugin is desktop-only). With
the test AppId `480`, make sure you own/own-via-free Spacewar or that a `steam_appid.txt` exists during development.
The engine log always says which of these it was — look for the `[Steam]` lines.

The plugin **retries at most once every five seconds** while it has no backend, so starting the Steam client after
the game or the editor is already running is enough: the log then shows `[Steam] SteamAPI connected on retry` and
`IsAvailable()` starts returning `true`. Creating `steam_config.json` after the fact works the same way — no restart
needed. Each distinct failure is logged once, not once per retry.

The retry runs from the plugin's per-frame update, which the engine drives **while the runtime is running** — in the
editor that means Play mode, not edit mode. In a shipped game it is always running. So in the editor: start Steam,
enter Play mode, and the backend attaches within five seconds.

**Scripts fail with "attempt to index a nil value (global 'Storefront')".**
Either the plugin is not enabled in **Tools → Plugins & Mods**, or the engine log carries a
`[Steam] Lua ABI mismatch` line. The plugin is linked against the very same Lua the engine links, and the two
must be the same version — a plugin built against a stale dependency tree refuses to bind rather than crash the
editor. Download the plugin release built for **your** engine version — the release notes state which one each
build targets — and replace the plugin library with it. You cannot rebuild it yourself; the source is not
published.

**My callbacks / events never fire.**
The queue is flushed from the plugin's engine update, which only runs while the **runtime** is running — in the
editor that means Play mode, not edit mode. Check that the plugin is enabled in `Config/Plugins.json`, that
`Storefront.IsAvailable()` returns `true`, and that the callback itself is not raising a Lua error (errors inside a
callback are swallowed by the protected call). A manual [`Storefront.Tick()`](#the-frame-tick--storefronttick) in a
level-script `OnUpdate` will not hurt, but it is no longer what makes callbacks fire.

**`ReceiveP2P()` always returns `nil` even though messages are arriving.**
You registered an [`OnP2PMessage`](#191-storefrontonp2pmessage) handler. While a handler is set, messages are
delivered to it and not queued for polling — use one mechanism or the other.

**Achievements unlock in code but I see no toast.**
The Steam overlay must be enabled and the game launched through Steam. Also confirm the achievement **API name**
(not the display name) is correct and that stats finished syncing — reads come from a cache populated shortly after
init.

**`globalUnlockPercent` is `-1`.**
Global stats have not arrived yet. The plugin requests them on init; call
[`RequestGlobalAchievementPercentages`](#164-storefrontrequestglobalachievementpercentages) and read the value a few
frames later.

**Friend avatars / achievement icons return `nil`.**
Steam fetches images asynchronously. The first call requests the image; retrieve the pixels again a frame or two
later once it has loaded.

**Lobby data set by a member isn't visible to others.**
Lobby metadata that other players read should be set by the lobby **owner**. Per-member data is separate.

**Should I call `SteamAPI_RunCallbacks` myself?**
No. The plugin pumps native Steam callbacks and flushes the queued Lua results automatically every frame.
`Storefront.Tick()` is only there if you want the flush to happen at a specific point of your own frame.

**Do these scripts break on Android / iOS / Web?**
No — the plugin simply isn't present there, so `Storefront.IsAvailable()` returns `false` and every call is a safe
no-op. Guarding on `IsAvailable()` keeps one script working across all platforms.

---

## 25. Visual Scripting Nodes

The plugin ships its own visual-scripting catalog — `VisualScriptAPI.json` in the plugin root. The IceBoxEngineEditor
loads such catalogs from every plugin folder automatically, so when the Storefront plugin is present the whole
`Storefront` API is also available as nodes in the Visual Script editor. No engine configuration is required.

### What you get

- **A node for every function** of `Storefront`, `Storefront.Workshop` and `Storefront.Input`, grouped into
  categories in the node palette: *Steam*, *Steam Achievements*, *Steam Stats*, *Steam Leaderboards*,
  *Steam Cloud*, *Steam Lobby*, *Steam P2P*, *Steam DLC*, *Steam Overlay*, *Steam Voice*, *Steam Friends*,
  *Steam Auth*, *Steam Device*, *Steam Events*, *Steam Workshop*, *Steam Input*.
- **Enum dropdowns.** Arguments backed by a `Storefront` enum (lobby type, leaderboard sort / display / upload
  method, download range, P2P channel and send type, floating keyboard mode, result codes) get a dropdown picker
  with the full `Storefront.<Enum>.<Value>` expressions and a sensible default already selected.
- **Multiple return values.** Functions that return several values expose one output pin per value —
  `Get Stat Int` has `Value` and `Result` pins, `Get Image RGBA` has `Data`, `Width` and `Height`,
  `Decompress Voice` has `Result`, `Data` and `SampleRate`, and so on.
- **Pure getters.** Read-only calls (`Is Logged In`, `Get Persona Name`, `Cloud List`, …) are pure nodes without
  exec pins — plug their outputs straight into other inputs.
- **Callbacks.** Asynchronous functions expose a `Callback` pin. Create a Custom Event in your graph and feed it
  through a **Function Reference** node into the `Callback` input; the event fires with the same arguments the
  Lua callback would receive.

The golden rule from [§3](#the-frame-tick--storefronttick) still applies: run the **Tick** node (category
*Steam Events*) every frame — e.g. after `On Update` — or queued results and event callbacks will never arrive.

### Where the catalog comes from

`VisualScriptAPI.json` ships in the plugin folder, next to `plugin.json`, and is generated from the plugin's Lua
bindings before each release — so it always matches the library it came with. The editor reads it straight out of
the plugin folder; there is nothing to install, configure or regenerate.

It looks like a build artifact and it is not: **it is a required run-time data file.** Delete it and every
`Storefront` node silently disappears from the palette, with no error anywhere. Keep it with the plugin, and
replace it together with the library whenever you take a new release.

---

## Legal

**IceBoxStorefront Plugin** — © 2026 IceBoxCrew Studio. Licensed under `LICENSE.txt` in the plugin root.
Third-party notices: `THIRD_PARTY_NOTICES.txt`. Shipping your game: `DISTRIBUTION.md`. Summary: `NOTICE.md`.

The plugin is **free but not open source**: no source code is supplied, and republishing the plugin on its own is
not permitted. Using it in your games, including commercial ones, is free and unrestricted.

Steam, Steamworks, Steam Deck, Steam Machine, Steam Frame, Steam Cloud, Steam Workshop, Steam Input, Big Picture
and Proton are trademarks and/or registered trademarks of **Valve Corporation** in the United States and/or other
countries. This plugin and this document use those names descriptively only, to state which service the plugin
integrates with and to name the API calls, folders and configuration keys Valve itself defines. No Valve logo or
artwork is included.

**IceBoxStorefront Plugin is not made by, affiliated with, endorsed by or sponsored by Valve Corporation.**

The **Steamworks SDK is not distributed with this plugin**. Obtain it from
[Valve's Steamworks partner site](https://partner.steamgames.com/doc/sdk) under Valve's own *Steamworks SDK Access
Agreement*, which Valve concludes with you directly. Never place a file from that SDK into anything you hand to
another developer.

Using this plugin requires your own licensed copy of IceBoxEngine.

<sub>Nothing in this documentation is legal advice.</sub>








