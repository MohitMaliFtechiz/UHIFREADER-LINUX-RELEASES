# UHIFREADER-LINUX releases

Ready-to-use builds of **Ubuntu 22.04 for the ZTL RK3576 C576 UHF RFID reader board** and of the
**ID TECH UHF platform for Linux** (the ID TECH app, the reader SDK, the client SDK, `uhfctl`).
Downloads are on the [Releases](../../releases) page; what changed is in [RELEASES.md](RELEASES.md).
Source: [UHIFREADER-LINUX](https://github.com/MohitMaliFtechiz/UHIFREADER-LINUX).

## Files in a release

| File | What |
|---|---|
| `GB_RK3576_UBUNTU2204_<yymmdd_hhmmss>.img.zst` | The board image: Ubuntu 22.04 desktop with the ID TECH app installed (Rockchip `update.img`, zstd-compressed) |
| `idtech-uhf_<version>_arm64.deb` | The ID TECH platform for a board already running Ubuntu |
| `idtech-uhf_<version>_amd64.deb` | The same for an x86-64 PC with Ubuntu 22.04 and a USB reader |
| `idtech-uhf-sdk-<version>.zip` | Reader SDK kit: jar, sources, examples, guide |
| `idtech-connect-client-<version>.zip` | Client SDK kit: for programs that use an ID TECH app's web service |
| `SHA256SUMS` | Checksums of the files above |

## Flash the board

1. Download the image and unpack it: `zstd -d GB_RK3576_UBUNTU2204_*.img.zst`
2. Put the board in Loader mode: hold **Recovery** while powering it on, with its USB OTG port connected to the PC.
3. Flash:
   - Linux: `sudo ./upgrade_tool uf GB_RK3576_UBUNTU2204_*.img`
   - Windows: RKDevTool > Upgrade Firmware > Firmware: choose the `.img` > Upgrade

   Both tools come with Rockchip's SDK for the RK3576 (`RKDevTool` needs Rockchip's USB driver).
4. First boot takes a little longer: the root filesystem grows to the whole eMMC and the board
   makes its own SSH keys.

Sign in as `ubuntu` / `ubuntu` and **change that password** (`passwd`): it is the same on every
board flashed from this image, and SSH accepts it.

## Install on Ubuntu

```
sudo apt install ./idtech-uhf_<version>_arm64.deb     # amd64 on a PC
```

Then open **ID TECH** from the menu and sign in as `admin` / `admin@123` (a new password is
required at first sign-in). Your user must be in group `dialout` to reach the reader.

## The ID TECH app

The app's window is the ReadersHub web console. **Setting > Web** puts the same console and REST API
on the network (port 8080, `idtech.local`) for browsers and the client SDK; MQTT publishing is in
**Setting > MQTT**. With either on, the app keeps running in the system tray when its window is
closed, and starts at login.

Developer guide (reader SDK, client SDK, REST API, `uhfctl`): [docs/SDK.md](docs/SDK.md).

## Not in these builds

The superadmin recovery account of the factory images is not in the public builds.

## Third-party software

- The image contains Ubuntu 22.04 (Canonical and the Ubuntu package authors; each package's
  license is in `/usr/share/doc/<package>/copyright` on the board), a Linux 6.1 kernel (GPL-2.0)
  and Rockchip's RK3576 loader and U-Boot.
- The ID TECH packages contain OpenJFX 22.0.2 (GPL-2.0 with the Classpath Exception) and the
  Outfit, Roboto and Roboto Mono fonts (SIL OFL 1.1); see
  `/usr/share/doc/idtech-uhf/NOTICE` once installed.
