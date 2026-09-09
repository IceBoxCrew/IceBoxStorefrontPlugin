# Notice

**IceBox Storefront Plugin** — Steamworks integration for IceBox Engine.
Copyright © 2026 IceBoxCrew Studio. Licensed under [LICENSE.txt](LICENSE.txt).

---

## Free, but not open source

The plugin costs nothing, has no royalties and may go into commercial games. It is
still **proprietary**: it is supplied as a **compiled library** together with its
manifest, node catalog, icon and documentation. There is no source code in this
package, and none is published anywhere.

What that means in practice:

- **Use it freely.** Any number of machines, any kind of game, commercial included.
- **Ship it inside your game.** The compiled plugin travels with the game you release
  to players — it has to, or the game will not run.
- **Do not republish the plugin on its own.** Not on another site, not in a
  marketplace, not in an asset pack, not inside someone else's SDK, and not for
  money. Point people at the official download instead.
- **Do not reverse engineer it.**

[LICENSE.txt](LICENSE.txt) is the operative text; Sections 2 and 4 are the ones above.

## This package contains no Valve software

Not one byte of the Steamworks SDK is in this package. No SDK headers, no import
libraries, no `steam_api64.dll`, no `libsteam_api.so`, no `libsteam_api.dylib`, no
SDK tools, no SDK samples.

The plugin **references** the SDK: it was compiled against Valve's headers on our
machine and it loads Valve's runtime library at start-up. Neither of those puts
Valve's code in this package.

**You obtain the Steam runtime yourself**, from
<https://partner.steamgames.com/doc/sdk>, by accepting Valve's *Steamworks SDK
Access Agreement*. Valve licenses it to you directly. IceBoxCrew Studio is not a
party to that agreement and sublicenses nothing under it.

You need exactly one file out of that SDK to run the plugin:

```
Windows   redistributable_bin/win64/steam_api64.dll
Linux     redistributable_bin/linux64/libsteam_api.so
macOS     redistributable_bin/osx/libsteam_api.dylib
```

Put it in the plugin folder, next to `Steam.dll` / `Steam.so` / `Steam.dylib`. That is
the one location all three dynamic loaders agree on.

## Shipping the runtime inside your own game is fine

Section 1.1 of Valve's *Steamworks SDK Access Agreement* lets you reproduce and
distribute the contents of the SDK's `redistributable_bin` folder **together with
your own application in object code form**. That is what makes `steam_api64.dll`
inside your released game legitimate, and it comes from Valve under your own
agreement with Valve.

It is also the only place a Valve file may travel. Valve's license to you is
*nontransferable*, so a Valve file must never be inside anything you hand to another
developer.

[DISTRIBUTION.md](DISTRIBUTION.md) lists exactly what belongs in the build you upload
and what must never.

## Trademarks

Steam, Steamworks, Steam Deck, Steam Machine, Steam Frame, Steam Cloud, Steam
Workshop, Steam Input, Big Picture and Proton are trademarks and/or registered
trademarks of Valve Corporation in the United States and/or other countries.

This plugin uses those names **descriptively only** — to say truthfully which
service it integrates with, and to name the API calls, folders, configuration keys
and identifiers Valve itself defines. It contains no Valve logo, no Valve artwork
and no Valve branding.

**IceBox Storefront Plugin is not made by, affiliated with, endorsed by or
sponsored by Valve Corporation.**

IceBox, IceBoxEngine and IceBoxCrew are marks of IceBoxCrew Studio.

## Third-party components

The plugin is compiled against **sol2**, **Lua**, **nlohmann/json** and, optionally,
**fmt** — all permissively licensed, all header-based, all compiled into the plugin
library. Their full notices are in
[THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt). **Keep that file in the plugin
folder** — it travels with the plugin into your game and it is what satisfies those
components' attribution requirements.

## IceBox Engine

The plugin is an Extension under Section 6 of the *IceBox Engine License Agreement*,
which permits publishing, distributing and selling Extensions. It contains no engine
source, binaries or core libraries, and it was compiled against exactly one engine
header, `PluginInterface.h`.

Using this plugin requires your own licensed copy of IceBox Engine. Receiving it
gives you no license to the engine.

A plugin build is tied to an engine version — both link the same Lua. Each release
states which engine version it was built for; on a mismatch the plugin declines to
register the `Storefront` table and says so in the log instead of crashing the editor.

---

<sub>This notice is a plain-language summary. [LICENSE.txt](LICENSE.txt) and
[THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt) are the operative documents and
govern if anything here reads differently. Nothing in this package is legal
advice.</sub>
