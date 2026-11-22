# CameraKey WiFi

Turn an Android phone camera into a configurable, wireless keyboard detector for a Windows or Linux PC.

**Current prototype:** fixed-camera, manual per-key calibration, grayscale image-difference detection, multi-key events, and a WebSocket PC receiver. The project is intentionally dependency-light and does not use cloud AI or OCR.

> This is an experimental prototype, not a finished replacement for a physical keyboard. It detects image changes over calibrated key locations. A hand shadow, lighting change, camera movement, or a finger hovering over a key can cause false events. Mount the phone firmly, lock exposure/focus if your device allows it, use steady lighting, and calibrate carefully.

## Repository layout

```text
camerakey-wifi/
├── .github/workflows/android-apk.yml  # GitHub Actions APK build
├── android/                           # Android Studio project
│   ├── app/src/main/AndroidManifest.xml
│   ├── app/src/main/java/com/camerakey/wifi/MainActivity.kt
│   └── app/build.gradle.kts
├── pc-receiver/
│   ├── receiver.py                     # Windows/Linux keyboard receiver
│   ├── requirements.txt
│   ├── run-windows.bat
│   └── run-linux.sh
└── README.md
```

## 1. Start the PC receiver

Both devices must be connected to the **same trusted Wi-Fi network**. Do not expose port `8765` to the public internet.

### Windows 10/11

1. Install Python 3.10 or newer from https://www.python.org/downloads/ and enable **Add Python to PATH**.
2. Open PowerShell in `pc-receiver`.
3. Set a private token and run the receiver:

```powershell
$env:CAMERAKEY_TOKEN = "choose-a-long-private-token"
py -m pip install -r requirements.txt
py receiver.py
```

4. If Windows Firewall asks, allow Python on **Private networks only**.
5. Copy the LAN address printed by the receiver (for example `192.168.1.20`).

You can also run `run-windows.bat`; it installs dependencies and prompts for a token.

### Linux

For X11 desktops, install Python and the X11 input dependencies first. On Debian/Ubuntu/Linux Mint:

```bash
sudo apt update
sudo apt install python3 python3-venv python3-tk python3-dev
```

Then:

```bash
cd pc-receiver
chmod +x run-linux.sh
./run-linux.sh
```

Or run manually:

```bash
export CAMERAKEY_TOKEN="choose-a-long-private-token"
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python receiver.py
```

**Linux limitation:** `pynput` generally injects input through Xorg/X11. On Wayland, desktop security may block global keyboard injection. If input does not reach applications, log into an Xorg session and retry. Do not run the receiver as root as a workaround.

## 2. Build the Android APK

1. Install Android Studio with Android SDK 35 and JDK 17.
2. Open the `android` folder in Android Studio.
3. Let Gradle sync and download dependencies.
4. Choose **Build → Build Bundle(s) / APK(s) → Build APK(s)**.
5. The debug APK is generated at:

```text
android/app/build/outputs/apk/debug/app-debug.apk
```

### Build automatically with GitHub

1. Create a GitHub repository named `camerakey-wifi`.
2. Upload/push the contents of this folder to the repository's `main` branch.
3. Open **Actions → Build CameraKey APK**.
4. Wait for the workflow to finish, open the workflow run, and download the `camerakey-debug-apk` artifact.

The workflow builds a debug APK. It is suitable for testing and sideloading; a release APK should be signed before distribution.

## 3. Configure the phone

1. Install the APK and grant camera permission.
2. Mount the phone securely above the keyboard. Keep it fixed; do not hold it by hand.
3. Keep the whole keyboard in the camera preview. Use landscape orientation.
4. Enter the PC's LAN IP address in the app, port `8765` is used automatically, and enter the same token used by the receiver.
5. Tap **Connect**. Wait for **Connected · PC ready**.
6. Tap **Add key**, tap the centre of a key in the preview, and enter its label (examples: `A`, `B`, `1`, `SPACE`, `ENTER`, `BACKSPACE`, `SHIFT`, `CTRL`). Repeat for every key you want to use. The app can track more than four keys at once; the four-key requirement is not a hard limit.
7. Make sure nobody's hands are over the keyboard, then tap **Capture baseline**. The app records the uncovered keyboard image.
8. Tap **Detect: ON** and test one key at a time.

**Important:** calibrate the letter/character key area, not the gap between keys. Add separate entries for left and right Shift/Ctrl if you need them; the prototype treats labels as logical keys, so duplicate labels represent the same logical key and should not be used for two independently held copies at once.

## 4. Detection model

- Calibration stores normalized tap coordinates for each key.
- A baseline grayscale camera frame is captured with the keyboard uncovered.
- For each key, the app compares a circular region around its calibrated centre with the baseline.
- A key-down is emitted when enough pixels differ beyond a brightness threshold.
- A lower release threshold provides basic hysteresis to reduce flicker.
- Events are sent as JSON over WebSocket. The receiver presses/releases the corresponding desktop key.
- If the phone disconnects, the receiver releases held keys to avoid stuck modifiers.

This first version uses fixed thresholds. Results depend heavily on camera angle, focus, exposure, lighting, keycap reflectivity, and the size of the finger relative to the key. The baseline is not automatically refreshed because doing so while a finger is down could erase the reference state.

## 5. Supported key labels

Letters `A-Z`, digits `0-9`, and common labels: `SPACE`, `ENTER`, `TAB`, `BACKSPACE`, `DELETE`, `ESC`, `UP`, `DOWN`, `LEFT`, `RIGHT`, `HOME`, `END`, `PAGEUP`, `PAGEDOWN`, `SHIFT`, `CTRL`, `ALT`, `CAPSLOCK`, `INSERT`, and `F1-F12`.

Punctuation can be entered as the actual character (for example `,`, `.`, `-`, `=`). Keyboard layout and modifier behavior depend on the PC's active input layout.

## 6. Troubleshooting

- **Phone cannot connect:** verify both devices are on the same Wi-Fi, use the receiver's LAN IP (not `127.0.0.1`), allow TCP port `8765` through the private firewall, and confirm the token matches.
- **No key events:** check that the app says PC ready, capture a fresh baseline, and turn detection on.
- **False key presses:** improve even lighting, stop camera movement, increase the distance from direct glare, and recalibrate. Avoid shadows crossing other keys.
- **Key does not release:** move the finger fully away. If the phone disconnects, the receiver releases all held keys. Stop the receiver if an input device behaves unexpectedly.
- **Linux input does not work:** test in an Xorg session; Wayland may block `pynput` injection.
- **Preview is cropped or calibration is offset:** use a landscape mount and ensure the entire keyboard is visible. The prototype assumes the preview and analysis streams have matching orientation/aspect ratio.

## Security notes

The shared token is a basic LAN access check, not end-to-end encryption. WebSocket traffic is unencrypted (`ws://`). Use only a trusted private network, choose a long random token, and do not port-forward the receiver. For public or untrusted networks, add TLS and stronger authentication before use.

## Current prototype limitations / next improvements

- Calibration is in-memory and must be repeated after restarting the app.
- Key locations are added by tapping each key; there is no automatic keyboard-layout recognition yet.
- Image-difference detection may confuse shadows and finger occlusion. It does not prove a mechanical key was physically depressed.
- No auto exposure lock, camera motion detection, key-specific threshold tuning, or calibration export/import yet.
- Linux Wayland support is not guaranteed.

A future version should add calibration persistence, per-key region sizing, adaptive illumination compensation, camera exposure/focus controls, a test screen, and a signed release build.

**Calibration persistence:** key labels and positions are saved locally on the Android device. After reopening the app, the key map is restored, but you must capture a new clear-keyboard baseline before enabling detection.
