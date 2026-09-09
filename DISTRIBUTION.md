# Shipping a game that uses this plugin

*Русская версия ниже — [Как выпускать игру с этим плагином](#как-выпускать-игру-с-этим-плагином).*

This file answers one question: **what goes into the build you upload, and what must
never.** Read it once before your first release.

The short version: the plugin folder travels with your game, Valve's runtime travels
with it, and your `steam_appid.txt` does not.

---

## The plugin folder, and what is in it

After you unpack the release and drop Valve's runtime beside it, `Plugins/Steam/`
looks like this:

```
Plugins/Steam/
├── Steam.dll / Steam.so / Steam.dylib   the plugin        — from us
├── steam_api64.dll / libsteam_api.so /
│   libsteam_api.dylib                   Steam runtime     — from Valve, you add it
├── plugin.json                          manifest          — from us
├── VisualScriptAPI.json                 node catalog      — from us
├── icon.png                             editor icon       — from us
├── steam_config.json                    your AppId        — you create it
├── LICENSE.txt  NOTICE.md
├── THIRD_PARTY_NOTICES.txt
├── README.md  DISTRIBUTION.md
└── Documentation/EN/…  Documentation/RU/…
```

Four of those files are **required at run time** and are easy to delete by mistake:

- **`Steam.dll` / `Steam.so` / `Steam.dylib`** — the plugin itself. One per platform;
  a `Steam.dll` is of no use to a Linux build.
- **`steam_api64.dll` / `libsteam_api.so` / `libsteam_api.dylib`** — Valve's runtime.
  It must sit **in the plugin folder**, beside the plugin library: that is the one
  location all three dynamic loaders agree on. On Windows the plugin loads it from
  there explicitly, on Linux the plugin carries an `$ORIGIN` RPATH, and on macOS
  `libsteam_api.dylib` is built with the install name `@loader_path/libsteam_api.dylib`
  and *must* be next to the library that links it. Do not move it.
- **`plugin.json`** — the manifest. Without it the engine does not see the folder as a
  plugin at all.
- **`VisualScriptAPI.json`** — the node catalog. It looks generated, and it is, but the
  editor reads it straight out of the plugin folder to fill the visual-script node
  palette. A build without it silently loses every `Storefront` node.

**`icon.png`** is what the editor's Plugins panel and the launcher draw next to the
plugin's name. Drop it and your plugin shows up as a blank tile.

**`LICENSE.txt`, `NOTICE.md` and `THIRD_PARTY_NOTICES.txt` stay in the folder.**
`THIRD_PARTY_NOTICES.txt` in particular is what satisfies the attribution that sol2,
Lua, nlohmann/json and fmt require — those are compiled into the plugin library, so the
notice has to travel with it. It is a few kilobytes.

**`Documentation/` and `README.md` may go.** About 400 KB of Markdown your players do
not need. Nothing breaks if you leave them; delete them from the staged build if
package size matters to you.

---

## The four things that are easy to get wrong

They produce no error when you get them wrong, which is what makes them worth a list.

### 1. Enable the plugin in `Config/Plugins.json`

The engine discovers every plugin folder but loads only the ones listed as enabled
there. Tick **Steam** once in the editor's **Tools → Plugins & Mods** — that writes the
file, and **Tools → Build Game** copies it into the package. Without it your game ships
the plugin and never loads it.

### 2. Do not ship `steam_appid.txt`

It tells Steam "assume this AppId, skip the launch check" — exactly what you want next
to the executable while testing, and exactly what you do not want in a release. Keep it
beside the built game during development and delete it from the uploaded build.

`steam_config.json`, on the other hand, **does** ship: it is where your released game
reads its AppId from.

### 3. Ship one plugin build per platform

The plugin links the same Lua the engine does, so a build is tied to an engine version
*and* a platform. Take the release archive that matches your engine version, and take
one archive per platform you publish on.

### 4. Remember the editor initializes Steam too

With a valid AppId configured, running the editor marks you as playing the game on
Steam and lets you unlock real achievements from Play mode. That is deliberate — it is
how you test the integration — but use AppId `480` or disable the plugin while you are
not testing Steam.

---

## What must never leave your machine

### Valve's files, outside your game

| File | Where it comes from |
|---|---|
| `steam_api.dll`, `steam_api64.dll` | `redistributable_bin/` |
| `libsteam_api.so`, `libsteam_api.dylib` | `redistributable_bin/linux64`, `/osx` |
| `steam_api.lib`, `steam_api64.lib` | `redistributable_bin/` |
| `sdkencryptedappticket.*` | `public/steam/lib/` |
| `steamclient*.dll`, `tier0_s.dll`, `vstdlib_s.dll` | Steam client libraries |
| `public/steam/*.h` | SDK headers |
| `tools/` — ContentBuilder, steamcmd, ContentServer, goldmaster, codesigning | SDK tools |
| `steamworksexample/`, `glmgr/` | SDK samples |

Inside **your released game**, `steam_api64.dll` is fine and expected: Section 1.1 of
your own *Steamworks SDK Access Agreement* lets you distribute the contents of
`redistributable_bin` together with your own application in object code form.

Anywhere else it is not. Valve licenses the SDK to *you*, personally and
**nontransferably**. You cannot pass that license on, so a Valve file must never be
inside anything you hand to another developer — a plugin folder, a template project, a
sample repository, a zip in a chat.

### Yours, but private

| File | Why |
|---|---|
| `steam_appid.txt` | Tells the Steam client to skip the launch check. Development only. |
| `steam_config.json` | Holds **your** AppId. It belongs inside your game and nowhere else — hand it to someone and their build reports as your game. |

### The plugin itself, on its own

The plugin is free, and it is not yours to republish. Uploading `Steam.dll` to another
site, putting it in an asset pack or a plugin bundle, shipping it inside a modified
engine distribution, or charging for it — none of that is permitted, whether or not
money changes hands. See Section 4.1 of [`LICENSE.txt`](LICENSE.txt).

Inside your game it travels freely; that is Section 2.2 and it is the whole point.
Someone else who wants the plugin gets it from the official download page, in about a
minute, for free — the same page you got it from.

---

## Keep the two situations apart

|  | Goes out with your **game** | Goes out to another **developer** |
|---|---|---|
| `Steam.dll` / `.so` / `.dylib` | ✅ required | 🚫 no — send them the download link |
| `plugin.json`, `VisualScriptAPI.json`, `icon.png` | ✅ required | 🚫 no |
| `LICENSE.txt`, `NOTICE.md`, `THIRD_PARTY_NOTICES.txt` | ✅ keep them | — |
| `Documentation/`, `README.md` | optional | — |
| `steam_api64.dll` / `libsteam_api.so` / `.dylib` | ✅ yes — Valve permits this | 🚫 **never** |
| `steam_config.json` | ✅ yes — your game reads its AppId from it | 🚫 never — it holds *your* AppId |
| `steam_appid.txt` | 🚫 never — delete it before you upload | 🚫 never |

---

<sub>Steam, Steamworks, Steam Deck, Steam Machine, Steam Frame, Steam Cloud, Steam
Workshop, Steam Input, Big Picture and Proton are trademarks and/or registered
trademarks of Valve Corporation, used here descriptively only. IceBoxStorefront
Plugin is not made by, not affiliated with, not endorsed by and not sponsored by
Valve Corporation. See `LICENSE.txt`, `NOTICE.md` and `THIRD_PARTY_NOTICES.txt`.
Nothing here is legal advice.</sub>

---
---

# Как выпускать игру с этим плагином

Этот файл отвечает на один вопрос: **что попадает в билд, который вы заливаете, и чего
там быть не должно.** Прочитайте один раз перед первым релизом.

Коротко: папка плагина уезжает вместе с игрой, рантайм Valve уезжает вместе с ней, а
ваш `steam_appid.txt` — нет.

---

## Папка плагина и что в ней лежит

После того как вы распаковали релиз и положили рядом рантайм Valve, `Plugins/Steam/`
выглядит так:

```
Plugins/Steam/
├── Steam.dll / Steam.so / Steam.dylib   плагин            — от нас
├── steam_api64.dll / libsteam_api.so /
│   libsteam_api.dylib                   рантайм Steam     — от Valve, кладёте вы
├── plugin.json                          манифест          — от нас
├── VisualScriptAPI.json                 каталог нод       — от нас
├── icon.png                             иконка в редакторе — от нас
├── steam_config.json                    ваш AppId         — создаёте вы
├── LICENSE.txt  NOTICE.md
├── THIRD_PARTY_NOTICES.txt
├── README.md  DISTRIBUTION.md
└── Documentation/EN/…  Documentation/RU/…
```

Четыре из этих файлов **обязательны во время выполнения** и их легко удалить по ошибке:

- **`Steam.dll` / `Steam.so` / `Steam.dylib`** — сам плагин. По одному на платформу:
  `Steam.dll` бесполезен для сборки под Linux.
- **`steam_api64.dll` / `libsteam_api.so` / `libsteam_api.dylib`** — рантайм Valve.
  Он должен лежать **в папке плагина**, рядом с библиотекой плагина: это единственное
  место, с которым согласны все три динамических загрузчика. На Windows плагин грузит
  его оттуда явно, на Linux плагин несёт RPATH `$ORIGIN`, а на macOS
  `libsteam_api.dylib` собран с install name `@loader_path/libsteam_api.dylib` и
  *обязан* находиться рядом с библиотекой, которая его линкует. Не переносите его.
- **`plugin.json`** — манифест. Без него движок вообще не считает папку плагином.
- **`VisualScriptAPI.json`** — каталог нод. Выглядит сгенерированным, и он такой и есть,
  но редактор читает его прямо из папки плагина, чтобы наполнить палитру нод
  визуального скриптинга. Билд без него молча теряет все ноды `Storefront`.

**`icon.png`** — то, что панель Plugins в редакторе и лаунчер рисуют рядом с именем
плагина. Уберёте — плагин будет пустой плиткой.

**`LICENSE.txt`, `NOTICE.md` и `THIRD_PARTY_NOTICES.txt` остаются в папке.**
`THIRD_PARTY_NOTICES.txt` — это то, что закрывает требования атрибуции sol2, Lua,
nlohmann/json и fmt: они вкомпилированы в библиотеку плагина, значит и уведомление
должно ехать вместе с ней. Это несколько килобайт.

**`Documentation/` и `README.md` можно убрать.** Около 400 КБ Markdown, которые вашим
игрокам не нужны. Ничего не сломается, если оставить; удалите из подготовленного билда,
если вам важен размер пакета.

---

## Четыре вещи, в которых легко ошибиться

Ошибка в каждой из них не даёт никакой ошибки на экране — потому список и нужен.

### 1. Включите плагин в `Config/Plugins.json`

Движок находит все папки плагинов, но грузит только те, что перечислены там как
включённые. Отметьте **Steam** один раз в **Tools → Plugins & Mods** — это запишет файл,
а **Tools → Build Game** скопирует его в пакет. Без этого ваша игра увезёт плагин и
никогда его не загрузит.

### 2. Не отгружайте `steam_appid.txt`

Он велит Steam «считай, что AppId такой, проверку запуска пропусти» — ровно то, что
нужно рядом с исполняемым файлом во время тестов, и ровно то, чего не должно быть в
релизе. Держите его рядом со сборкой во время разработки и удаляйте из заливаемого
билда.

`steam_config.json`, наоборот, **уезжает с игрой**: именно оттуда выпущенная игра читает
свой AppId.

### 3. По одной сборке плагина на платформу

Плагин линкует ту же Lua, что и движок, поэтому сборка привязана и к версии движка, и к
платформе. Берите архив релиза под свою версию движка — и по одному архиву на каждую
платформу, под которую вы публикуетесь.

### 4. Помните, что редактор тоже инициализирует Steam

При настроенном рабочем AppId запуск редактора отмечает вас как играющего в игру в Steam
и позволяет разблокировать настоящие достижения из режима Play. Это сделано намеренно —
так вы и тестируете интеграцию, — но пользуйтесь AppId `480` или отключайте плагин,
когда Steam вам не нужен.

---

## Что не должно уходить с вашей машины

### Файлы Valve — везде, кроме вашей игры

| Файл | Откуда он |
|---|---|
| `steam_api.dll`, `steam_api64.dll` | `redistributable_bin/` |
| `libsteam_api.so`, `libsteam_api.dylib` | `redistributable_bin/linux64`, `/osx` |
| `steam_api.lib`, `steam_api64.lib` | `redistributable_bin/` |
| `sdkencryptedappticket.*` | `public/steam/lib/` |
| `steamclient*.dll`, `tier0_s.dll`, `vstdlib_s.dll` | библиотеки клиента Steam |
| `public/steam/*.h` | заголовки SDK |
| `tools/` — ContentBuilder, steamcmd, ContentServer, goldmaster, codesigning | инструменты SDK |
| `steamworksexample/`, `glmgr/` | примеры SDK |

Внутри **вашей выпущенной игры** `steam_api64.dll` уместен и ожидаем: Раздел 1.1 вашего
собственного *Steamworks SDK Access Agreement* разрешает распространять содержимое
`redistributable_bin` вместе с вашим приложением в объектном коде.

Везде в другом месте — нет. Valve лицензирует SDK лично *вам* и **без права передачи**.
Передать эту лицензию дальше нельзя, поэтому файл Valve не должен оказаться ни в чём,
что вы отдаёте другому разработчику: ни в папке плагина, ни в шаблонном проекте, ни в
репозитории с примером, ни в архиве в чате.

### Ваше, но приватное

| Файл | Почему |
|---|---|
| `steam_appid.txt` | Велит клиенту Steam пропустить проверку запуска. Только разработка. |
| `steam_config.json` | Содержит **ваш** AppId. Его место — внутри вашей игры и больше нигде: отдадите — чужая сборка будет отчитываться как ваша игра. |

### Сам плагин — отдельно от игры

Плагин бесплатный, но перезаливать его нельзя. Выложить `Steam.dll` на другой сайт,
положить его в пак ассетов или в бандл плагинов, увезти его внутри изменённой сборки
движка, брать за него деньги — ничего из этого не разрешено, независимо от того, идут
ли деньги. См. Раздел 4.1 в [`LICENSE.txt`](LICENSE.txt).

Внутри вашей игры он ездит свободно — это Раздел 2.2, и в этом весь смысл. Тот, кому
плагин нужен, берёт его на официальной странице загрузки: минута времени, бесплатно, та
же самая страница, с которой взяли вы.

---

## Держите две ситуации раздельно

|  | Уезжает с вашей **игрой** | Уезжает другому **разработчику** |
|---|---|---|
| `Steam.dll` / `.so` / `.dylib` | ✅ обязательно | 🚫 нет — дайте ссылку на загрузку |
| `plugin.json`, `VisualScriptAPI.json`, `icon.png` | ✅ обязательно | 🚫 нет |
| `LICENSE.txt`, `NOTICE.md`, `THIRD_PARTY_NOTICES.txt` | ✅ оставьте на месте | — |
| `Documentation/`, `README.md` | по желанию | — |
| `steam_api64.dll` / `libsteam_api.so` / `.dylib` | ✅ да — Valve это разрешает | 🚫 **никогда** |
| `steam_config.json` | ✅ да — игра читает оттуда AppId | 🚫 никогда — там *ваш* AppId |
| `steam_appid.txt` | 🚫 никогда — удалите перед заливкой | 🚫 никогда |

---

<sub>Steam, Steamworks, Steam Deck, Steam Machine, Steam Frame, Steam Cloud, Steam
Workshop, Steam Input, Big Picture и Proton — товарные знаки и/или зарегистрированные
товарные знаки Valve Corporation, используемые здесь исключительно описательно.
IceBoxStorefront Plugin не создан Valve Corporation, не аффилирован с ней, не одобрен
и не спонсируется ею. См. `LICENSE.txt`, `NOTICE.md` и `THIRD_PARTY_NOTICES.txt`.
Ничто здесь не является юридической консультацией.</sub>
