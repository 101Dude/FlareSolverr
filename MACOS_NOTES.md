# macOS Notes – FlareSolverr

This branch contains macOS-specific fixes to improve FlareSolverr stability and
performance when running headless.

## Why this exists
- Prevent Chrome UI windows from appearing or flashing on macOS
- Avoid Xvfb / XQuartz entirely
- Speed up startup by avoiding spawning Chrome just to detect version
- Keep Chrome out of the Dock
- Make LaunchAgent usage stable

## What was changed
- `src/utils.py`
  - Prefer standard macOS Chrome app bundle paths
  - Read Chrome version from the app bundle `Info.plist`
  - Fallback to binary `--version` only if needed
  - Normalize version parsing for both plist and binary output
  - No DRIVER env var dependency (official branch behavior)

## What was NOT changed
- `flaresolverr_service.py` (no longer required)
- No fork-only logic
- No Windows/Linux behavior altered

## LaunchAgent notes
- HEADLESS=true
- PYTHONWARNINGS used to suppress `resource_tracker` shutdown noise
- DRIVER env var is no longer required on official master

## How to update when upstream changes
```bash
git fetch upstream
git switch john/macos-headless-fixes
git rebase upstream/master
git push --force-with-lease

---
If conflicts occur, they are expected to be limited to src/utils.py.

Known behavior
	•	Chrome does not appear in Dock
	•	No visible browser windows
	•	FlareSolverr only launches Chrome when handling requests
	•	Startup test still logs once at launch (expected)

Last verified working: 2025-12-12
