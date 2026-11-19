# Hyprland Remote Desktop Portal


## About this fork

This is a fork of [gac3k/xdg-desktop-portal-hypr-remote](https://github.com/gac3k/xdg-desktop-portal-hypr-remote). On top of upstream it adds:

- A port to sdbus-c++ 2.x.
- A layout clamp for the EIS regions.
- Clipboard sharing through `org.freedesktop.impl.portal.Clipboard`, for text and `image/png`.
- Quieter logging (no per-event output).
- Fixes for file descriptor leaks (`ConnectToEIS`) and for the init-failure path.

**Tested with:** Synergy 3.9.1 (Deskflow 1.26) on Hyprland 0.56.2. It should work with any RemoteDesktop portal client, but only that combination has been tested.

**Security caveat (from upstream):** the portal approves every remote-input request without showing a prompt. Anything that can talk to it on your session bus can get input access.

**Known issues:**

- Non-ASCII text copied from a Mac to Linux arrives garbled. This looks like Synergy's own conversion; laptop to Mac is exact.
- About 2 file descriptors are leaked per Synergy reconnect, and a harmless "Failed to add device" line is logged on disconnect.

## Build and install (this fork)

Dependencies: cmake, pkg-config, a C/C++ compiler, wayland, wayland-protocols, libei (libei and libeis), sdbus-c++ 2.x, systemd.

`DEVELOPMENT_MODE` defaults to ON, which uses a `.dev` D-Bus name. Turn it off for a real install:

```bash
cmake -B build -DDEVELOPMENT_MODE=OFF -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBEXECDIR=lib
cmake --build build
sudo cmake --install build
```

Note that `cmake --install` puts the binary in `bindir`, while the service files expect it in `libexecdir` (`@LIBEXECDIR@/xdg-desktop-portal-hypr-remote`). Packagers should install it there (for example, move it to `/usr/lib/`). The install step also installs `data/hypr-remote.portal` into `share/xdg-desktop-portal/portals/`. It does not install the service files; generate them from the templates and copy them yourself:

- D-Bus service: `build/org.freedesktop.impl.portal.desktop.hypr-remote.service` to `/usr/share/dbus-1/services/`
- systemd user unit: `build/contrib/systemd/xdg-desktop-portal-hypr-remote.service` to `/usr/lib/systemd/user/`

Then route the interfaces to this portal in `hyprland-portals.conf` (for example `~/.config/xdg-desktop-portal/hyprland-portals.conf`):

```ini
[preferred]
org.freedesktop.impl.portal.RemoteDesktop=hypr-remote
org.freedesktop.impl.portal.Clipboard=hypr-remote
```

and restart: `systemctl --user restart xdg-desktop-portal`.

Arch packaging is not part of this repo.

> The older "System Installation" and "Quick Start" sections below come from upstream and describe a development build; prefer the steps above.

A complete implementation of the `org.freedesktop.impl.portal.RemoteDesktop` portal for Hyprland, providing remote desktop capabilities using libei and Wayland virtual input protocols.

#### **System Installation (upstream, legacy)**

```bash
# Install portal system-wide
sudo make install

# Copy portal configuration
sudo cp data/hyprland.portal /usr/share/xdg-desktop-portal/portals/

# Restart portal services
systemctl --user restart xdg-desktop-portal
```

## 🚀 Quick Start

### Manual Build

Install dependencies:
- cmake, pkg-config, gcc
- wayland-client, wayland-protocols, wayland-scanner
- libei-1.0, sdbus-c++, systemd

Then:
```bash
./build.sh
```
## 🔧 Troubleshooting

### ✅ "Permission denied" D-Bus Errors - SOLVED
- **Fixed**: Now uses session bus instead of system bus
- **Fixed**: Correct method signatures prevent registration errors

### ✅ "File exists" D-Bus Errors - EXPECTED
- **Normal**: Occurs when another portal uses the same service name
- **Solution**: Use development mode (automatic in `test_portal.sh`)

### Testing Portal Integration

Check if the portal is discoverable:

```bash
# List all portal services
busctl --user list | grep portal

# Monitor D-Bus calls to your portal
busctl --user monitor org.freedesktop.impl.portal.desktop.hyprland.dev

# Test method calls
busctl --user call org.freedesktop.impl.portal.desktop.hyprland.dev \
  /org/freedesktop/portal/desktop \
  org.freedesktop.impl.portal.RemoteDesktop \
  CreateSession 'a{sv}' 0
```
## 📁 Project Structure

```
hyprland-remote-desktop/
├── src/
│   ├── main.cpp                    # Main application entry point
│   ├── portal.cpp/.h               # D-Bus portal implementation
│   ├── wayland_virtual_keyboard.cpp/.h  # Virtual keyboard protocol
│   ├── wayland_virtual_pointer.cpp/.h   # Virtual pointer protocol
│   └── libei_handler.cpp/.h        # LibEI event processing
├── protocols/
│   ├── virtual-keyboard-unstable-v1.xml      # Wayland keyboard protocol
│   └── wlr-virtual-pointer-unstable-v1.xml   # wlroots pointer protocol
├── data/
│   ├── hyprland.portal                        # Portal configuration
│   └── org.freedesktop.impl.portal.desktop.hyprland.service.in
├── shell.nix                       # NixOS development environment
├── CMakeLists.txt                  # Build configuration
├── build.sh                        # Build script
├── test_portal.sh                  # Development testing script
└── README.md                       # This file
```

## 🧪 Testing Commands

```bash
# Build and test
./test_portal.sh

# Manual testing
nix-shell
./build.sh
./build/hyprland-remote-desktop

# D-Bus testing
busctl --user introspect org.freedesktop.impl.portal.desktop.hyprland.dev /org/freedesktop/portal/desktop
busctl --user call org.freedesktop.impl.portal.desktop.hyprland.dev /org/freedesktop/portal/desktop org.freedesktop.impl.portal.RemoteDesktop CreateSession 'a{sv}' 0
```

## 🤝 Contributing

1. Use the provided `shell.nix` for development
2. Follow the existing code structure
3. Test with `./test_portal.sh`
4. Ensure both keyboard and pointer protocols work
5. Verify D-Bus method signatures match the portal specification

## 📄 License

MIT License, see the [LICENSE](LICENSE) file. Upstream's README states MIT but upstream never committed a LICENSE file, so this fork's LICENSE was added based on that statement, with the original author's copyright (2025 gac3k) retained.
