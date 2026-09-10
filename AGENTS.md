# AGENTS.md — DuneCity (milanstarcraft fork)

**All agent rules live in [`CLAUDE.md`](CLAUDE.md). Read that file. It is the single source
of truth for this fork.**

This file exists only so agents that look for `AGENTS.md` by convention are pointed there.

Upstream's original `AGENTS.md` was removed in this fork. It described a macOS Homebrew
build with no vcpkg, on a checkout at `~/Documents/projects/dunecity` — none of which
applies here. This fork is built on Windows with vcpkg and Visual Studio 2026.

Both this file and `CLAUDE.md` are marked `merge=ours` in `.gitattributes`, so a merge from
upstream will never restore their versions.
