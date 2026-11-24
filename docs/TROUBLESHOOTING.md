# Troubleshooting

## `No module named 'cv2'` or `No module named 'scipy'`

```bash
sudo apt update
sudo apt install -y python3-opencv python3-scipy
```

Check every runtime dependency:

```bash
python3 - <<'PY'
import importlib

for name in ("pyrealsense2", "cv2", "numpy", "scipy", "gpiozero", "lgpio", "psutil"):
    try:
        module = importlib.import_module(name)
        print("OK", name, getattr(module, "__file__", "built-in"))
    except Exception as exc:
        print("MISSING", name, exc)
PY
```

## `pyrealsense2` Imports Only Inside the Build Directory

```bash
cd ~/librealsense/build
sudo cmake --install .
sudo ldconfig
cd ~
python3 -c "import pyrealsense2 as rs; print(rs.__file__)"
```

## RealSense Viewer Reports a GLX 1.3 Error

RealVNC Virtual Mode may not expose the GLX version required by RealSense Viewer. Use the headless smoke test and recorder logs instead. This error does not indicate a failed SDK or camera installation.

## The Preview Window Is Black

The program disables the D456 emitter while waiting for the next recording boundary, so the depth preview may be almost entirely black. The emitter is enabled during the 20-second recording interval. No preview window is created when the recorder runs as a headless systemd service.

To use the visible preview, enable desktop mode and disable systemd mode:

```bash
cd ~/raspberry-pi5-d456-recorder &&
git pull --ff-only &&
sudo systemctl disable --now d456-monitor.service &&
bash scripts/enable_preview_autostart.sh
```

## `scripts/enable_preview_autostart.sh: No such file or directory`

The setup scripts are in `~/raspberry-pi5-d456-recorder/scripts`, not in the home directory or `~/python_code`. Run the complete command block above, including `cd` and `git pull --ff-only`. The pull also retrieves the latest launcher fixes. No SDK rebuild is needed for this switch.

## `Health status write failed`

Confirm that the installation directory exists and is writable:

```bash
mkdir -p ~/python_code
test -w ~/python_code && echo writable
```

The current recorder creates the health-status directory automatically.

## The Watchdog Shows No Recorder Output

This is expected. The watchdog redirects the child recorder's standard output to:

```bash
tail -f ~/python_code/D456_recorder_output.log
```

## Center Depth Is `0.0 m`

The first frame, an invalid center pixel, a target that is too close, or glass, dark, and reflective materials can all return zero. Point the camera at an ordinary object approximately 1–2 meters away and run `scripts/smoke_test.py` to inspect the valid-pixel ratio and median depth.

## USB or Frame Timeout Errors

- Use a Raspberry Pi 5 USB 3 port.
- Use a reliable USB 3 data cable.
- Avoid an unpowered USB hub.
- Run `rs-enumerate-devices` and confirm that `Usb Type Descriptor` is 3.x.
- Check Raspberry Pi power delivery and cooling.

## Inspect Recent Errors

```bash
tail -n 100 ~/python_code/D456_recorder_output.log
tail -n 100 ~/python_code/monitor_d456_log.txt
journalctl -u d456-monitor.service -n 100 --no-pager
```

