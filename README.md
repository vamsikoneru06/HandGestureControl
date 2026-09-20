# Hand Gesture Touchpad

Webcam-based hand gesture control that replaces touchpad interaction on
Windows: point to move the cursor, pinch to click/drag, and use fixed hand
poses for copy/paste, scroll, browser back/forward, and tab switching.

## Setup

    python -m venv venv
    source venv/Scripts/activate   # Git Bash; use venv\Scripts\activate.bat on cmd.exe
    pip install -r requirements.txt
    python main.py

Press `q` in the camera window to quit.

## Building a standalone .exe (for users without Python)

To hand this to someone who shouldn't have to install Python or run pip,
package it into a single Windows executable with
[PyInstaller](https://pyinstaller.org/):

    pip install pyinstaller
    pyinstaller HandGestureTouchpad.spec

This produces `dist/HandGestureTouchpad.exe` — a single file (roughly
250 MB, since it bundles Python, MediaPipe, and OpenCV) that runs on any
Windows machine with no separate install. Give that one file to users; they
double-click it to run. It opens a console window alongside the camera
window — don't close the console, since closing it exits the app, and any
startup errors (e.g. no camera found) print there.

`HandGestureTouchpad.spec` is checked into the repo as the build recipe, so
rebuilding after code changes is always just the one `pyinstaller` command
above — no need to remember the underlying flags. If you add a new
dependency, `pip install` it into the venv first so PyInstaller can find it.

## Gesture cheat sheet

| Gesture | Action |
|---|---|
| Index finger only, hand moves | Move cursor |
| Thumb+index pinch, held/released | Click (no movement) / drag (moved while pinched) |
| Closed fist | Copy (Ctrl+C) |
| Open palm (5 fingers) | Paste (Ctrl+V) |
| Index+middle, fast horizontal swipe | Back / Forward (Alt+Left / Alt+Right) |
| Index+middle, slow vertical motion | Scroll |
| Four fingers (no thumb), horizontal slide | Previous / next tab |
| F9 | Toggle gesture control on/off |

## Calibration notes

All thresholds live in `config.py`:

- `CONTROL_REGION` — the sub-rectangle of the camera frame that maps to the
  full screen. Narrow it if you have to reach too far to hit screen edges;
  widen it if the cursor feels too sensitive near your hand's resting position.
- `POINTER_SMOOTHING_ALPHA` / `CURSOR_SMOOTHING_ALPHA` / `CURSOR_DEADZONE` —
  two-stage cursor smoothing. `POINTER_SMOOTHING_ALPHA` filters the raw hand
  landmark before it's mapped to screen pixels; `CURSOR_SMOOTHING_ALPHA`
  smooths again in screen space. Lower either one (more smoothing, more lag)
  if the cursor is jittery/shaky; raise either (less smoothing, more
  responsive) if it feels laggy. Raise `CURSOR_DEADZONE` if it still drifts
  at rest.
- `PINCH_DISTANCE_THRESHOLD` — lower if pinch triggers too easily, raise if
  it's hard to trigger.
- `POSE_CONFIRMATION_FRAMES` — raise if wrong gestures fire during pose
  transitions; lower if gestures feel laggy to register.
- `*_COOLDOWN_SECONDS`, `SWIPE_*`, `SCROLL_*`, `TAB_SWITCH_*` — tune based on
  false positives/negatives observed in the verification pass below.

Run with the camera window open and watch the "Pose: ..." overlay label
while calibrating — it shows exactly what the classifier sees.

## Known limitations

- **Windows only** — uses `ctypes.windll.user32.GetSystemMetrics` for screen
  size and Windows keyboard conventions (Alt+Left/Right for browser
  back/forward).
- **Global hotkey may need Administrator** — the `keyboard` library's
  system-wide hotkey hook can require running the terminal as Administrator
  on some Windows configurations. If F9 doesn't respond, try that.
- **Lighting/background sensitive** — MediaPipe's hand tracking accuracy
  degrades in low light or busy backgrounds. Use the debug overlay to check
  tracking quality; this isn't addressed robustly in v1.
- **Single hand only** — the first detected hand per frame is used; no
  two-handed gestures.
- **The packaged .exe is unsigned** — Windows SmartScreen or antivirus
  software may flag `HandGestureTouchpad.exe` as unrecognized on first run,
  since it isn't code-signed. This is expected for an unsigned PyInstaller
  build, not a sign of tampering; users will need to click "More info" ->
  "Run anyway" (or whitelist it) to proceed. Code-signing would resolve this
  but requires a paid certificate.

## Verification plan

1. `pip install -r requirements.txt`, run `python main.py`.
2. With the overlay on, confirm the pose label is correct for: fist, open
   palm, pointing, two-finger, four-finger, pinch.
3. Press F9: confirm the overlay's ON/OFF state flips and all actions are
   gated by it.
4. End-to-end pass with a text editor + multi-tab browser open: move the
   cursor, click, drag-select text, scroll a page, copy (fist) then paste
   (open palm) elsewhere, swipe back/forward in browser history, and slide
   four fingers to switch tabs.
5. Tune `config.py` thresholds based on any false positives/negatives
   observed in step 4.
