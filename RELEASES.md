# Releases

## v2026.10.21 - ID TECH platform 3.1.0

Board image: `GB_RK3576_UBUNTU2204_<yymmdd_hhmmss>.img.zst` (Rockchip `update.img`, Ubuntu 22.04
desktop, kernel 6.1), on the release page.

**ID TECH app**

- The app window no longer shows the web console's own minimise / maximise / close buttons:
  the desktop's title bar has them. Sign out stays in the console's own menu ((i) > Exit
  Application). The web console in a browser is unchanged.
- No more ID TECH menu bar. Network access to the web console moved to **Setting > Web**, as on
  Android (Settings > Web): an Enabled switch, its state (Running on port 8080 / Off / Not
  running) and the addresses to open. It is in the app window only; browsers do not get it.
  Quit: the tray icon or Ctrl+Q; F5 reloads.
- Reader unplugged and plugged back in (USB-serial adapter or the board's UART header): the app
  picks the reader up again on the next action (Connect, Start, Inventory, ...). Before, it kept
  reporting "no reader connected" until the app was closed and opened again.

**Reader SDK** (`idtech-uhf-sdk.jar`)

- An open session survives a replug: the next call re-detects the port, resets the reader and
  re-applies its region; a stream that the unplug ended can be started again. While the reader is
  away, calls fail with `ERROR_DEVICE_LOST`.

**Board image**

- The release image is named like the Android build's (`GB_RK3576_UBUNTU2204_<yymmdd_hhmmss>.img`).
- Each board makes its own SSH host keys and machine id at first boot (they were the same on every
  board flashed from one image, so boards could get the same DHCP address).

Tests: 195 unit tests, the end-to-end test against the simulated reader (now with replug checks:
unplug while streaming, unplug while idle, Connect after a replug), the GUI smoke test.

## ID TECH platform 3.0.0 (2026-10-15)

- The app window is the web console itself (JavaFX WebView): identical UI.
- Reader UART (`ttyDC3`) clock pinned to 24 MHz: base_baud 1500000, 115200 exact.
- PCIe controller off (no reader on it; removes the ~8 s link-training wait at boot).
- USB autosuspend off; ModemManager leaves the reader's ports alone.
