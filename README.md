<p align="center">
  <img src="docs/img/logo.svg" width="110" alt="LudemOS">
</p>

<h1 align="center">LudemOS for PS5</h1>

<p align="center">
  A gaming Linux distribution for the <b>PlayStation 5 Slim</b>: it installs to the M.2 SSD with no USB drive or
  cables, boots from the console's home screen with the controller, and puts Windows and Linux games in a
  full-screen launcher designed for the TV.
</p>

<p align="center">
  <img alt="Console" src="https://img.shields.io/badge/console-PS5%20Slim%20·%20fw%2013.60-0B0B0F?style=flat-square">
  <img alt="Kernel" src="https://img.shields.io/badge/kernel-7.1.7--ludemos-2F6FED?style=flat-square">
  <img alt="Base" src="https://img.shields.io/badge/base-CachyOS%20(Arch)-1793D1?style=flat-square">
  <img alt="Launcher" src="https://img.shields.io/badge/launcher-Qt6%20%2F%20QML%20%2B%20SDL3-41CD52?style=flat-square">
</p>

<p align="center">
  <b>English</b> · <a href="README.it.md">Italiano</a>
</p>

<p align="center">
  <img src="docs/img/launcher-01-home.jpg" alt="LudemOS Home launcher" width="900">
</p>

> [!NOTE]
> This page presents the work done so far. **The source code will be published in this repository later**, once
> the project is ready to be used by others.

---

## Contents

- [At a glance](#at-a-glance)
- [What works](#what-works)
- [How it boots](#how-it-boots)
- [Installation](#installation)
- [The system](#the-system)
- [The desktop](#the-desktop)
- [LudemOS Home, the launcher](#ludemos-home-the-launcher)
- [Problems solved](#problems-solved)
- [Tested games](#tested-games)
- [Measurements](#measurements)
- [LudemNR: a neural upscaler for the PS5 GPU](#ludemnr-a-neural-upscaler-for-the-ps5-gpu)
- [Known limitations](#known-limitations)
- [Credits](#credits)
- [Notice](#notice)

---

## At a glance

LudemOS starts from the work of the [ps5-linux](https://github.com/ps5-linux) community (loader, kernel patches,
tools) and builds a complete, maintained system on top of it, with its own fixes for the **PS5 Slim**, whose
hardware differs from the models the original project was tested on.

- **Dual boot from the console.** A LudemOS icon on the PS5 home screen runs the exploit and opens our own boot
  menu: pick *PlayStation 5* or *LudemOS* with the controller. No PC, no USB drive, no DNS tricks.
- **Installation with no external hardware.** The kernel and installer load from the console's internal storage;
  the installer joins Wi-Fi, prepares the M.2 and downloads the system, resuming after interruptions.
- **Slim hardware fixed in the kernel.** MediaTek MT7921 Wi-Fi and Bluetooth, DualSense over Bluetooth, the Slim
  fan, CPU/GPU frequency ranges, HDMI audio, HDR10: seven patches (two ported from upstream).
- **Updates that never break booting.** A signed pacman repository; the boot kernel is never touched, new kernels
  are installed alongside it and confirmed only if the network works.
- **Power management.** A daemon drives fan and clocks: idle at 43 °C, full boost within seconds of starting a
  game.
- **LudemOS Home.** A console-style full-screen launcher (Qt6/QML + SDL3): Windows games through Proton **without
  Steam**, Linux games, optional Steam library, per-game presets, virtual gamepad, automatic cover art.
- **LudemNR.** Ongoing research on a neural upscaler written for the PS5 GPU (RDNA 1, packed FP16), already
  running inside real games.

---

## What works

| Area | Status | Notes |
|---|---|---|
| Boot from the PS5 home screen, no PC | ✅ | icon → exploit → menu → LudemOS in about two minutes |
| Wake with the DualSense PS button | ✅ | no need for the console's power button |
| Installation to M.2 over Wi-Fi | ✅ | 1 TB SSD, system on `nvme0n1p1` |
| MT7921 Wi-Fi (5 GHz) | ✅ | 867/585 Mbit/s link, stable |
| Bluetooth and DualSense | ✅ | buttons, sensors, touchpad, battery; pairing over USB cable too |
| Slim fan | ✅ | temperature target set by the kernel and the daemon |
| CPU/GPU clocks | ✅ | idle 1600–1800 / 800–1000 MHz, in-game boost 3500 / 2230 MHz |
| KDE Plasma desktop (Wayland) at 4K | ✅ | 2× scaling, on-screen keyboard |
| Windows (Proton) and Linux games | ✅ | Control and The Witcher 3 Remastered at 1080p, locked 60 fps with LudemNR |
| System and kernel updates | ✅ | `ludemos-aggiorna`, switch via kexec |
| HDMI audio | ✅ | fix for the FLAVA3 HDMI chip |
| HDR10 | ✅ | KDE desktop and games started from the launcher; 12-bit YCbCr 4:2:2 at 1080p60/2160p60 |
| Clean shutdown from Linux | ✅ | `systemctl poweroff` really powers off the console |
| 120 Hz output | ❌ | the HDMI bridge only exposes 60 Hz modes |
| Same DualSense pairing in PS5 OS and Linux | ❌ | must be re-paired when switching systems |

---

## How it boots

<p align="center"><img src="docs/img/catena-avvio.svg" alt="Boot chain" width="900"></p>
<p align="center"><sub>The diagram labels are in Italian; the steps are described below.</sub></p>

1. **LudemOS app**: an app on the PS5 home screen that keeps in the browser cache the Relapse exploit (original
   files untouched plus a small patch), the boot menu and an install page.
2. **Relapse** exploits WebKit and the PS5 OS kernel and starts `elfldr`, which accepts programs on port 9021.
   If `elfldr` is already running the exploit is skipped.
3. **Boot menu** (`ludemos-boot`): our own payload that opens the console browser on a local page with the
   entries read from `/data/ludemos-boot/avvio.txt` (editable over FTP) and, when one is chosen, sends the right
   programs to `elfldr`. Controller driven: arrows to choose, ✕ to start, ◯ to close.
4. **Linux loader**: a fork of `ps5-linux-loader` that reads kernel, initrd, command line and VRAM size from
   `/data/linux/` on the internal storage and puts the console into rest mode.
5. Pressing the **PS button** wakes the console straight into Linux.
6. The **boot kernel** (7.1.7-ludemos) starts. It is never updated: it is the safety net.
7. **`ludemos-kernel`** switches with `kexec` to the newest updated kernel (`-ludemosN`) and confirms it only if
   the network works; otherwise the next power-on stays on the boot kernel.
8. **CachyOS** from the M.2: KDE Plasma with automatic login and LudemOS Home full screen.

<p align="center"><img src="docs/img/menu-avvio.png" alt="Boot menu on the PS5" width="820"></p>
<p align="center"><sub>The boot menu (<code>ludemos-boot</code>) as it appears on the TV ("Avvio" = Boot).</sub></p>

---

## Installation

> [!IMPORTANT]
> It requires a **PS5 Slim on firmware 13.60** (or a firmware supported by Relapse and `ps5-linux-loader`) and a
> dedicated M.2 SSD: PS5 game storage on that SSD is lost. Consoles on firmware 14.00 or later are not supported.
> **Do not update the firmware.**

<p align="center"><img src="docs/img/installazione.svg" alt="Installation with no external hardware" width="900"></p>

The official ps5-linux procedure needs a USB drive, keyboard and mouse. LudemOS installs **over Wi-Fi only**:

1. **Build on the PC** (Debian LXC container): kernel 7.1.7 with the PS5 patches and ours, an installer initrd
   (busybox, `wpa_supplicant` with the network preconfigured, `m2_init`, `e2fsprogs`, `dropbear` for SSH control),
   and the CachyOS system built with distrobuilder and split into sha256-checked chunks.
2. **PS5 internal storage**: with the exploit running and the FTP server up, the installer kernel and initrd go
   into `/data/linux/`; then the loader is sent.
3. **Phase 1**: `m2_init` writes a partition table with a single Linux partition on the M.2, leaving intact the
   metadata PS5 OS checks (so the console never asks to reformat). Back to PS5 OS to verify it.
4. **Phase 2**: ext4 formatting and download of the system from the PC with resume (`wget -c`), sha256 check of
   every chunk and extraction with GNU `tar`, which keeps xattrs, capabilities and setuid bits (busybox's does not).
5. **Normal boot**: `/data/linux/` gets the system kernel and initrd with `root=LABEL=ludemos-m2`.

Every phase prints its status on screen and answers over SSH, with a timeout on every wait. The whole path was
tested first in **QEMU** with the PS5 kernel (console-specific drivers excluded via `initcall_blacklist`), a fake
250 GB NVMe M.2 and emulated USB networking: installer bugs surfaced and were fixed there before touching the
console.

### PS5 settings

- Power Saving → Features Available in Rest Mode → Supply Power to USB Ports: **Always**.
- HDMI → **disable** "Enable HDMI Device Link".
- Automatic updates **off**.
- For Linux sessions the exploit must run **without kstuff** (it prevents rest mode and blocks Linux from booting).

---

## The system

The base is the ps5-linux CachyOS image; a script applies a set of explicit, reviewed changes to the official
recipe, and a second script checks the result (enabled services, files, packages) before producing the archive.

**Our packages** (GPG-signed pacman repository `[ludemos]`):

| Package | Contents |
|---|---|
| `linux-ps5-ludemos` | 7.1.7 boot kernel, pinned with `IgnorePkg` |
| `linux-ps5-ludemos-agg` | updated `-ludemosN` kernels, built with `-march=x86-64-v3 -mtune=znver2` |
| `ludemos-tools` | system tools (table below) |
| `ludemos-home` | the launcher |

**`ludemos-tools`**:

| Tool | Purpose |
|---|---|
| `ludemos-fand` | fan and clocks: Quiet / Balanced / Performance profiles, boost while a game runs, 92 °C protection, M.2 SSD protection above 75 °C |
| `ludemos-kernel` | `kexec` switch to the updated kernel, confirmation once the network works, one switch per power-on |
| `ludemos-aggiorna` | system update (`pacman -Syu`), `--passa` to switch to the new kernel right away |
| `ludemos-dualsense` | DualSense pairing over USB: writes the console address and a link key into the controller and registers them in BlueZ; runs automatically when the cable is plugged in |
| `ludemos-schermo` | restores the video signal after the TV is switched off and on |
| `ludemos-frequenze` | measurements: real CPU frequency (APERF/MPERF) and temperatures over N seconds |
| `ludemos-nome` | system identity (LudemOS name and logo), restored after base-system updates |
| `ludemos-moduli-*` | pacman hooks that keep the running kernel's modules during updates |
| `gamescope-session-ps5` | Gamescope + Steam session with full logging |
| udev rules | the chip's "ghost" Bluetooth adapter, automatic DualSense pairing |

Defaults: Wi-Fi power saving off, zram, `systemd-timesyncd` (under Linux the PS5 clock starts at 1970), 10 GiB
GTT, key-only SSH, KDE without suspend or screen lock.

---

## The desktop

Behind the launcher there is a full **KDE Plasma 6** desktop on Wayland, at 3840×2160 with 2× scaling, reachable at
any time from Power → Desktop (the PS button brings the launcher back). Screenshots taken on the console; the
interface is in Italian.

<p align="center"><img src="docs/img/desktop-desktop.jpg" alt="KDE Plasma desktop on PS5" width="900"></p>

<table>
  <tr>
    <td width="50%"><img src="docs/img/desktop-menu.jpg" alt="Application menu"><br><sub><b>Application menu</b>: Firefox, Dolphin, Discover, system settings.</sub></td>
    <td width="50%"><img src="docs/img/desktop-info.jpg" alt="System information"><br><sub><b>System information</b>: LudemOS PS5, Plasma 6.8.1, Qt 6.12, kernel 7.1.7-ludemos5, the PS5's AMD APU (16 threads), 13.6 GiB of usable RAM.</sub></td>
  </tr>
</table>

---

## LudemOS Home, the launcher

Qt6/QML + SDL3 · C++20

It runs full screen inside KDE and **does not depend on Steam**: Windows games start through Proton via
`umu-run`, with one prefix per game. It is driven by the controller (DualSense, Xbox and compatible pads) or the
keyboard. The look follows console interfaces: full-screen artwork, portrait covers, a floating menu bar. The
interface is in Italian.

<table>
  <tr>
    <td width="50%"><img src="docs/img/launcher-02-libreria.jpg" alt="Library"><br><sub><b>Library</b>: filters, sorting, Proton or Linux for each game.</sub></td>
    <td width="50%"><img src="docs/img/launcher-03-pagina-gioco.jpg" alt="Game page"><br><sub><b>Game page</b>: activity, settings, tools.</sub></td>
  </tr>
  <tr>
    <td><img src="docs/img/launcher-09-gioco-impostazioni.jpg" alt="Game settings"><br><sub><b>Per-game settings</b>: Performance / Quality presets, Proton version, FSR, NTSync, MangoHud.</sub></td>
    <td><img src="docs/img/launcher-05-accesso-rapido.jpg" alt="Quick settings"><br><sub><b>Quick settings</b> (hold Guide): volume, Wi-Fi, Bluetooth, fan, fps limit, controller.</sub></td>
  </tr>
  <tr>
    <td><img src="docs/img/launcher-04-menu.jpg" alt="Menu"><br><sub><b>Menu</b> (PS/Guide button), also on top of a running game.</sub></td>
    <td><img src="docs/img/launcher-08-accensione.jpg" alt="Power"><br><sub><b>Power</b>: shut down, restart, switch to desktop.</sub></td>
  </tr>
  <tr>
    <td><img src="docs/img/launcher-06-aggiungi.jpg" alt="Add game"><br><sub><b>Add game</b>: from USB drive, network share or M.2; copy or run an installer.</sub></td>
    <td><img src="docs/img/launcher-07-impostazioni.jpg" alt="Settings"><br><sub><b>Settings</b>: system, Proton versions, games, information.</sub></td>
  </tr>
</table>

**Main features**

- **Windows games without Steam.** Proton per game: automatic (Proton-CachyOS if present, otherwise UMU-Proton),
  GE-Proton downloaded with sha512 verification, Steam's versions or manually installed ones. Windows installers
  run inside the game's prefix, then the executable is chosen; `winecfg`, `winetricks`, prefix reset.
- **Per-game presets**: *Performance* (1080p at 60 fps) and *Quality* modes. The launcher writes only the listed
  entries into the game's config files (JSON or INI), preserving types and formatting, and keeps the original so it
  can be restored.
- **Virtual gamepad**: for every real pad it creates a `uinput` pad with the identity of Steam's virtual gamepad
  and hides the real one from the game. It fixes characters that keep walking on their own when Proton reads some
  controllers directly, without needing Steam Input.
- **Optional Steam games**: read only when enabled; launch options (`localconfig.vdf`) are written by the launcher
  while Steam is closed.
- **Automatic cover art** from the Steam store for local games, looked up by title.
- **Folder to scan**, chosen by the user: every subfolder becomes a game (installers excluded). No scanning without
  consent.
- **In game**: the PS button brings the launcher on top of the game (through a KWin script over D-Bus, since focus
  stealing prevention rejects normal requests), then *Resume* or *Quit game*, which also closes processes that
  detached inside Steam's container.
- **HDR per game**: when the KDE desktop is in HDR, games start in Gamescope with HDR enabled and Proton exposes an
  HDR10 swapchain, so the game's own HDR option appears.
- **In-game boost**: the launcher tags the game process (`LUDEMOS_GIOCO=1`) and `ludemos-fand` raises the clocks.
- **System**: fan, Wi-Fi, Bluetooth, volume, updates, DualSense pairing, MangoHud levels, GameMode, nested
  Gamescope for integer-scaled 1080p output on the 4K TV.

---

## Problems solved

Each entry gives the symptom, the cause found and the fix.

### Wi-Fi

<details open>
<summary><b>The PS5 Slim doesn't have the Wi-Fi chip the original project expects</b></summary>

- **Symptom**: no Wi-Fi interface.
- **Cause**: this Slim has a **MediaTek MT7921** (`14c3:7961`), not the Marvell IW620 of other revisions, and the
  official kernel doesn't enable the PCIe driver.
- **Fix**: `CONFIG_MT7921E=m` with `disable_aspm=1`, redistributable firmware from `linux-firmware`.
</details>

<details>
<summary><b>"driver own failed": the chip never powers on</b></summary>

- **Cause**: the 7.2 branch of the PS5 kernel dropped the Salina USB controller initialisation; without it the
  MT7921 stays off. This is also why other users saw failures on 7.2.3.
- **Fix**: kernel 7.1.7 with Salina initialisation enabled.
</details>

<details>
<summary><b>The console freezes about 10 seconds after Wi-Fi starts</b></summary>

- **Symptom**: "xHCI host controller not responding, assume dead", then a full freeze.
- **Cause**: MT7921 power saving (runtime PM, deep sleep) keeps handing the chip back and forth between driver and
  firmware; on the Slim the chip shares its link with the Salina controller, which dies.
- **Fix**: patch `0001` (power saving disabled in the driver), `pcie_aspm=off`, `usbcore.autosuspend=-1`, power
  save off in NetworkManager.
</details>

<details>
<summary><b>WPA2 in the initrd: "key addition failed"</b></summary>

- **Cause**: the kernel loads `ccm` through `/sbin/modprobe`, missing from the busybox initrd; `wpa_cli` looked for
  `/var/run`.
- **Fix**: a real `modprobe` in the initrd and `/var/run` linked to `/run`.
</details>

### Bluetooth and DualSense

<details open>
<summary><b>The DualSense won't connect: packets go to the wrong endpoint</b></summary>

- **Symptom**: pairing succeeds but the controller never connects; no "Number of Completed Packets".
- **Cause**: the MT7921 Bluetooth (`0e8d:3603`) has two bulk OUT endpoints (0x01 and 0x02); `btusb` picks the
  first, but ACL data belongs on the second.
- **Fix**: patch `0002` for `btusb`.
</details>

<details>
<summary><b>"security block" on the controller channel</b></summary>

- **Cause**: the Salina controller has no interrupts and is polled roughly every 20 ms. The DualSense channel
  request arrives before the "Encryption Change" event, even though the key has already been delivered, and Linux
  rejects it.
- **Fix**: patch `0003` for L2CAP: in that case it answers "pending" and accepts the channel as soon as encryption
  is active.
</details>

<details>
<summary><b>A second, "ghost" Bluetooth adapter</b></summary>

- **Cause**: interfaces 3 and 4 of the chip show up as another adapter that fails to reset.
- **Fix**: a udev rule that disables them.
</details>

<details>
<summary><b>Pairing over USB cable, like PS5 OS</b></summary>

- `ludemos-dualsense` reads the controller's address over USB (HID report 0x09), writes the console address and a
  new link key into it (report 0x0A), stores the key in BlueZ's format and restarts `bluetoothd`. A udev rule runs
  it when the cable is plugged in and skips controllers that are already paired.
- PS5 OS uses the **same Bluetooth address** as the console: to tell whether re-pairing is needed, the tool checks
  whether the controller actually connected over Bluetooth during this power-on.
</details>

### Fan, temperatures, clocks

<details open>
<summary><b>The Slim fan ignores commands</b></summary>

- **Cause**: the Slim board (family 05) is unknown to the driver, which doesn't load its table and leaves the
  factory target at 91 °C; the only effective command is the temperature target written once the system is up.
- **Fix**: patch `0004`: an 80 °C boot target written and read back every 2 seconds until the firmware confirms it
  (with the correct mask for the GDDR6 revision). Then `ludemos-fand` holds 75 °C and rewrites it if the firmware
  changes it.
</details>

<details>
<summary><b>CPU stuck at 3.2 GHz even on the desktop</b></summary>

- **Cause**: `amdgpu` doesn't know Sony's power management firmware (with `dpm=1` the HDMI bridge locks up), so the
  firmware decides; the driver only offered "boost on/off".
- **Fix**: patch `0005`: `/sys/class/misc/mp1/intervallo` to set CPU and GPU frequency ranges. Measured that the
  firmware always holds the top of the range; idle set to 1600–1800 / 800–1000 MHz: from 54 °C down to **43 °C**
  at idle, desktop still smooth.
- `ludemos-fand` switches to boost when it finds a game (Steam processes or the launcher's variable) and returns to
  idle 30 seconds after it exits.
</details>

<details>
<summary><b>Non-Steam games stayed at idle clocks</b></summary>

- **Cause**: the daemon only recognised games started by Steam.
- **Fix**: the launcher sets `LUDEMOS_GIOCO=1` in the game's environment and `ludemos-fand` looks for it.
</details>

<details>
<summary><b>M.2 SSD without a heatsink</b></summary>

- 59–60 °C at idle. `ludemos-fand` lowers the fan target to 65 °C when the NVMe goes above 75 °C and returns to
  normal below 68 °C.
</details>

### Video, VRAM and audio

<details open>
<summary><b>Gamescope crashes after a few minutes of play ("fatal flip error")</b></summary>

- **Analysis**: core dump examined with gdb, symbols from debuginfod and libraries copied from the console: `ENOMEM`
  in the DRM commit.
- **Cause**: on this GPU scanout buffers can only live in VRAM, and with 512 MB and 4K output VRAM ran out.
- **Fix**: 1 GB of VRAM (`vram.txt` = `0x40000000`) and 10 GiB GTT: Control in DX12 with no crashes.
</details>

<details>
<summary><b>"No signal" after switching the TV off and on</b></summary>

- **Cause**: the kernel re-reads the EDID but doesn't notify KDE; the HDMI chip is left without a video
  configuration.
- **Fix**: the `ludemos-schermo` service follows the kernel log and, after the last EDID read, switches to a nearby
  refresh rate and back, reconfiguring the chip.
</details>

<details>
<summary><b>Choppy HDMI audio</b></summary>

- **Fix**: patch `0006`, ported from the ps5-linux community fix for the FLAVA3 HDMI chip (different `Naud` value
  in the audio encoder), plus a log line reporting the detected chip.
</details>

<details>
<summary><b>HDR: washed-out picture, the TV never switched to HDR</b></summary>

- **Analysis**: KDE reported the screen as "HDR incapable" because the base image disables it on purpose (the PS5
  routes DisplayPort through an HDMI converter chip). With that removed, the GPU sent a proper HDR signal, but the
  picture was washed out: the HDMI chip was still configured for SDR and never told the TV it was HDR.
- **Fix**: patch `0007`, adapted from the ps5-linux community work on the FLAVA3 chip: the HDR metadata packet
  (DRM InfoFrame) and BT.2020 colorimetry are written to the chip, and in HDR the output becomes 12-bit YCbCr 4:2:2,
  which fits the same HDMI bandwidth as 8-bit RGB. The launcher then passes HDR through Gamescope to the games.
  Result: the TV shows HDR10 with correct colours.
</details>

<details>
<summary><b>Wine games tiny or at the wrong resolution on the 4K TV</b></summary>

- KDE uses 2× scaling and Xwayland only exposes native 4K and 4:3 modes: in exclusive full screen games can't find
  1920×1080 and fall back to full 4K. The presets use borderless mode with a lower render resolution; the launcher
  sets Wine's DPI (192 on the desktop, 96 inside Gamescope).
</details>

### Games and controllers

<details open>
<summary><b>The character keeps walking on its own</b></summary>

- **Cause**: with some controllers, Proton reading the pad directly through SDL outside Steam Input gets wrong
  axes; inside a nested Gamescope, on the other hand, Steam Input doesn't deliver input to the game at all.
- **Fix**: the launcher's virtual gamepad, with the same identity and variables as Steam's virtual gamepad; Steam
  games start without nested Gamescope.
</details>

<details>
<summary><b>60 fps on the counter but "uneven" motion</b></summary>

- **Cause**: two frame limiters at once (the game's VSync and the DXVK/VKD3D or Gamescope limit).
- **Fix**: the presets keep only the game's VSync.
</details>

<details>
<summary><b>"Quit game" left the game running</b></summary>

- **Cause**: processes inside Steam's container (pressure-vessel) detach from their parent.
- **Fix**: the launcher refreshes the set of game PIDs every 2 seconds (detached ones included) and on quit sends
  SIGTERM to all of them, then SIGKILL after 5 seconds.
</details>

<details>
<summary><b>The launcher didn't exit at shutdown</b></summary>

- **Cause**: SDL intercepts SIGTERM.
- **Fix**: `SDL_HINT_NO_SIGNAL_HANDLERS`.
</details>

<details>
<summary><b>The PS button didn't bring the launcher back after "Desktop"</b></summary>

- **Cause**: on Wayland Qt can't tell whether the window is minimised.
- **Fix**: an internal flag plus a check on the active window.
</details>

### Boot, updates, session

<details open>
<summary><b>A kernel update left the controller without a driver</b></summary>

- **Cause**: upgrading `linux-ps5-ludemos-agg` made pacman remove the running kernel's modules; on the next pad
  hot-plug `xpad` was gone.
- **Fix**: `ludemos-moduli-salva` / `ludemos-moduli-rimetti` pacman hooks that keep the running kernel's modules.
</details>

<details>
<summary><b>The boot menu browser wouldn't close</b></summary>

- **Analysis**: the PS5 browser lives inside the home screen process (`SceShellUI`): killing it from the payload
  restarts the home screen. Five versions of the menu were tried.
- **Fix**: no closing from the payload; the page closes with the ◯ button (a user gesture the browser accepts), as
  Payload Manager does.
</details>

<details>
<summary><b>Other fixes</b></summary>

- Boot menu showing a blank page: the single-connection server stalled on the browser's preconnections → one
  thread per connection.
- Steam: started with `-steamos3` (otherwise "Switch to Desktop" appears to hang) plus small stand-ins for
  `steamos-update` / `steamos-select-branch`.
- Clock: under Linux the PS5 starts at 1970 → `systemd-timesyncd`.
- On-screen keyboard in KDE: Onboard with a KWin rule (exact class "Onboard", no focus, always on top).
- distrobuilder in an unprivileged LXC container: a patch and a private mount namespace, otherwise unmounting
  stripped `/dev` devices from the container.
- Busybox initrd: real `tar`, `blkid`, `mkfs.ext4` with full paths (the busybox versions lost capabilities and
  xattrs).
- System identity: the base image reported itself as "Arch Linux"; now it is LudemOS, and a pacman hook restores it
  after base-system updates.
</details>

---

## Tested games

Tested on the console, on a 4K 60 Hz TV, from the LudemOS Home launcher with Proton-CachyOS.

| Game | API | Resolution | Upscaler | Result |
|---|---|---|---|---|
| **Control** | DX12 | 1080p | **LudemNR** | **locked 60 fps** |
| **The Witcher 3: Wild Hunt** (Remastered, next-gen) | DX12 | 1080p (rendered at 960×540) | **LudemNR** replacing XeSS | **locked 60 fps**, sharp image, no ghosting |

---

## Measurements

**Clocks and temperatures** at idle (CPU and GPU are one chip, Tctl sensor), about 2 minutes per step:

| CPU / GPU range (MHz) | Actual CPU | Chip |
|---|---|---|
| Sony firmware default | 3187 MHz | 54–55 °C |
| 2400–3200 / 1600–2000 | 3188 MHz | 55 °C |
| 1600–2400 / 800–1400 | ~2500 MHz | 46 °C |
| **1600–1800 / 800–1000** (chosen idle) | ~1960 MHz | **43–45 °C** |

In game (Control): boost within seconds, CPU at 3490 MHz, chip at 53–74 °C, 60 fps.

**GPU** (gfx1013, RDNA 1 with ray tracing instructions): measured peak **9.9 TFLOPS FP32** and **20.1 TFLOPS
FP16**, matching the theoretical figures; int8 dot product is not accelerated (which is why XeSS is slow and FSR 4
is not possible).

---

## LudemNR: a neural upscaler for the PS5 GPU

Research work, not yet integrated into the launcher.

The idea: something like DLSS, but written for the PS5 GPU, in every game that already has a temporal upscaler
(DLSS, FSR 2/3, XeSS). The inputs (low-resolution colour, depth, motion vectors) come from **OptiScaler**, extended
with a `ludemnr` module.

- **Our own network**, small, trained with PyTorch on synthetic data we generate (no third-party weights).
- **Runs in Vulkan inside the D3D12 command list** through vkd3d-proton interop: going through DXIL, the driver
  doesn't use packed FP16 instructions and throughput halves.
- **Temporal accumulation** in D3D12 passes (reprojection, Gaussian weights relative to the nearest sample, colour
  clamping box), verified against the PyTorch reference (62 dB: FP16 rounding only).
- **Built from a Linux PC** with clang-cl, lld-link and the Windows SDK; found and fixed a case where clang reduced a
  function to a single `int3` instruction (the game crashed at startup).

| U-Net, 540p → 1080p | Cost on PS5 (boost) |
|---|---|
| minimal, 8 channels | **1.0 ms** |
| small, 16 channels | 3.5 ms |
| large, 32 channels | 17.2 ms |

Current result on an unseen test scene (960×540 → 1080p): bilinear 26.8 dB, LudemNR **30.1 dB in 0.32 ms**. In
games: **Control** and **The Witcher 3 Remastered** in DX12 at 1080p and **locked 60 fps** with LudemNR (see
[Tested games](#tested-games)).

---

## Known limitations

- Every Linux boot goes through the exploit: it depends on security researchers' work on firmware 13.60.
- Only 60 Hz is guaranteed (the DP→HDMI bridge exposes three mode families).
- Switching between PS5 OS and Linux requires re-pairing the DualSense: the console uses the same Bluetooth address
  with different keys.
- GPU load can't be read (the driver's power management is off for compatibility with Sony's firmware).

---

## Credits

LudemOS exists thanks to the work of other open source projects:

- [ps5-linux](https://github.com/ps5-linux): loader, kernel patches and PS5 drivers, `m2_init` tools, base images,
  HDR support for the FLAVA3 HDMI chip.
- [ps5-payload-dev](https://github.com/ps5-payload-dev): payload SDK and `elfldr`.
- [Relapse](https://github.com/ntfargo/Relapse-Exploit) (ntfargo and contributors): WebKit and kernel exploit.
- [WebKit Autoloader](https://github.com/itsPLK/ps5-webkit-autoloader) and
  [Payload Manager](https://github.com/itsPLK/ps5-payload-manager) (itsPLK): the app and menu approach.
- [j4lenj/ps5-mt7921-wifi](https://github.com/j4lenj/ps5-mt7921-wifi): first MT7921 support on the Slim.
- [CachyOS](https://cachyos.org), [Mesa](https://mesa3d.org) (PS5 GPU support),
  [umu-launcher](https://github.com/Open-Wine-Components/umu-launcher), Proton,
  [OptiScaler](https://github.com/optiscaler/OptiScaler), KDE Plasma, SDL, Qt.

---

## Notice

This page presents the work done so far. The source code, scripts and system files are not published yet: they
will be released later, in this repository.

Personal project, not affiliated with Sony Interactive Entertainment. Game artwork in the screenshots is shown only
to demonstrate the launcher and belongs to its respective owners.
