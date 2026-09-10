# CLAUDE.md — DuneCity (milanstarcraft fork)

Rules for AI agents working in this repository. **Read this file in full before any task.**

This file belongs to this fork and replaces upstream's version. It is protected by a
`merge=ours` rule in `.gitattributes`, so a future merge from upstream will never overwrite
it. Do not restore upstream's content here.

---

## 1. Project

A Dune Legacy fork: C++17 / SDL2 real-time strategy, with a Micropolis-style city
simulation built into the RTS game loop. The game ships with all original Dune II PAK
files in `data/`, so it runs without a copy of the 1992 game.

| | |
|---|---|
| This fork | `milanstarcraft/dunecity` — remote `origin` |
| Upstream | `VR48/dunecity` — remote `upstream`, **read-only** |
| Working branch | `my-dev` — all work goes here |
| `main` | mirror of upstream as it was at fork time; do not commit to it |

The push URL for `upstream` is deliberately invalid, so `git push upstream` fails loudly
instead of writing to someone else's repository.

**Never push to upstream and never open a pull request against it.** Nothing goes to VR48
unless the owner asks for it explicitly.

## 2. Build

Real paths are in `LOCAL-PATHS.md`, which is deliberately not committed. Read it first.
If it is missing, ask the owner rather than guessing.

```
cmake -S <source> -B <build folder> -G "Visual Studio 18 2026" -A x64 \
      -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake \
      -DVCPKG_INSTALLED_DIR=C:/vcpkg-installed/dunecity

cmake --build <build folder> --target installer --config Release
```

Use the CMake bundled with Visual Studio 2026 (4.x). The one bundled with Build Tools 2019
is 3.20 and does not know the `Visual Studio 18 2026` generator.

### Two hard rules, both learned the painful way

**Never build inside the Dropbox folder.** Dropbox holds file handles open on the tree and
build steps fail at random with `Device or resource busy` or `Error removing directory` —
then the same command succeeds by hand seconds later. Adding the folder to
`rules.dropboxignore` does **not** help: Dropbox must still watch a folder to apply a rule,
and the watching is what locks files. The build folder lives on another drive, outside
Dropbox. Keep it there.

**Never let vcpkg's install directory contain square brackets.** CMake treats `[` and `]`
as special characters, and vcpkg writes absolute paths into the config files it generates.
A bracket produces a failure that blames a library instead of the folder:

```
ninja: error: 'SDL2::SDL2-NOTFOUND', needed by 'SDL2_mixerd.dll'
```

Hence `-DVCPKG_INSTALLED_DIR=C:/vcpkg-installed/dunecity` on every configure. The build
folder itself may contain brackets; the vcpkg install dir may not.

### What a healthy build looks like

- Cold vcpkg cache: configure takes ~8 minutes. Warm: ~1.5 minutes.
- The C++ compiles with **zero errors and roughly 5,000 warnings** on MSVC 2026. That is
  normal for this codebase on a newer compiler. **Do not "fix" the warnings as a side quest.**
- `--target installer` produces `DuneCity-<version>-Windows-x64.exe` in the build folder root,
  and the game at `bin\Release\dunecity.exe`. NSIS must be installed.
  `WINDOWS_QUICKSTART.md` claims CMake installs NSIS automatically — it does not.

Upstream's `AGENTS.md` and `WINDOWS_QUICKSTART.md` describe other machines (macOS/Homebrew,
"no vcpkg", `~/development/dunecity`). Ignore their build instructions. Use this section.

## 3. "do M" — the maintenance action

When the owner types **"do M"** or asks for a commit, do all of this, in order:

1. **Commit** to `my-dev` with a message describing *why*, not just what.
2. **Add a changelog entry** to `FORK-CHANGELOG.md`, newest first, with Belgrade local time
   and the upstream version the work sits on:
   `## 2026-09-10 20:56 Belgrade — on upstream 1.0.630`
3. **Update any documentation the change affects**, including this file if a rule changed.
4. **Push** to `origin` with `git push`.

Never run "do M" just because a task finished. Only when asked.

**The fork is public.** Never commit secrets, tokens, credentials or personal data. Machine
paths belong in `LOCAL-PATHS.md`, which is excluded from Git.

Recommended, not required: if you changed C++, build before committing. A broken commit on
a public fork is visible to everyone.

## 4. Architecture constraints — these are facts about the code

Inherited from upstream and still true. Breaking them breaks the game.

- R/C/I zones stay **2x2 in gameplay footprint**. Do not convert simulation, placement or
  zoning rules to 3x3. Micropolis 3x3 art may be imported, but adapt it inside the 2x2
  gameplay footprint.
- Tiles remain the substrate for city state and overlays. Lot objects are a possible future
  architecture, not an opportunistic refactor.
- **Roads are player-built structures** but are special-cased in `House::placeStructure` to
  flip a tile flag (`Tile::isRoad_`) rather than spawn a `StructureBase` — so roads are not
  selectable. They appear in the build menu only in city-sim mode
  (`BuilderBase::updateBuildList`). Cost 10 credits, instant build.
- Concrete slabs (Slab1/Slab4) and roads are **independent tile states**. Concrete renders as
  concrete; roads render as an auto-tiled overlay on the underlying rock. They share no
  rendering or placement logic.
- **Power is global**, based on `producedPower >= powerRequirement` for the structure's owner.
  There is no per-tile grid and no power lines. `Tile::cityPowered_` exists in the save format
  but is unused; the old `cityConductive_` was renamed to `isRoad_`.
- Preserve deterministic simulation and save/load compatibility.

**The classic trap:** solving a 3x3 art or rendering problem by accidentally changing the
2x2 gameplay architecture. If a task tempts you to redesign zoning, stop and write down the
tradeoff first.

### Where the code lives

- City simulation: `include/dunecity/`, `src/dunecity/`, `Tile.{h,cpp}`, `Game.{h,cpp}`, `Command.{h,cpp}`
- Zones/build/render: `src/structures/ZoneStructure.cpp`, `ConstructionYard.cpp`,
  `BuilderBase.cpp`, `src/FileClasses/GFXManager.cpp`
- Sprite pipeline: `scripts/import-micropolis.py`, `scripts/import-sprites.py`, `imported_sprites/`
- Tests: `tests/`, `tests/CMakeLists.txt`

Deeper background: `ARCHITECTURE.md`, `docs/dunecity-current-architecture.md`, and
`analysis/` (historical, may be stale).

### Versioning

The app version lives in three files — `CMakeLists.txt`, `include/config.h`, `vcpkg.json` —
kept in sync by `scripts/bump-version.sh`. Never hand-edit them separately.

This fork does not publish releases, so upstream's tagging and CI rules do not apply here.
The version is only a marker of which upstream code this fork sits on.

## 5. Working agreements

- **Q&A mode**: if the owner's message contains `q:` or `Q:`, answer only. No file changes.
- **Scope discipline**: do exactly what was asked. Flag unrelated problems you notice instead
  of fixing them unprompted. Ask before expanding scope.
- **Prefer small, compileable changes.** Fix the direct cause before broad refactors.
- **Add or update tests** when touching placement, rendering lookup, command routing,
  save/load, or city-sim behaviour.
- **Every source change ends up in git.** When you finish, `git status` must show no
  untracked `.h`/`.cpp`/`.py` files. Upstream once shipped a release built from 88 modified
  plus 23 untracked files — do not repeat that.
- **Temporary scripts** start with `// TEMPORARY UTILITY: [purpose]. Date: [date]` and are
  deleted after use. **Reusable scripts** use `// REUSABLE UTILITY: [purpose]. Created:
  [date]. Last used: [date]` — update `Last used:` whenever you run one.
- **No agent scratch notes in the repo root.** `.gitignore` already excludes `/AI-*.md` and
  `/*-REVIEW.md`. Anything worth keeping goes into `FORK-CHANGELOG.md` or `docs/`.
- **Do not hand-edit generated or imported assets** unless the task is about the asset pipeline.

## 6. Upstream: merging is optional and rare

The owner does not routinely pull upstream changes. This fork has diverged on purpose.

If a merge is ever requested:

```bash
git fetch upstream
git merge upstream/main
```

`.gitattributes` marks `CLAUDE.md`, `AGENTS.md` and `FORK-CHANGELOG.md` as `merge=ours`, so
this fork's versions survive untouched and produce no conflicts.

That protection needs a one-time local setup per clone, because merge drivers live in local
config and cannot be committed. If it is missing, run:

```bash
git config merge.ours.driver true
```

Verify with `git check-attr merge CLAUDE.md` — it must report `merge: ours`.

Everything else merges normally, so expect conflicts in source files this fork has touched.
