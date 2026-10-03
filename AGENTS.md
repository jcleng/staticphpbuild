# AGENTS.md

This repo contains no application code. Its only job is a GitHub Actions workflow
(`.github/workflows/main.yml`) that drives the external **spc** (static-php-cli)
binary to build a static PHP CLI and publish it to a GitHub Release.

## How the build actually runs
- Triggered only via `workflow_dispatch` (Actions tab → "Run workflow") or
  `repository_dispatch`. There is no local CLI, test, or build script to run here.
- The workflow downloads a **prebuilt** spc binary from `spc_url` (`curl … -o spc`),
  then runs it. This repo never compiles spc itself.
- Build command (from the workflow, do not improvise flags):
  `./spc build:php "<extensions>" --build-cli --dl-with-php=<ver> --dl-parallel=<n> --dl-retry=<n> --dl-ignore-cache=<cache> --dl-prefer-binary -vv`
- `./spc doctor --auto-fix` runs before the build to satisfy system deps.

## Defaults that differ across files (verify before trusting)
- **`php_version` default is `8.2` in the workflow**, but the README says `8.4`.
  Trust the workflow (`8.2`). If the README is wrong, fix it rather than copying it.
- The README's default `extensions` list is missing `swoole`; the workflow's list
  is authoritative.

## Extension-list gotchas
- `swoole` was **intentionally removed** from the default list (commit `9d6291d`):
  swoole 6.2 fails to compile against PHP 8.4 (`std::ap_php_snprintf`). Do not
  re-add it unless that is resolved.
- The full default extension string lives in the workflow `extensions` input;
  copy it verbatim rather than reconstructing it.

## Artifacts, release, and secrets
- On success the workflow zips `buildroot/bin/` and also extracts the standalone
  `buildroot/bin/php` binary, writes `release.txt` (params + sha256 checksums),
  and publishes a Release with tag `static-php_<ver>_<YYYYMMDDHHMMSS>`.
- Release publishing needs the `GITHUB_TOKEN` secret (provided by Actions); no
  extra token config is needed for the standard flow.
- **Debugging failures:** on a failed build the workflow prints and uploads
  `log/spc.output.log` and `log/spc.shell.log` as an artifact. Read those first.

## Repo conventions
- `.vscode/` is gitignored (editor config only — not meaningful to agents).
- Commit messages are bilingual (Chinese + English); match that style for consistency.
