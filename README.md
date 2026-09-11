# DriftwoodXI — client distribution

This repository is what a DriftwoodXI player's launcher talks to. It is public
so it can be read without credentials; it holds no server code and no game data.

Two things live here, on two mechanisms that deliberately do not touch:

| What | Where | Read by |
|---|---|---|
| **Client content** — the DriftwoodXI Ashita addons | tracked files: `content.json` + `packages/` | the launcher's content sync |
| **Ashita bundle** — a complete Ashita v4 for new installs, and its **core set** for existing ones | tracked files: `ashita.json` + `bundles/` | "Install Ashita for me", and the content sync's core lane (launcher ≥ 0.33.0) |
| **Launcher installers** | GitHub **Releases** (`latest.yml` + `DriftwoodXI-Setup-*.exe`) | electron-updater |

**Do not move the content feed into release assets.** electron-updater reads
whichever release is newest and expects `latest.yml` inside it. A release
carrying content would sooner or later be the newest one, and every installed
launcher would start looking for a launcher build that is not in it — silently,
because a failed background update check is deliberately never shown to
players. Committed files and releases coexist here with no such coupling.

## What a player actually downloads

Once, by hand, from the website: the launcher installer and the DriftwoodXI
Ashita bundle. After that, nothing by hand — the launcher updates itself from
the Releases here and keeps the addons current from `content.json`.

No FFXI game files are distributed here or anywhere else by this project. The
launcher verifies a player's own retail install against published hashes and
reports what differs; it never downloads or repairs game data.

## content.json

```jsonc
{
  "schema": 1,                     // the launcher refuses anything it doesn't know
  "contentVersion": "2026.07.31",  // label for this generation, shown in Settings
  "publishedAt": "…",              // ISO timestamp
  "minLauncherVersion": "0.4.0",   // optional; see "the one dangerous field"
  "packages": [
    {
      "id": "dwbags",              // Ashita load name = the folder it installs to
      "kind": "ashita-addon",
      "version": "1.9",            // from the addon's own Lua header, for humans
      "url": "https://raw.githubusercontent.com/…/packages/dwbags-18a19866cd61.zip",
      "sha256": "…",               // of the zip; refused on mismatch
      "size": 26992,               // exact byte count; also checked
      "treeHash": "…",             // hash of the extracted tree — the real version
      "summary": "…"               // optional one-liner for the UI
    }
  ]
}
```

`treeHash`, not `version`, decides whether a player needs an update: it hashes
every file in the addon folder, so a payload republished without a version bump
still reaches everybody, and a locally damaged addon is repaired by the same
comparison that installs a new one.

### Payloads are content-addressed

`packages/<id>-<first 12 of treeHash>.zip`. raw.githubusercontent caches for a
few minutes, so a stable filename could be served stale beside a fresh manifest
and every checksum would fail. A new manifest names files that were never
cached, so that cannot happen.

Superseded payloads are **kept, not deleted**. A launcher that has not checked
in since the last publish is still holding the old manifest, and removing what
it points at turns a working install into a failed download. They cost a few
hundred KB per generation. `--prune-orphans` on the build script is the
housekeeping pass, and it is only safe long after a publish.

### The one dangerous field

`minLauncherVersion` makes a launcher update **mandatory**: below it a launcher
stops applying content, says why, and refuses to launch until it has updated
itself. It is the only thing in this system that can stand between a player and
their characters. Set it only when the addons genuinely need something older
launchers lack.

## ashita.json

```jsonc
{
  "schema": 1,
  "version": "20260910-2210",       // the bundle's build stamp
  "url": "…/bundles/DriftwoodXI-Ashita-20260910-2210-<sha12>.zip",
  "sha256": "…", "size": 43024557,  // the whole bundle, for NEW installs
  "core": {                         // launcher ≥ 0.33.0; older ones ignore it
    "version": "4.3.2.0",           // Ashita.dll's own FileVersion
    "url": "…/bundles/DriftwoodXI-AshitaCore-4.3.2.0-<sha12>.zip",
    "sha256": "…", "size": 20171972,
    "files": [                      // every file the zip carries, and may write
      { "path": "Ashita.dll", "sha256": "…", "size": 3751424 },
      { "path": "plugins/addons.dll", "sha256": "…", "size": 14285824 }
    ]
  }
}
```

The bundle is a bootstrap: "Install Ashita for me" downloads it once into a
fresh folder. The **core** block is the update channel for the Ashita files
inside every install that already exists — `Ashita.dll`, `Ashita-cli.exe`,
Ashita's two config inis, and the plugins and POL plugins Ashita itself ships.
It exists because a retail client patch can change a DAT layout under Ashita's
resource parser (2026-09-09, build 30260904_1: every addon read blank item
names until Ashita v4.3.2.0), and until then nothing could deliver a new
`Ashita.dll` to a player who had one.

The launcher reads `core` on the same rhythm as `content.json`, compares each
listed file's sha256 against the install, and replaces exactly the stale ones
before the next launch — the previous bytes kept under
`.driftwood-ashita-core.previous/`. Paths are held to an **allowlist** on
arrival (root binaries, `config/ashita/*.ini`, `plugins/*.dll`,
`polplugins/*.dll` — never `addons/`, `bootloader/`, DAT overlays or boot
inis), and a `core` block that fails any rule refuses the whole manifest, so
the publisher (`Publish-AshitaBundle.ps1`) re-checks the same rules before it
commits. Both zips are content-addressed and append-only like `packages/`.

## Publishing

Never by hand. From the server repo:

```powershell
M:\FFXI\LocalServer\server\PLANS\Publish-Content.ps1
```

It rebuilds the feed from `server\addons`, refuses to publish `dwgmpanel`
(staff tooling), verifies every zip round-trips to the hash written into the
manifest, commits and pushes. `deploy-driftwoodxi.ps1` calls it after a
successful deploy so the addons and the server modules they render move
together — server first, then content.

The generator lives in the launcher repo at `scripts/build-content.mjs`; the
schema it writes is mirrored by `src/shared/content.ts`, which re-validates
every field on arrival and drops anything it dislikes — including any attempt
to ship staff tooling.

## Safety, from the launcher's side

Everything here is treated as hostile data by the thing that reads it: package
ids are pattern-matched before they can become folder names, URLs must be
https, a download is refused unless its SHA-256 matches, it is expanded to a
temporary folder and re-hashed against `treeHash` before anything is replaced,
and the live folder is swapped last with the previous copy retained until the
new one is in place. A failure at any step leaves the player's existing addon
exactly as it was, and content being unreachable never blocks a launch.
