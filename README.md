# Raspberry Pi 5 + RealSense D456 Recorder

A headless Intel RealSense D456 recorder and watchdog for Raspberry Pi 5.

## Features

- Keeps the D456 pipeline open continuously.
- Starts a 20-second capture at `:00` and `:30` of every minute.
- Saves RGB, colorized depth, and infrared video at 5 FPS.
- Saves an emitter-OFF infrared snapshot immediately before each recording.
- Saves camera calibration metadata and DM point-cloud MAT data.
- Restarts the D456 capture subprocess after a crash or native SDK stall.
- Restarts the main recorder if it exits unexpectedly.
- Supports SSH-only and systemd operation without a monitor or desktop login.

The default deployment configuration is:

```python
ENABLE_ARDUCAM = False
ENABLE_HIGH_RATE_CLIPS = False
RECORD_VDA = False
```

`v4l2_sequence_capture.py` is retained for optional future Arducam support. It is not loaded while `ENABLE_ARDUCAM` is `False`.

## Tested Environment

- Raspberry Pi 5
- 64-bit Raspberry Pi OS / Debian 13
- Python 3.13
- Intel RealSense D456 connected through USB 3.2
- librealsense `v2.59.2`, built with the RSUSB backend
- D456 firmware `5.16.0.1`, verified with simultaneous depth, color, and infrared capture

The librealsense `v2.59.2` release notes recommend D456 USB firmware `5.17.3.10` or later. Firmware updates are deliberately excluded from the automated installer because power or USB interruption during an update can damage the device configuration.

## Documentation

1. [Installation from a blank SD card](docs/INSTALL.md)
2. [Operation, logs, and service management](docs/OPERATIONS.md)
3. [Troubleshooting](docs/TROUBLESHOOTING.md)

## Clone the Repository

```bash
git clone https://github.com/Yu-Andy-Wang/raspberry-pi5-d456-recorder.git
cd raspberry-pi5-d456-recorder
```

## Repository Layout

```text
src/
  D456_pig_recorder.py
  monitor_d456_rgb_recorder.py
  v4l2_sequence_capture.py
scripts/
  build_librealsense.sh
  install_recorder.sh
  uninstall_recorder.sh
  run_preview.sh
  enable_preview_autostart.sh
  disable_preview_autostart.sh
  smoke_test.py
systemd/
  d456-monitor.service.in
docs/
  INSTALL.md
  OPERATIONS.md
  TROUBLESHOOTING.md
```

## Data Directory

Data is saved by default under:

```text
~/Pictures/farrow_data/
```

Override the location with:

```bash
export D456_DATA_ROOT=/path/to/data
```

Recordings and calibration files may contain experimental data and camera serial numbers. They are excluded by `.gitignore` and should not be committed to a public repository.

## Choose a Launch Mode

The installer creates a systemd service that starts the watchdog at boot:

```text
Raspberry Pi boot
  -> d456-monitor.service
  -> monitor_d456_rgb_recorder.py
  -> D456_pig_recorder.py
```

The service is installed but not enabled automatically. Complete a manual recording test first, then enable it:

```bash
sudo systemctl enable --now d456-monitor.service
```

This headless mode is recommended for unattended operation. It does not open a terminal or video preview.

For a visible terminal and `RealSense Depth` preview after graphical login, use desktop autostart instead:

```bash
cd ~/raspberry-pi5-d456-recorder &&
git pull --ff-only &&
sudo systemctl disable --now d456-monitor.service &&
bash scripts/enable_preview_autostart.sh
```

The `scripts/` directory is in the repository, not in `~/python_code`. Desktop mode requires a graphical login; configure desktop automatic login with `sudo raspi-config` for unattended boot. See the [installation guide](docs/INSTALL.md#desktop-preview-mode) for verification steps.

Do not enable both modes at the same time. Stop any running desktop recorder with `q` or `Ctrl+C` before switching back:

```bash
cd ~/raspberry-pi5-d456-recorder &&
bash scripts/disable_preview_autostart.sh &&
sudo systemctl enable --now d456-monitor.service
```

## Important Notes

- Connect the D456 directly to a Raspberry Pi 5 USB 3 port with a reliable USB 3 cable.
- RealSense Viewer may fail with a GLX 1.3 error inside RealVNC Virtual Mode. This does not affect headless Python capture.
- Test `D456_pig_recorder.py` manually before enabling the watchdog service.
- The watchdog redirects recorder output to log files; it does not provide a full graphical recording interface.

## Upstream Documentation

- [Raspberry Pi headless setup](https://www.raspberrypi.com/documentation/computers/getting-started.html)
- [librealsense Linux installation](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)
- [librealsense RSUSB backend](https://github.com/realsenseai/librealsense/blob/master/doc/installation_jetson.md)
- [librealsense v2.59.2 release](https://github.com/realsenseai/librealsense/releases/tag/v2.59.2)

## License

[MIT License](LICENSE)

