# Hand Gesture Touchpad

## What it is

A Windows desktop app that turns your webcam into a full touchpad. It reads
your hand through the camera and translates a fixed set of poses and motions
into real mouse and keyboard input: point to move the cursor, pinch to
click and drag, and dedicated hand shapes for copy/paste, scrolling, browser
back/forward, and switching browser tabs — no physical mouse or keyboard
shortcuts required for any of it. An on-screen debug overlay shows the
tracked hand landmarks and the currently recognized pose live, and a global
hotkey (F9) instantly disables all gesture-driven input if you need your
hands back for typing.

It's a single-user, single-machine personal productivity tool — no
accounts, no network calls, nothing installed system-wide beyond what you
choose to run.

## Built with

- **Python 3.11**
- **[MediaPipe](https://developers.google.com/mediapipe)** (`mediapipe.solutions.hands`) — 21-point hand landmark tracking from the webcam feed
- **[OpenCV](https://opencv.org/)** (`opencv-python`) — webcam capture and the live debug overlay window
- **[pynput](https://pynput.readthedocs.io/)** — simulates mouse movement, clicks, drags, and scrolling
- **[keyboard](https://github.com/boppreh/keyboard)** — simulates keyboard shortcuts and provides the global F9 on/off hotkey
- **NumPy** — vector math for landmark distances and cursor smoothing
- **`ctypes`** (Python standard library) — reads Windows screen resolution via `GetSystemMetrics`
- **[PyInstaller](https://pyinstaller.org/)** — packages the whole app into the standalone `.exe` below
- **pytest** — unit tests for the pure gesture-classification logic (see `tests/`)

## How to use it

### Option A: Download the .exe (no Python required)

1. Download `HandGestureTouchpad.exe` from this repo.
2. Double-click it to run.
3. Windows SmartScreen will likely warn that it's from an unrecognized
   publisher, since it isn't code-signed — click **More info** -> **Run
   anyway** to proceed. This is expected for an unsigned build, not a sign
   of tampering.
4. Two windows open: a console (leave it open — closing it exits the app,
   and startup errors like "no camera found" print there) and a camera
   window showing your webcam feed with tracking dots and a pose label.
5. Hold your hand up in the camera window and try the gestures in the
   [cheat sheet](#gesture-cheat-sheet) below. Press **F9** any time to
   toggle gesture control off/on, and **`q`** in the camera window to quit.

Start over an empty text editor the first time, not your terminal or
anything important — it's driving your real mouse and keyboard from the
moment it recognizes a gesture.

### Option B: Run from source

    python -m venv venv
    source venv/Scripts/activate   # Git Bash; use venv\Scripts\activate.bat on cmd.exe
    pip install -r requirements.txt
    python main.py

Press `q` in the camera window to quit.

### Building your own .exe

To rebuild the standalone executable after changing the code:

    pip install pyinstaller
    pyinstaller HandGestureTouchpad.spec

This produces `dist/HandGestureTouchpad.exe` (roughly 250 MB, since it
bundles Python, MediaPipe, and OpenCV). `HandGestureTouchpad.spec` is the
checked-in build recipe, so rebuilding is always just that one command — no
need to remember the underlying PyInstaller flags. If you add a new
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
while calibrating — it shows exactly what the classifier sees. (Editing
`config.py` requires running from source — the packaged `.exe` bundles a
fixed copy of it.)

## Known limitations

- **Windows only** — uses `ctypes.windll.user32.GetSystemMetrics` for screen
  size and Windows keyboard conventions (Alt+Left/Right for browser
  back/forward).
- **Global hotkey may need Administrator** — the `keyboard` library's
  system-wide hotkey hook can require running as Administrator on some
  Windows configurations. If F9 doesn't respond, try that.
- **Lighting/background sensitive** — MediaPipe's hand tracking accuracy
  degrades in low light or busy backgrounds. Use the debug overlay to check
  tracking quality; this isn't addressed robustly in v1.
- **Single hand only** — the first detected hand per frame is used; no
  two-handed gestures.
- **The packaged .exe is unsigned** — see step 3 under "Download the .exe"
  above. Code-signing would resolve the SmartScreen warning but requires a
  paid certificate.

## Verification plan

1. `pip install -r requirements.txt`, run `python main.py` (or just launch
   the `.exe`).
2. With the overlay on, confirm the pose label is correct for: fist, open
   palm, pointing, two-finger, four-finger, pinch.
3. Press F9: confirm the overlay's ON/OFF state flips and all actions are
   gated by it.
4. End-to-end pass with a text editor + multi-tab browser open: move the
   cursor, click, drag-select text, scroll a page, copy (fist) then paste
   (open palm) elsewhere, swipe back/forward in browser history, and slide
   four fingers to switch tabs.
5. Tune `config.py` thresholds based on any false positives/negatives
   observed in step 4 (source checkout only).
