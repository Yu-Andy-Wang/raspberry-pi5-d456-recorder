# Operation and Maintenance

## Recording Schedule

The main program keeps the D456 pipeline open and begins a 20-second recording at `:00` and `:30` of every minute. The emitter is disabled while waiting and enabled at the recording boundary.

Default settings:

- Arducam: disabled
- VDA export: disabled
- Scheduled high-rate clips: disabled
- D456 saved frame rate: 5 FPS
- DM output: one file per recording clip

## Launch Modes

```text
Raspberry Pi boot
  -> d456-monitor.service
  -> monitor_d456_rgb_recorder.py
  -> D456_pig_recorder.py
```

The project provides two mutually exclusive modes. Headless systemd mode starts the watchdog before graphical login. Desktop preview mode starts the main recorder after graphical login and keeps the terminal and OpenCV depth window visible. Never enable both modes.

## Service Control

```bash
sudo systemctl start d456-monitor.service
sudo systemctl stop d456-monitor.service
sudo systemctl restart d456-monitor.service
systemctl status d456-monitor.service --no-pager
```

Enable or disable boot startup:

```bash
sudo systemctl enable d456-monitor.service
sudo systemctl disable d456-monitor.service
```

## Logs

Watchdog log:

```bash
tail -f ~/python_code/monitor_d456_log.txt
```

Recorder log:

```bash
tail -f ~/python_code/D456_recorder_output.log
```

systemd journal:

```bash
journalctl -u d456-monitor.service -f
```

## Health Status

```bash
cat ~/python_code/recorder_camera_health.status
```

A healthy configuration contains:

```text
OK arducam_enabled=0 rgb=1 realsense=1
```

The PID file contains only the current recorder process ID. It is created and removed at runtime and must not be committed to GitHub:

```bash
cat ~/python_code/D456_rgb_recorder.pid
```

## Headless Operation

When started by SSH or systemd without a graphical display, the recorder disables its OpenCV preview automatically and continues recording. The watchdog has no full graphical interface; verify operation through logs, the health file, and generated output files.

## Desktop Preview Operation

```bash
cd ~/raspberry-pi5-d456-recorder &&
git pull --ff-only &&
sudo systemctl disable --now d456-monitor.service &&
bash scripts/enable_preview_autostart.sh
```

After the next graphical login, the terminal and `RealSense Depth` window are displayed. Press `q` in the preview window or `Ctrl+C` in the terminal to stop safely.

The repository (`~/raspberry-pi5-d456-recorder`) contains the setup scripts; `~/python_code` contains installed runtime files. The explicit `cd` above is required when following these steps from a fresh SSH session or after testing the installed recorder.

The enable script updates the installed preview launcher without replacing recorder settings. Use `sudo raspi-config` to select desktop boot and desktop automatic login if the windows must appear after power-on without a manual login. This requires Raspberry Pi OS with a desktop environment. Reboot and connect to that desktop to see the terminal and preview. To test immediately, run `bash ~/python_code/run_preview.sh` in a terminal inside the graphical desktop.

Desktop preview mode launches the recorder directly. Its internal camera supervisor remains active, but the separate main-process watchdog is only used in headless service mode.

## Change the Data Directory

For a manual run:

```bash
D456_DATA_ROOT=/mnt/storage/farrow_data \
  python3 -u ~/python_code/D456_pig_recorder.py
```

For background operation, edit the `D456_DATA_ROOT` environment line in `/etc/systemd/system/d456-monitor.service`, then reload and restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart d456-monitor.service
```

## Update the Recorder

Stop the service, pull the new repository version, rerun the installer, and start the service again:

```bash
cd ~/raspberry-pi5-d456-recorder
sudo systemctl stop d456-monitor.service
git pull --ff-only
./scripts/install_recorder.sh
sudo systemctl start d456-monitor.service
```

The installer does not delete existing recording data.

