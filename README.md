# IceBoxStorefront Plugin

**Steamworks integration for IceBoxEngine.** Windows · Linux · macOS.

Exposes the `Storefront` Lua API — 194 functions, all of them also visual-scripting nodes: achievements,
stats, leaderboards, Steam Cloud, friends, rich presence, overlay, lobbies, P2P networking, voice,
Workshop (UGC), DLC, authentication tickets, Steam Timeline / Game Recording, and Steam Input /
Steam Deck / Steam Machine / Steam Frame support.

> **Free, but not open source.** The plugin is distributed as a **compiled library** with its manifest,
> node catalog and documentation. There is no source code in this repository or in the release archive,
> and none is published. Use is free, commercial games included — see [License](#license).

> **Not a Valve product.** This plugin is developed by IceBoxCrew Studio and is **not made by, affiliated
> with, endorsed by or sponsored by Valve Corporation**. The **Steamworks SDK is not included** — you
> download it from Valve yourself under Valve's own
> [Steamworks SDK Access Agreement](https://partner.steamgames.com/doc/sdk).

## Get it

Download the release archive for **your platform and your engine version** from
[Releases](https://github.com/IceBoxCrew/IceBoxStorefrontPlugin/releases) or from itch.io. This repository
holds everything the plugin folder needs *except* the compiled library — the manifest, the node catalog,
the icon, the documentation and the legal texts — so a clone is a plugin folder waiting for one file.

## Install

**1 — Unzip into `Plugins/`.** The archive already contains a folder named `IceBoxStorefront`, so unpacking
it into your engine's `Plugins/` (available to every project) or a project's `Plugins/` (that project only)
puts it exactly where it belongs. **Do not rename the folder**: the engine identifies the plugin by that name.

**2 — Add Valve's runtime.** Download the Steamworks SDK from
[partner.steamgames.com/doc/sdk](https://partner.steamgames.com/doc/sdk) and copy **one file** out of it
into the plugin folder, next to `IceBoxStorefront.dll` / `IceBoxStorefront.so` / `IceBoxStorefront.dylib`:

| Platform | File | From |
| --- | --- | --- |
| Windows x64 | `steam_api64.dll` | `redistributable_bin/win64/` |
| Linux x64 | `libsteam_api.so` | `redistributable_bin/linux64/` |
| macOS | `libsteam_api.dylib` | `redistributable_bin/osx/` |

The plugin folder is the one location all three dynamic loaders agree on: on Windows the plugin loads
the runtime from there explicitly, on Linux it carries an `$ORIGIN` RPATH, and on macOS
`libsteam_api.dylib` is built with the install name `@loader_path/libsteam_api.dylib` and *must* sit
beside the library that links it. The engine matches a plugin's library by name, so the runtime in the
same folder is never mistaken for the plugin itself.

**3 — Configure your AppId.** Create `steam_config.json` next to `plugin.json`:

```json
{
    "AppId": 480,
    "RestartIfNecessary": true
}
```

| Key | Default | Meaning |
| --- | --- | --- |
| `AppId` | — | Your Steam AppId. `480` is Valve's public Spacewar test app. |
| `RestartIfNecessary` | `true` | Relaunch the shipped game through Steam when it was started directly. Set to `false` for a DRM-free build of the same game, or when you use Valve's DRM wrapper. |

`ICEBOX_STEAM_APPID` overrides the AppId, and `ICEBOX_STEAM_NO_RESTART=1` skips the relaunch check for
one run. During development, keep a `steam_appid.txt` holding the same number next to the built game —
and delete it from the release.

**4 — Enable it.** In the editor: **Tools → Plugins & Mods**, tick **IceBoxStorefront**. The `Storefront`
table appears in Lua and the nodes appear in the Visual Script editor. Nothing else to configure.

## Documentation

| | |
| --- | --- |
| English API reference | [`Documentation/EN/Storefront-LuaAPI-EN.md`](Documentation/EN/Storefront-LuaAPI-EN.md) |
| Русский справочник API | [`Documentation/RU/Storefront-LuaAPI-RU.md`](Documentation/RU/Storefront-LuaAPI-RU.md) |
| Shipping your game | [`DISTRIBUTION.md`](DISTRIBUTION.md) |
| License | [`LICENSE.txt`](LICENSE.txt) |
| Legal summary | [`NOTICE.md`](NOTICE.md) |
| Third-party notices | [`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt) |

The reference covers every function, enum, callback and returned table, with signatures, parameter
tables and runnable examples. It is the complete description of the API — you do not have the sources,
and you do not need them.

```lua
if Storefront.IsAvailable() then
    Storefront.UnlockAchievement("ACH_FIRST_WIN")
    Storefront.SetRichPresence({ statusText = "In the arena" })
end
```

Every call is a safe no-op when no backend is active, so the same script runs unchanged on platforms
without Steam. Guard with `Storefront.IsAvailable()` when you want to branch.

## What is in the plugin folder

- `IceBoxStorefront.dll` / `IceBoxStorefront.so` / `IceBoxStorefront.dylib` — the plugin, from the release
  archive. **Required.**
- `plugin.json` — the manifest. The engine identifies the plugin by its `Name`, which is
  `IceBoxStorefront`. **Required.**
- `VisualScriptAPI.json` — the node catalog. The editor loads it from the plugin folder and shows every
  `Storefront` function as a node. It looks generated, and it is, but it is a **required run-time data
  file** — a build without it silently loses every `Storefront` node.
- `icon.png` — 256×256 plugin icon, shown in the editor's Plugins panel and in the launcher's
  Plugins & Mods tab. Drop it and the plugin shows up as a blank tile.
- `Documentation/` — the full API reference, about 400 KB of Markdown. Optional at run time.
- `LICENSE.txt`, `NOTICE.md`, `THIRD_PARTY_NOTICES.txt` — keep them in the folder.
  `THIRD_PARTY_NOTICES.txt` is what satisfies the attribution sol2, Lua, nlohmann/json and fmt require.

## Engine versions

The plugin links the **same Lua the engine does**, so each build is tied to an engine version. Every
release says which one it was built against; when you update the engine, take the matching plugin build.

If the two do drift apart, the plugin **refuses to register the `Storefront` table** and writes a
`[Steam] Lua ABI mismatch` line to the log, rather than taking the editor down with an access violation.
Seeing that line means: get the release built for your engine version.

## Shipping your game

Things that are easy to get wrong and produce no error when you do — the full list is in
[`DISTRIBUTION.md`](DISTRIBUTION.md):

- **Enable the plugin in `Config/Plugins.json`.** The engine discovers every plugin folder but loads only
  the ones listed as enabled. Ticking **IceBoxStorefront** in the editor writes that file, and
  **Tools → Build Game** copies it. Without it your game ships the plugin and never loads it.
- **Do not ship `steam_appid.txt`.** It tells Steam to skip the launch check — right on your machine
  during testing, wrong in a release. `steam_config.json` *does* ship; that is where your game reads its
  AppId from.
- **Leave the Steam runtime in the plugin folder.** Distributing it *with your game* is permitted to you
  by Section 1.1 of your own Steamworks SDK Access Agreement.
- **`Documentation/` ships as-is** unless you delete it. Nothing breaks either way.
- **The editor initializes Steam too.** With a valid AppId, running the editor marks you as playing the
  game on Steam and lets you unlock real achievements from Play mode. That is deliberate — it is how you
  test — but use AppId `480` or disable the plugin while you are not testing Steam.

## Early boot

The plugin exports `IcePluginEarlyBoot`, which the **game runtime** (never the editor) calls before the
engine starts. It resolves the AppId and runs `SteamAPI_RestartAppIfNecessary`, so a player who launches
the executable directly still ends up running under Steam with the overlay, Cloud and achievements
intact. There is nothing to call from Lua.

## Visual scripting nodes

Every `Storefront`, `Storefront.Workshop`, `Storefront.Input` and `Storefront.Timeline` function is
available as a node, with enum dropdowns, multi-value output pins and pure getter nodes. The editor picks
`VisualScriptAPI.json` up on its own — there is nothing to install or configure. See the
"Visual Scripting Nodes" section of the documentation.

The golden rule: run the **Tick** node (category *Steam Events*) every frame in a graph that needs
callbacks, or use the automatic per-frame pump the plugin already performs while the runtime is running.

## License

© 2026 IceBoxCrew Studio. All rights reserved. See [`LICENSE.txt`](LICENSE.txt).

**Free of charge, forever, commercial games included** — no royalties, no per-project fee, no forced
credit. The compiled plugin travels inside a game you release to players; it has to, or the game does not
run.

**Not open source.** No source code is supplied and none is published. Do not republish the plugin on its
own — another site, a marketplace, an asset pack, someone else's SDK — and do not reverse engineer it. If
a colleague needs it, send them the download page; it is free and it takes a minute.

**The Steamworks SDK is not part of this package** and never has been. Obtain it from Valve, under
Valve's *Steamworks SDK Access Agreement*, which Valve concludes with you directly. IceBoxCrew Studio is
not a party to that agreement and sublicenses nothing under it. Never place a file from that SDK into
anything you hand to another developer.

Steam, Steamworks, Steam Deck, Steam Machine, Steam Frame, Steam Cloud, Steam Workshop, Steam Input, Big
Picture and Proton are trademarks and/or registered trademarks of **Valve Corporation** in the United
States and/or other countries, used here descriptively only. No Valve logo or artwork is included.
**This plugin is not made by, affiliated with, endorsed by or sponsored by Valve Corporation.**

The plugin is compiled against sol2, Lua, nlohmann/json and optionally fmt — see
[`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt). Using it requires your own licensed copy of
IceBoxEngine.

<sub>Nothing here is legal advice.</sub>
