# AGENTS.md

This repo contains **no application code**. Its only job is a GitHub Actions workflow
(`.github/workflows/main.yml`) that drives the external **spc** (static-php-cli) binary
to build a static PHP CLI and publish it to a GitHub Release.

## How the build actually runs

- Triggered only via `workflow_dispatch` (Actions tab → "Run workflow") or
  `repository_dispatch`. There is no local CLI, test, or build script to run here.
- The workflow downloads a **prebuilt** spc binary from `spc_url`
  (`curl … -o spc && chmod +x spc`), then runs it. This repo never compiles spc itself.
- Build command (from the workflow, do not improvise flags):
  `./spc build:php "<extensions>" --build-cli --dl-with-php=<ver> --dl-parallel=<n> --dl-retry=<n> --dl-ignore-cache=<cache> --dl-prefer-binary -vv`
- `./spc doctor --auto-fix` runs before the build to satisfy system deps.

## Current default configuration (as of latest commit)

- **`php_version` default = `8.2`** (changed from `8.4` in commit `8535769`).
- **`swoole` is removed** from the default `extensions` list (commit `9d6291d`).
- On success, the workflow also extracts the standalone `buildroot/bin/php` binary and
  publishes it alongside the zip (commit `909618c`).

> ⚠️ **README is out of date**: `README.md` parameter table still says `php_version`
> default is `8.4`. The workflow (`8.2`) is authoritative. Fix the README rather than
> copying from it.

## Extension-list gotchas

- `swoole` was **intentionally removed** from the default list: it does NOT compile
  against the current spc nightly, on **either PHP 8.2 or 8.4** (see "Known failures"
  below). Do not re-add it unless that is resolved. The upstream issue is
  [crazywhalecc/static-php-cli#1246](https://github.com/crazywhalecc/static-php-cli/issues/1246).
- The full default extension string lives in the workflow `extensions` input; copy it
  verbatim rather than reconstructing it.

## Known build failures (debugging history)

These were diagnosed by adding `-vv` to the build and dumping/uploading
`log/spc.output.log` + `log/spc.shell.log` as artifacts on failure (commit `ca6264e`).

### 1. swoole 6.2.3 compile error (the reason swoole is disabled)

- Symptom: `make cli` fails compiling `ext/swoole/ext-src/swoole_admin_server.cc`.
- Error:
  ```
  ext/swoole/thirdparty/nlohmann/detail/input/binary_reader.hpp:273:23:
    error: no member named 'ap_php_snprintf' in namespace 'std'
  .../main/snprintf.h:99:18: note: expanded from macro 'snprintf'
     99 | #define snprintf ap_php_snprintf
  ```
- Root cause: PHP's `main/snprintf.h` does `#define snprintf ap_php_snprintf`. swoole
  vendors a copy of `nlohmann/json` that calls `(std::snprintf)(...)`, which expands to
  `std::ap_php_snprintf` — but `ap_php_snprintf` only exists in the global namespace,
  not `std`. This is a swoole-src bug (v6.2.3), reproduced identically on PHP 8.2 and
  8.4. Reported upstream: **static-php-cli#1246** (also relevant: swoole/swoole-src).
- Affected runs: `36967448557` (8.4), `37115276294` (8.2) — both failed.
- Workaround: remove `swoole` from the extension list.

### 2. libde265 GitHub API 403 during download (intermittent)

- Symptom: very early failure (~35s) at the download stage, before any compile:
  ```
  ✘ Download failed: Download artifact 'libde265' failed.
  Last failed command: curl ... 'https://api.github.com/repos/strukturag/libde265/releases/assets/...'
    - Exit code: 22  (curl: (22) The requested URL returned error: 403)
  ```
- Root cause: spc fetches some release assets via the unauthenticated GitHub API,
  which rate-limits / 403s. **Intermittent** — the same run config later succeeded past
  this stage (e.g. `37115276294` downloaded all 45 artifacts fine, only failing later
  on the swoole compile). Not a config bug; may need `GITHUB_TOKEN` / `SPC_GITHUB_TOKEN`
  to raise API quota if it persists.
- Affected runs: `36977687149` failed here; `37115276294` passed it.

### 3. What works

- Run `36970748509` (PHP 8.4, swoole already removed): **SUCCESS** (1h18m), proving
  the other 50+ extensions build and link fine. This is the template for a good build.

## Artifacts, release, and secrets

- On success the workflow zips `buildroot/bin/` into
  `static-php-cli-<ver>_<timestamp>.zip`, extracts the standalone `php-<ver>_<timestamp>`
  binary, writes `release.txt` (params + sha256 checksums of both files), and publishes
  a Release with tag `static-php_<ver>_<YYYYMMDDHHMMSS>`.
- Release publishing uses the `GITHUB_TOKEN` provided by Actions; no extra token config
  is needed for the standard flow.
- **Debugging failures:** on a failed build the workflow prints and uploads
  `log/spc.output.log` and `log/spc.shell.log` as an artifact named
  `spc-build-logs-<ver>_<timestamp>`. Download it with
  `gh run download <run_id> -n spc-build-logs-<ver>_<timestamp>` and grep
  `spc.shell.log` for `error:` / `ap_php_snprintf` / `swoole_admin_server` / `403`.

## Repo conventions

- `.vscode/` is gitignored (editor config only — not meaningful to agents).
- Commit messages are bilingual (Chinese + English); match that style for consistency.
- The local build dir `build-local/` that was used for experiments has been removed;
  do not recreate it — build only via GitHub Actions.

## Useful commands (run from repo root, with proxy if needed)

```bash
# list recent runs
gh run list --repo jcleng/staticphpbuild --limit 5

# view a failed run's step summary
gh run view <run_id> --repo jcleng/staticphpbuild

# pull the real compile/link error after a failure
gh run download <run_id> --repo jcleng/staticphpbuild -n spc-build-logs-<ver>_<timestamp>
grep -nE "error:|ap_php_snprintf|swoole_admin_server|403|libde265" spc.shell.log

# re-run only the failed jobs of a previous run
gh run rerun <run_id> --repo jcleng/staticphpbuild --failed
```
