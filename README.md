# IrisOS Build

A custom Debian-based live/installable Linux distribution built with `live-build` (Debian Live), targeting `debian/bookworm/amd64`.

## Overview

- **Base:** Debian bookworm, amd64
- **Desktop:** GNOME Core
- **Installer:** Calamares
- **Init system:** sysvinit
- **Image type:** iso-hybrid (BIOS + UEFI bootable)
- **Bootloaders:** syslinux (BIOS), grub-efi (UEFI)
- **Shell used for build:** fish

## Directory Structure

```
irisos-build/
├── auto/
│   └── config                  # Safe, repeatable lb config invocation
├── config/
│   ├── binary                  # Binary-stage options (bootloader, ISO metadata, etc.)
│   ├── common                  # Common live-build options
│   ├── package-lists/
│   │   ├── desktop.list.chroot     # Network stack (NetworkManager, wireless-tools, wpasupplicant)
│   │   ├── iris-core.list.chroot   # GNOME core, dev tools, embedded tools, productivity apps
│   │   ├── live.list.chroot        # live-boot, live-config
│   │   ├── my.list.chroot          # Calamares installer, syslinux-utils, libpam-systemd
│   │   ├── firmware.list.chroot    # Hardware firmware (WiFi, GPU, etc.)
│   │   └── kernel.list.chroot      # linux-image-amd64
│   ├── hooks/
│   │   └── normal/
│   │       ├── 0150-plymouth-theme.hook.chroot   # Activates the irisos Plymouth theme
│   │       └── 0200-compile-schemas.hook.chroot  # Compiles glib schema overrides
│   └── includes.chroot/
│       ├── etc/
│       │   ├── os-release                        # Custom OS identity (IrisOS 1.0)
│       │   ├── motd                               # Login welcome message
│       │   └── calamares/branding/irisos/
│       │       ├── branding.desc                  # Calamares branding config
│       │       ├── logo.png
│       │       └── welcome.png
│       └── usr/share/
│           ├── glib-2.0/schemas/
│           │   └── 99_irisos.gschema.override     # Default theme/icon/cursor/wallpaper
│           └── plymouth/themes/irisos/
│               ├── irisos.plymouth
│               ├── irisos.script
│               └── logo.png
└── chroot/                     # Generated chroot filesystem (build artifact, not source-controlled)
```

## Package Lists

| File | Contents |
|---|---|
| `desktop.list.chroot` | `network-manager`, `wireless-tools`, `wpasupplicant` |
| `iris-core.list.chroot` | `gnome-core`, `gnome-tweaks`, `gnome-shell-extension-prefs`, `neofetch`, `htop`, `btop`, `tree`, `build-essential`, `gdb`, `git`, `python3`, `python3-pip`, `python3-venv`, `gcc`, `g++`, `make`, `cmake`, `curl`, `wget`, `zip`, `unzip`, `nano`, `vim`, `geany`, `python3-serial`, `minicom`, `openocd`, `avrdude`, `firefox-esr`, `libreoffice-writer`, `libreoffice-calc`, `pdfarranger`, `evince`, `flatpak`, `calamares`, `calamares-settings-debian`, `plymouth`, `plymouth-themes` |
| `live.list.chroot` | `live-boot`, `live-config`, `syslinux-utils`, `libpam-systemd` |
| `my.list.chroot` | `orchis-gtk-theme`, `bibata-cursor-theme` |
| `firmware.list.chroot` | `firmware-linux`, `firmware-misc-nonfree`, `firmware-iwlwifi`, `firmware-realtek`, `firmware-amd-graphics` |
| `kernel.list.chroot` | `linux-image-amd64` |

## Theming

Configured via `config/includes.chroot/usr/share/glib-2.0/schemas/99_irisos.gschema.override`:

```ini
[org.gnome.desktop.interface]
color-scheme='prefer-dark'
gtk-theme='Orchis-Dark'
icon-theme='Papirus-Dark'
cursor-theme='Bibata-Modern-Classic'

[org.gnome.desktop.background]
picture-uri='file:///usr/share/backgrounds/irisos-default.jpeg'
picture-uri-dark='file:///usr/share/backgrounds/irisos-default.jpeg'
picture-options='zoom'

[org.gnome.shell]
favorite-apps=['firefox-esr.desktop', 'org.gnome.Terminal.desktop', 'code.desktop', 'libreoffice-writer.desktop', 'org.gnome.Nautilus.desktop']
```

- **GTK theme:** Orchis-Dark (from `orchis-gtk-theme`, Debian's own build — verified via chroot, differs from Ubuntu's package)
- **Icon theme:** Papirus-Dark (Tela was considered but is not packaged in bookworm)
- **Cursor theme:** Bibata-Modern-Classic (from `bibata-cursor-theme`; verified variants available: Modern-Amber, Modern-Classic, Modern-Ice, Original-Amber, Original-Classic, Original-Ice)

Schema overrides require compilation to take effect — handled by the `0200-compile-schemas.hook.chroot` hook, which runs `glib-compile-schemas /usr/share/glib-2.0/schemas/` during the chroot stage.

## Boot & Installer

### Plymouth (boot splash)
- Theme: `irisos` (custom), located at `config/includes.chroot/usr/share/plymouth/themes/irisos/`
- Files: `irisos.plymouth`, `irisos.script`, `logo.png`
- Activated via `0150-plymouth-theme.hook.chroot`, which runs `plymouth-set-default-theme -R irisos` during the chroot stage

### Calamares (installer)
- Branding directory: `config/includes.chroot/etc/calamares/branding/irisos/`
- `branding.desc` key settings:
  - `productName`: IrisOS
  - `versionedName`: IrisOS 1.0
  - `windowSize`: 800px, 520px
  - `sidebar`: widget (enabled, shows install step progress)
  - Sidebar colors: background `#2c3e50`, text `#ffffff`, selected `#2980b9`
  - `productLogo` / `productIcon`: `logo.png` (1024×1024)
  - `productWelcome`: `welcome.png` (currently 1024×1024 — square; window is 800×520 landscape, so this image is not yet cropped to match aspect ratio. **Known cosmetic issue, not yet fixed.**)

### OS Identity
- `/etc/os-release` — reports `IrisOS 1.0`, links to `irisos.org` and the project's GitHub
- `/etc/motd` — "Welcome to IrisOS"

## Build Configuration

### `config/binary` (key settings)
- `LB_IMAGE_TYPE="iso-hybrid"`
- `LB_BOOTLOADER_BIOS="syslinux"`, `LB_BOOTLOADER_EFI="grub-efi"`, `LB_BOOTLOADERS="syslinux grub-efi"`
- `LB_FIRMWARE_CHROOT="true"` (enabled to support WiFi/GPU firmware)
- `LB_DEBIAN_INSTALLER="none"` (Calamares used instead of debian-installer)
- `LB_COMPRESSION="xz"`

### `config/common` (key settings)
- `LB_MODE="debian"`, `LB_SYSTEM="live"`
- `LB_INITSYSTEM="sysvinit"`
- `LB_INITRAMFS="live-boot"`
- `LB_CACHE_STAGES="bootstrap"` (only bootstrap stage is cached between builds)

### `auto/config`
Wraps `lb config` with a fixed, repeatable set of flags so that `lb clean && lb config` never silently resets customizations:

```sh
#!/bin/sh
set -e
lb config noauto --architecture amd64 -b iso-hybrid --bootloaders "syslinux grub-efi" --debian-installer none --iso-application "IrisOS Live" --iso-volume "IrisOS bookworm" "$@"
```

**Note:** flag names depend on your installed live-build version. This project uses `-b`/`--binary-image` (singular) and a combined `--bootloaders "syslinux grub-efi"` flag — earlier attempts using `--binary-images` (plural) and separate `--bootloader-bios`/`--bootloader-efi` flags failed with "unrecognized option" on this version.

## Build Instructions

```fish
cd ~/irisos-build

# Full clean rebuild (safe — restores config via auto/config)
sudo lb clean
sudo lb config
sudo lb build
```

### Chroot inspection (manual apt/dpkg access)
The chroot has no network access or `/proc`, `/sys`, `/dev` by default when entered manually (outside of `lb build`'s own lifecycle). To manually run `apt-get`/`dpkg` inside it for package verification:

```fish
sudo cp /etc/resolv.conf ~/irisos-build/chroot/etc/resolv.conf
sudo mount --bind /proc ~/irisos-build/chroot/proc
sudo mount --bind /sys ~/irisos-build/chroot/sys
sudo mount --bind /dev ~/irisos-build/chroot/dev

sudo chroot ~/irisos-build/chroot apt-get update
# ... do work ...

sudo umount ~/irisos-build/chroot/proc
sudo umount ~/irisos-build/chroot/sys
sudo umount ~/irisos-build/chroot/dev
```

Always unmount before running `lb build` again.

## Build History Notes

- Initial build failed at `lb binary_linux-image` — `chroot/boot/vmlinuz-*` not found — caused by no `linux-image-*` package in any package list. Fixed by adding `kernel.list.chroot`.
- `lb clean --chroot` was found to leave stage markers inconsistent, causing `chroot: cannot change root directory to 'chroot': no such directory` on next build. Full `lb clean` (no args) resolved it, but subsequently required an `auto/config` script since `.build/config` stage marker was also cleared and this project originally had no `auto/config` (raw `lb config` would have reset custom settings in `config/binary`/`config/common`).
- Package name discrepancies caught during theming setup: `tela-icon-theme` does not exist in bookworm (kept Papirus-Dark instead); `orchis-gtk-theme` and `bibata-cursor-theme` do exist, but the Debian bookworm package versions differ from Ubuntu host-cache versions — always verify package details **inside the chroot**, not the host system's apt cache.



## Where to download using the website
IrisOS 1.0 is live: [https://irisos-two.vercel.app/]
Built from an empty directory. GNOME desktop, custom installer, full branding down to the boot splash.
Not a respin. Mine, start to finish.

## Where to download using GitHub
In this repo you can find "Releases"
Go there and with just a click you have your own iso downloading immediately.

<img width="1271" height="815" alt="image" src="https://github.com/user-attachments/assets/8b87444c-5c58-4825-abaf-244d2de6b838" />




## How it looks like
