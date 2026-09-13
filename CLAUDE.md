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

### Build the right target — this is the difference between 24 seconds and many minutes

Measured on this machine, 2026-09-13, nothing changed between the two runs:

| target | use it for | no-op cost |
|---|---|---|
| `dunecity` | **everyday testing** — this is the game | **24 s** |
| `installer` | producing a setup file for other people | **86 s** |

To test a change, build `dunecity` and run the binary directly:

```
cmake --build <build folder> --target dunecity --config Release
<build folder>\bin\Release\dunecity.exe
```

`installer` additionally **deletes** the whole `install\` folder, re-copies everything into it
(including 340 MB of HD mod graphics), and runs NSIS compression — at full cost even when
nothing changed. That is the extra ~62 seconds, and none of it helps test a code change.

Note what this does **not** explain: a slow build is usually a full recompile after an
upstream merge, or vcpkg rebuilding dependencies. Check those before blaming the target.

`BUILD.md` shouts "DO NOT just run `cmake --build build`" and demands the installer target.
That advice is written for people cutting releases, which is upstream's job, not this fork's.
Ignore it for local work.

**Close the game before building.** Windows will not delete a folder containing a running
`.exe`, so `installer` fails at its first step with `Error removing directory .../install`.
Check with `Get-Process dunecity` before assuming a lock is Dropbox.

### What a healthy build looks like

- Warm vcpkg cache: configure ~1.5 minutes. Cold, or after upstream adds a dependency: 8–25
  minutes. (1.0.682 added OpenSSL 3.6 and libcurl 8.17 for crossplay — that configure took 24
  minutes and is now cached.)
- A full compile after a large upstream merge takes ~10 minutes. Everyday incremental builds
  do not.
- The C++ compiles with **zero errors and 5,000–6,000 warnings** on MSVC 2026. That is normal
  for this codebase on a newer compiler. **Do not "fix" the warnings as a side quest.**
- `--target installer` produces `DuneCity-<version>-Windows-x64.exe` in the build folder root.
  NSIS must be installed. `WINDOWS_QUICKSTART.md` claims CMake installs NSIS automatically —
  it does not.
- CMake 4.x prints a `CMP0177` policy warning from upstream's `install()` calls. Upstream's to
  fix, harmless here.

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

### Versioning — upstream's rules

The app version lives in **three files that must always agree**:

| file | line |
|---|---|
| `CMakeLists.txt` | `project(DuneCity VERSION 1.0.682 ...)` |
| `include/config.h` | `#define VERSION "1.0.682"` |
| `vcpkg.json` | `"version": "1.0.682"` |

`scripts/bump-version.sh` is the single source of truth for changing them:

```bash
scripts/bump-version.sh 1.0.683            # set all three
scripts/bump-version.sh --check            # verify they agree
scripts/bump-version.sh 1.0.683 --dry-run  # preview
```

**Never hand-edit the three files separately.** Upstream's own README shows `sed` commands
doing exactly that, followed by `--check`. Use the script instead — it does the same job
without the chance of missing one.

Upstream's rule: **the version bump belongs in the same commit as the work**, never a
follow-up commit. A tag `vX.Y.Z` must find the source already at `X.Y.Z`. Their CI verifies
this and will not bump for you.

Generated outputs (`build/include/config.h`, app bundle `Info.plist`) are derived at build
time and are not the source of truth.

**`python3` on Windows — fixed, but know why.** The script calls `python3`, which Windows
installers never create; the name resolves to a Microsoft Store alias stub that fails with
"Python was not found". Fixed on 2026-09-13 by copying `python.exe` to `python3.exe` in
`%LOCALAPPDATA%\Programs\Python\Python310\`, which sits earlier in PATH than the stub. The
script now reports `OK - all files agree`.

Do **not** edit upstream's script to work around this — editing an upstream file guarantees
a merge conflict later. Fix the environment, not their code.

If `python3` ever breaks again (new machine, Python reinstall), either redo that copy or
check the three files directly:

```bash
grep -m1 'project(DuneCity VERSION' CMakeLists.txt
grep -m1 'define VERSION' include/config.h
grep -m1 '"version"' vcpkg.json
```

The sprite pipeline (`scripts/import-sprites.py`, `scripts/make-tornie-pak.py`) needs Python
too. `py` gives 3.13.5 if a newer interpreter is wanted than the 3.10.11 behind `python3`.

**In this fork**, none of the release machinery applies — this fork publishes nothing, has no
CI, and must never push tags. The version here is only a marker of which upstream code the
fork sits on. Record it in `FORK-CHANGELOG.md` entries; do not bump it for local work.

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

## 6. Upstream: syncing

Upstream is `ggtothemax/dunecity` (Stefan van der Wel). The project was transferred there
from `VR48/dunecity` around 2026-09-12; VR48 remains a maintainer. Old URLs still redirect.

Upstream ships releases most days — 210 commits in the three days after this fork was made.
Keep `main` as a pure mirror and merge it into `my-dev`:

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git switch my-dev
git merge main
```

`--ff-only` on `main` is deliberate: if it refuses to fast-forward, something was committed
to `main` by mistake, and that must be investigated rather than merged over.

`.gitattributes` marks `CLAUDE.md`, `AGENTS.md` and `FORK-CHANGELOG.md` as `merge=ours`, so
this fork's versions survive untouched and produce no conflicts.

That protection needs a one-time local setup per clone, because merge drivers live in local
config and cannot be committed. If it is missing, run:

```bash
git config merge.ours.driver true
```

Verify with `git check-attr merge CLAUDE.md` — it must report `merge: ours`.

Everything else merges normally, so expect conflicts in source files this fork has touched.

## 7. Contributing back to upstream

Upstream accepts outside pull requests — merged PRs exist from contributors outside the core
team. Channels, best first:

| channel | for |
|---|---|
| Discord — `https://discord.com/invite/6sAcZr6y3B` | saying hello, asking what needs doing |
| GitHub Issues on `ggtothemax/dunecity` | concrete bugs and proposals |
| Pull request | the actual contribution |

There is no `CONTRIBUTING.md`. The **"Development Workflow"** section of upstream's
`README.md` is the closest thing, and it is the file to re-read before a first PR.

### The hard rule: never open a PR from `my-dev`

`my-dev` carries this fork's private work — `CLAUDE.md`, `AGENTS.md`, `FORK-CHANGELOG.md`,
`.gitattributes` merge rules, and the deletion of upstream's stale `build.bad/`, `build2/`,
`build_phase4/`, `buildtests/` folders. A PR from that branch would bury the real change
under noise and would be rejected on sight.

Every contribution starts from a **fresh branch off `main`**, which mirrors upstream exactly:

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git switch -c contrib/<short-name> main
```

Work there, commit only the files the change needs, then `git push -u origin contrib/<name>`
and open the PR on GitHub against `ggtothemax/dunecity`. Never merge `my-dev` into a
`contrib/` branch, and never merge a `contrib/` branch into `main`.

### Open question: do contributors bump the version?

Upstream's rule says a version bump belongs in the same commit as the work. That rule is
written for maintainers pushing to `main`. For an outside PR it is ambiguous, and bumping
would collide with every other open PR. **Ask on Discord or in the PR description rather
than guessing.** Default to not bumping, and say so in the PR.
