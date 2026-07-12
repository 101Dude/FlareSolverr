# FlareSolverr on macOS — Local Headless Configuration

This branch preserves the macOS-specific changes used on John’s Mac Studio to run FlareSolverr as a user service without Xvfb, XQuartz, visible Chrome windows, or a root helper.

## Current working state

- Repository: `/Users/john/GitHub/FlareSolverr`
- Branch: `john/macos-headless-fixes`
- Service label: `org.flaresolverr`
- LaunchAgent: `~/Library/LaunchAgents/org.flaresolverr.plist`
- Listener: `127.0.0.1:8191`
- FlareSolverr version last tested: `3.4.6`
- Python version last observed: `3.12.6`
- Browser: Google Chrome
- Browser mode: native Chrome headless mode
- Last verified working: `2026-07-11`

## Why this branch exists

Older macOS setups used Xvfb from XQuartz. Xvfb required `/tmp/.X11-unix` to exist with suitable permissions, so a root LaunchDaemon named `com.johnprokos.mkdirX11` created that directory at startup.

That workaround is no longer used. The current branch runs Chrome in native headless mode on macOS and does not require:

- Xvfb
- XQuartz at runtime
- `/tmp/.X11-unix`
- a root FlareSolverr process
- the old `com.johnprokos.mkdirX11` LaunchDaemon

The obsolete LaunchDaemon was removed after confirming that FlareSolverr could complete a real browser request without it.

## Local source changes

The macOS-specific changes are in `src/utils.py`.

They are intended to:

- use native Chrome headless mode on macOS;
- avoid starting Xvfb on macOS;
- prevent Chrome windows from appearing or flashing;
- keep Chrome out of the Dock;
- prefer standard macOS Chrome application paths;
- read the Chrome version from the application bundle `Info.plist` when possible;
- fall back to the browser binary’s `--version` output only when necessary;
- preserve upstream behavior on Linux and Windows.

Relevant local commits include:

- `568b39a` — native Chromium headless on macOS; avoid Xvfb
- `1aa8ebb` — avoid launching Chrome solely for version detection; improve headless stability
- `f3479b3` — macOS maintenance notes
- `7ad0be9` — reusable LLM context and documentation

## LaunchAgent configuration

The service runs as the logged-in user, not as root.

Important environment values include:

- `HEADLESS=true`
- `DISPLAY=`
- `PYTHONWARNINGS` to suppress known `resource_tracker` shutdown noise

The LaunchAgent starts:

```text
/Users/john/GitHub/FlareSolverr/.venv/bin/python
/Users/john/GitHub/FlareSolverr/src/flaresolverr.py
```

## Service commands

Check whether the service is loaded:

```bash
launchctl print gui/$(id -u)/org.flaresolverr
```

Restart the service:

```bash
launchctl kickstart -k gui/$(id -u)/org.flaresolverr
```

Inspect the running process:

```bash
ps axww -o pid,ppid,user,start,command | grep -i '[f]laresolverr'
```

## Functional verification

Use this test after any update or service change. It prints only the important fields and does not dump the returned HTML:

```bash
curl -sS -X POST http://127.0.0.1:8191/v1 \
  -H 'Content-Type: application/json' \
  --data '{"cmd":"request.get","url":"https://www.google.com/","maxTimeout":60000}' \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); print("status:", d.get("status")); print("version:", d.get("version")); print("url:", d.get("solution",{}).get("url"))'
```

Expected result:

```text
status: ok
```

A successful result confirms that FlareSolverr can launch Chrome and complete a browser request using the native-headless path.

## Safe update procedure

Do not reset this branch to upstream or update it in place without preserving the local commits.

First confirm the working tree is clean and fetch upstream:

```bash
cd /Users/john/GitHub/FlareSolverr
git status --short --branch
git fetch upstream
```

Create a temporary update branch from the known-working branch:

```bash
git switch john/macos-headless-fixes
git switch -c john/macos-headless-update-YYYYMMDD
```

Rebase onto current upstream:

```bash
git rebase upstream/master
```

Conflicts are most likely in `src/utils.py`. Preserve the macOS native-headless behavior while incorporating upstream changes.

If Python requirements changed, update the existing virtual environment from the project’s current dependency files rather than recreating it blindly.

Then restart the LaunchAgent and run the functional verification test above.

Only after the service and browser test succeed should the updated branch replace or be merged into `john/macos-headless-fixes`.

## Rollback

Before updating, record the current commit:

```bash
git rev-parse HEAD
```

If an update fails, return to the known-working branch:

```bash
git switch john/macos-headless-fixes
launchctl kickstart -k gui/$(id -u)/org.flaresolverr
```

## Documentation location

Keep this information in `MACOS_NOTES.md` rather than replacing upstream’s main `README.md`. This minimizes documentation conflicts during future upstream updates.

The machine-readable companion file is:

```text
LLM_CONTEXT_flaresolverr_macos_headless.json
```