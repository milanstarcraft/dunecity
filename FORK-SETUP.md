# FORK-SETUP.md — rules for AI agents working in this fork

**Read this file before `CLAUDE.md` or `AGENTS.md`.** This is a fork. Those two files
belong to upstream and describe *other people's machines*.

---

## 1. Where you are

| | |
|---|---|
| This fork | `milanstarcraft/dunecity` — the owner's fork, remote `origin` |
| Upstream | `VR48/dunecity` — remote `upstream`, **read-only** |
| Working branch | `my-dev` — all local work goes here |
| `main` | clean mirror of upstream, never commit to it |

The push URL for `upstream` is deliberately set to an invalid value so `git push upstream`
fails loudly instead of attempting to write to someone else's repository.

**Never open a pull request against upstream and never push to it.** Nothing goes to VR48
unless the owner asks for it explicitly.

## 2. Upstream's agent docs are wrong for this machine

`CLAUDE.md` and `AGENTS.md` are written by upstream for their own setups:

- `CLAUDE.md` assumes the checkout is at `~/development/dunecity`
- `AGENTS.md` assumes macOS with Homebrew and states **"no vcpkg, no Xcode"**

This fork is developed on **Windows with vcpkg and Visual Studio**. Do not follow their
build instructions. Use section 3 below.

Their **architecture** constraints in `CLAUDE.md` (2x2 zone footprint, roads as tile flags,
global power, save/load compatibility) still apply — those are about the code, not the machine.
Read them before touching game logic.

## 3. Building on this machine

Real paths live in `LOCAL-PATHS.md`, which is intentionally not committed. Read it first.
If it is missing, ask the owner rather than guessing.

Shape of the build:

```
cmake -S <source> -B <build folder> -G "Visual Studio 18 2026" -A x64 \
      -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake \
      -DVCPKG_INSTALLED_DIR=C:/vcpkg-installed/dunecity

cmake --build <build folder> --target installer --config Release
```

Use the CMake bundled with Visual Studio 2026 (version 4.x). The CMake bundled with
Build Tools 2019 is 3.20 and does not know the `Visual Studio 18 2026` generator.

### Two hard rules, both learned the painful way

**Never build inside the Dropbox folder.** Dropbox keeps file handles open on the tree and
build steps fail at random with `Device or resource busy` or `Error removing directory`.
The same command then succeeds by hand seconds later. Adding the folder to
`rules.dropboxignore` does **not** fix it — Dropbox must still watch a folder to apply a
rule, and the watching is what locks files. The build folder lives on a different drive,
outside Dropbox. Keep it that way.

**Never let vcpkg's install directory contain square brackets.** CMake treats `[` and `]`
as special characters and vcpkg writes absolute paths into the config files it generates.
A bracket in the path produces a misleading failure that blames a library, not the folder:

```
ninja: error: 'SDL2::SDL2-NOTFOUND', needed by 'SDL2_mixerd.dll'
```

This is why `-DVCPKG_INSTALLED_DIR=C:/vcpkg-installed/dunecity` is always passed. The build
folder itself may contain brackets; the vcpkg install dir may not.

### Expected results

- A first configure with a cold vcpkg cache takes about 8 minutes; warm, about 1.5 minutes.
- The C++ compiles with **zero errors** and roughly **5,000 warnings** on MSVC 2026.
  That is normal for this codebase on a newer compiler. Do not "fix" them as a side quest.
- `--target installer` produces `DuneCity-<version>-Windows-x64.exe` in the build folder root.
  It needs NSIS installed. `WINDOWS_QUICKSTART.md` claims CMake installs NSIS automatically —
  it does not.

## 4. "do M" — the maintenance action

When the owner types **"do M"** or asks for a commit, perform all of these, in order:

1. **Commit** the work to `my-dev` with a clear message describing *why*, not just what.
2. **Add a changelog entry** to `FORK-CHANGELOG.md` — newest first, using Belgrade local
   time and the upstream version the work sits on:
   `## 2026-09-10 20:56 Belgrade — on upstream 1.0.630`
3. **Update any documentation the change affects**, including this file if a rule changed.
4. **Push** to `origin` (`git push`). The fork is public — never commit secrets, tokens,
   or personal data.

Never run "do M" just because a task finished. It happens only when asked.

Recommended but not required: if you changed C++, build before committing. A broken commit
on a public fork is visible to everyone.

## 5. Keeping merges with upstream clean

Upstream ships releases most days. Every upstream file you edit becomes a future merge
conflict, so **prefer creating new files over editing upstream's**.

Files owned by this fork, safe to edit freely:

- `FORK-SETUP.md` (this file)
- `FORK-CHANGELOG.md`
- `LOCAL-PATHS.md` (not committed)

To pull upstream's latest:

```bash
git fetch upstream
git switch main
git merge upstream/main
git push
git switch my-dev
git merge main
```

Do this **before** starting significant work, not after — merging into unfinished work is
how afternoons disappear.

## 6. Working agreements

- **Q&A mode**: if the owner's message contains `q:` or `Q:`, answer only. Make no file changes.
- **Scope discipline**: do exactly what was asked. Flag unrelated problems you notice instead
  of fixing them unprompted.
- **Every source change ends up in git.** When you finish, `git status` should show no
  untracked `.h`/`.cpp`/`.py` files. Upstream's `AGENTS.md` documents a real release
  (1.0.599) built from 88 modified plus 23 untracked files — do not repeat that.
- **Temporary scripts** start with `// TEMPORARY UTILITY: [purpose]. Date: [date]` and are
  deleted after use. **Reusable scripts** use `// REUSABLE UTILITY: [purpose]. Created: [date].
  Last used: [date]` and are kept — update `Last used:` whenever you run one.
- **Agent scratch notes** stay out of the repo root. `.gitignore` already excludes
  `/AI-*.md` and `/*-REVIEW.md`. Anything worth keeping goes in `FORK-CHANGELOG.md`
  or under `docs/`.
- **Do not hand-edit generated or imported assets** unless the task is specifically about
  the asset pipeline.
