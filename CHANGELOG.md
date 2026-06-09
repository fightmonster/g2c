# Changelog

All notable changes to `g2c` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/).

## [1.0.0] — 2026-06-09

### Added
- Initial public release.
- 50+ commands across `auth` / `config` / `user` / `me` / `list-repo` / `list-branch` / `change` / `review` / `batch` / `repo` / `server`.
- AI Agent friendly: `--json` envelope, `g2c help` bundles the full reference manual, default home view exposes doc path.
- Write-op protection protocol: `batch *` and `repo conflict-continue` require explicit `--yes`.
- Local conflict handling chain: `conflict-start → conflict-read → conflict-resolve → conflict-continue` with `abort` for recovery.
- `repo cherry-pick` via Gerrit REST (returns `mode: "gerrit-rest"`).
- Auto-pagination for `change list` and `me` (default range 1000).
- Bundled reference manual at `docs/HELP.md` (25 KB, Chinese), shipped via `package.json#files`.
- 24 passing tests (unit + integration).

### Notes
- Source code lives at the private mirror [`fightmonster/g2c-src`](https://github.com/fightmonster/g2c-src).
- Install: `npm install -g g2c-1.0.0.tgz` (from the release asset below).
