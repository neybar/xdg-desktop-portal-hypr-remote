# Hyprland Remote Desktop Portal

## Why this exists

The goal: drive a Hyprland machine from another computer with [Synergy](https://symless.com/synergy) or [Deskflow](https://github.com/deskflow/deskflow) — sharing one keyboard and mouse across machines, with copy/paste that crosses between them. On Hyprland, out of the box, **none of that worked.**

On Wayland, Synergy/Deskflow inject input and sync the clipboard through `xdg-desktop-portal`, which hands those requests to a desktop-specific backend. GNOME and KDE ship one; Hyprland relies on community backends, and this one ([upstream](https://github.com/gac3k/xdg-desktop-portal-hypr-remote)) is the RemoteDesktop backend for it. Three things stood in the way:

1. **The backend wouldn't run on a current system.** It was written against the old sdbus-c++ 1.x API, so it failed to build against sdbus-c++ 2.x (what Arch/Omarchy ship). With no backend running, Synergy had no RemoteDesktop portal to talk to — input didn't work at all.
2. **Pointer tracking was broken across mismatched monitors.** Even once it ran, the cursor fell into dead zones between monitors of different sizes, because the EI regions were merged into one bounding box.
3. **No clipboard.** The backend implemented keyboard and pointer input but not the `Clipboard` portal interface, so copy/paste never crossed between machines.

This fork fixes all three: it ports the backend to sdbus-c++ 2.x so it builds and runs, gives Synergy a pointer region that matches the real monitor layout, and adds the missing `Clipboard` implementation — plus a round of hardening and reliability fixes (see below). The result is keyboard, mouse and clipboard working seamlessly across linked machines on Hyprland.

It was built and tested on an [Omarchy](https://omarchy.org) laptop driven from a Mac over Synergy, but nothing in it is Omarchy- or Mac-specific: **any Hyprland user with the same setup should benefit.**

## About this fork

This is a fork of [gac3k/xdg-desktop-portal-hypr-remote](https://github.com/gac3k/xdg-desktop-portal-hypr-remote). On top of upstream it adds:

- A port to sdbus-c++ 2.x.
- A layout clamp for the EIS regions.
- Clipboard sharing through `org.freedesktop.impl.portal.Clipboard`, for text and `image/png`.
- Quieter logging (no per-event output).
- EIS through libeis's fd backend: no socket or lock files in `/tmp`, no bridge thread, and EIS threads stopped cleanly at shutdown.
- No file descriptor or EIS object leaks across client reconnects: devices follow the capabilities the client binds and are released on disconnect.
- Hardening from a security review: an app-ID allow-list (see below), clipboard MIME-type validation and a cap on pending pastes, and sanitised pointer values.
- Thread-safety fixes (one lock for all input paths, signal-safe shutdown), a fix for vertical smooth scroll, and no `hyprctl` calls during pointer motion.

**Tested with:** Synergy 3.9.1 (Deskflow 1.26) on Hyprland 0.56.2. It should work with any RemoteDesktop portal client, but only that combination has been tested.

**Security caveat:** the portal approves remote-input and clipboard requests without a prompt, but only from host apps (`app_id` empty), such as Synergy run as a systemd user service. Sandboxed apps (Flatpak, Snap) are refused unless their app ID is listed in `HYPR_REMOTE_ALLOWED_APP_IDS` (comma-separated; an empty item means host apps; `*` allows all). Set it with `systemctl --user edit xdg-desktop-portal-hypr-remote`, adding `[Service]` / `Environment=HYPR_REMOTE_ALLOWED_APP_IDS=,org.example.App`. Any unsandboxed process on your session bus can still call the portal directly.

**Unicode:** text is passed through byte for byte. A Mac-to-Linux test with Greek, Cyrillic, CJK, RTL scripts, emoji (including ZWJ sequences and flags), non-BMP characters, combining marks and zero-width characters arrived exact. If you test with `pbcopy` over ssh, set `LANG=en_US.UTF-8` first, or `pbcopy` itself garbles UTF-8.

## Build and install (this fork)

Dependencies: cmake, pkg-config, a C/C++ compiler, wayland, wayland-protocols, libei (libei and libeis), sdbus-c++ 2.x, systemd.

`DEVELOPMENT_MODE` (default ON) only adds a compile definition; the D-Bus name is always `org.freedesktop.impl.portal.desktop.hypr-remote`. A release install:

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

### Arch Linux

A `PKGBUILD` is provided in [`packaging/arch/`](packaging/arch/PKGBUILD). It is a `-git` package that builds from the current `main`:

```bash
cd packaging/arch
makepkg -si
```

It installs the binary, the `Clipboard`/`RemoteDesktop` portal file, the D-Bus activation file and the systemd user unit, then restart `xdg-desktop-portal` as above. It `provides`/`conflicts` `xdg-desktop-portal-hypr-remote`, so it replaces any other build of this backend.

## Development build

For hacking on the backend without installing it system-wide. A Nix dev shell with the dependencies is provided (`shell.nix`); or install them from your distro (see the dependency list above).

```bash
./build.sh          # configures and builds into build/
```

`test_portal.sh` builds, then installs a development D-Bus service and portal file under `~/.local/share` that point at `build/xdg-desktop-portal-hypr-remote`, so the backend runs from the build tree. The D-Bus name is always `org.freedesktop.impl.portal.desktop.hypr-remote`.

Check that the backend is reachable and exercise it directly:

```bash
# Is it registered on the session bus?
busctl --user list | grep hypr-remote

# Watch the D-Bus traffic to it
busctl --user monitor org.freedesktop.impl.portal.desktop.hypr-remote

# Call a method directly
busctl --user call org.freedesktop.impl.portal.desktop.hypr-remote \
  /org/freedesktop/portal/desktop \
  org.freedesktop.impl.portal.RemoteDesktop \
  CreateSession 'a{sv}' 0
```

## Project structure

```
xdg-desktop-portal-hypr-remote/
├── src/
│   ├── main.cpp                           # Entry point, signal handling
│   ├── portal.cpp/.h                      # D-Bus RemoteDesktop impl, EIS handling
│   ├── clipboard.cpp/.h                   # Clipboard impl (ext-data-control-v1)
│   ├── wayland_virtual_keyboard.cpp/.h    # Virtual keyboard protocol
│   ├── wayland_virtual_pointer.cpp/.h     # Virtual pointer protocol
│   └── libei_handler.cpp/.h               # libei/libeis event processing
├── protocols/
│   ├── ext-data-control-v1.xml            # Clipboard protocol
│   ├── virtual-keyboard-unstable-v1.xml   # Wayland keyboard protocol
│   └── wlr-virtual-pointer-unstable-v1.xml# wlroots pointer protocol
├── data/
│   └── hypr-remote.portal                 # Portal interface declaration
├── contrib/systemd/
│   └── xdg-desktop-portal-hypr-remote.service.in   # systemd user unit template
├── org.freedesktop.impl.portal.desktop.hypr-remote.service.in  # D-Bus service template
├── shell.nix                              # Nix development environment
├── CMakeLists.txt                         # Build configuration
├── build.sh                               # Development build script
└── test_portal.sh                         # Development run/test script
```

## Contributing

1. Build with `./build.sh` (or the Nix shell in `shell.nix`).
2. Run from the build tree with `./test_portal.sh`.
3. Test keyboard, pointer and clipboard against a real RemoteDesktop client.
4. Keep D-Bus method signatures matching the portal specification.

## License

MIT License, see the [LICENSE](LICENSE) file. Upstream's README states MIT but upstream never committed a LICENSE file, so this fork's LICENSE was added based on that statement, with the original author's copyright (2025 gac3k) retained.
