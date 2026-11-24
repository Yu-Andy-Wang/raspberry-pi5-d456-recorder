# Installation from a Blank SD Card

## 1. Flash Raspberry Pi OS

Use Raspberry Pi Imager to install a 64-bit Raspberry Pi OS image. For a headless deployment, configure the following options in Imager before writing the SD card:

- Username and password
- Wi-Fi network and wireless country
- SSH access
- Time zone and keyboard layout
- Hostname

Insert the SD card into the Raspberry Pi, power it on, and connect over SSH:

```bash
ssh <username>@<hostname>.local
```

Update the operating system:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

Reconnect and verify that the system is 64-bit:

```bash
uname -m
python3 --version
```

`uname -m` should report `aarch64`.

## 2. Clone This Repository

```bash
cd ~
git clone https://github.com/Yu-Andy-Wang/raspberry-pi5-d456-recorder.git
cd raspberry-pi5-d456-recorder
```

## 3. Build and Install librealsense

The build script installs compiler dependencies, clones librealsense `v2.59.2`, configures the RSUSB backend and Python bindings, builds the SDK, installs the udev rules, and installs the resulting libraries.

```bash
cd ~/raspberry-pi5-d456-recorder
chmod +x scripts/build_librealsense.sh
./scripts/build_librealsense.sh
```

The RSUSB backend avoids applying RealSense kernel patches to the current Raspberry Pi kernel. Compilation time depends on Raspberry Pi memory, cooling, and the value of `BUILD_JOBS`, which defaults to `2`.

After installation, reconnect the D456 and verify detection:

```bash
rs-enumerate-devices
```

Test the Python binding from the home directory, not from the librealsense build directory:

```bash
cd ~
python3 -c "import pyrealsense2 as rs; print('pyrealsense2 OK:', rs.__file__)"
```

Testing only inside `~/librealsense/build` can accidentally import a temporary build artifact instead of the globally installed module.

## 4. Test Simultaneous Streams

Return to the cloned repository and run:

```bash
cd ~/raspberry-pi5-d456-recorder
python3 scripts/smoke_test.py
```

Expected output includes:

```text
Camera: RealSense D456
Depth: True
Color: True
Infrared 1: True
```

A single center depth value of `0.0 m` does not necessarily indicate a failure. The smoke test also reports the valid-pixel count and median depth after discarding startup frames.

## 5. Install the Recorder

```bash
cd ~/raspberry-pi5-d456-recorder
chmod +x scripts/install_recorder.sh
./scripts/install_recorder.sh
```

The default installation directory is `~/python_code/`, and the default data directory is `~/Pictures/farrow_data/`. The installer creates the systemd service file but does not enable it automatically.

## 6. Perform the First Manual Recording Test

```bash
cd ~/python_code
python3 -u D456_pig_recorder.py
```

Allow at least one `:00` or `:30` capture cycle to finish, then press `Ctrl+C` for a clean shutdown.

Inspect the generated files:

```bash
find ~/Pictures/farrow_data -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %s %p\n' | sort
```

Confirm that RGB, DP, IR, IR_SN, DM, and CALIBRATION outputs are correct before enabling the watchdog.

## 7. Test the Watchdog Manually

```bash
cd ~/python_code
python3 -u monitor_d456_rgb_recorder.py
```

Open a second SSH session and follow the recorder log:

```bash
tail -f ~/python_code/D456_recorder_output.log
```

The health file should contain `OK` after camera startup:

```bash
cat ~/python_code/recorder_camera_health.status
```

Stop the watchdog with `Ctrl+C` after testing.

## 8. Choose Autostart Mode

After the manual tests pass, choose one mode. Do not enable both.

### Headless systemd mode

Enable and start the systemd service:

```bash
sudo systemctl enable --now d456-monitor.service
systemctl status d456-monitor.service --no-pager
```

Verify operation after a reboot:

```bash
sudo reboot
```

Reconnect and run:

```bash
systemctl is-enabled d456-monitor.service
systemctl is-active d456-monitor.service
tail -n 100 ~/python_code/D456_recorder_output.log
```

The first two commands should report `enabled` and `active`.

### Desktop preview mode

To open a terminal and the `RealSense Depth` preview after graphical login:

```bash
cd ~/raspberry-pi5-d456-recorder &&
git pull --ff-only &&
sudo systemctl disable --now d456-monitor.service &&
bash scripts/enable_preview_autostart.sh
```

The repository (`~/raspberry-pi5-d456-recorder`) and installed programs (`~/python_code`) are different directories. The `scripts/` directory belongs to the repository. The commands above work after a new SSH login or after the manual tests, regardless of the previous working directory. They update the preview launcher without rebuilding the SDK or replacing recorder settings.

Use `sudo raspi-config` to select desktop boot and desktop automatic login if the windows should appear after power-on without manual login. This mode requires Raspberry Pi OS with a graphical desktop. Then log out and back in, or reboot. The preview launcher refuses to start if the systemd recorder is active.

Check the generated entry before rebooting:

```bash
cat ~/.config/autostart/d456-recorder-preview.desktop
```

For the default installation it should contain `Exec=/bin/bash "/home/<username>/python_code/run_preview.sh"` and `Terminal=true`. To test immediately, open a terminal inside the graphical desktop and run:

```bash
bash ~/python_code/run_preview.sh
```

An SSH terminal does not display the graphical preview. If enabling autostart reports an error, resolve it before rebooting.

## 9. Firmware Compatibility

This project completed simultaneous stream and recording tests with D456 firmware `5.16.0.1`. The librealsense `v2.59.2` release notes recommend firmware `5.17.3.10` or later for the D456 USB model.

Firmware flashing writes to the camera's internal storage. The scripts therefore do not download or install camera firmware. If an update is required, use only the official firmware and instructions, and ensure that neither the camera nor Raspberry Pi loses power during the update.

