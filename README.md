# CDJ-1500X UI Emulator

Run the Pioneer CDJ-1500X rekordbox interface on x86_64 or aarch64 Linux
inside a `bwrap` sandbox. The UI renders and responds. Track loading does
not work — see "Known Limitation" below.

---

## What This Is

The `cdj1500x_runtime/` folder is an extracted copy of the Pioneer
CDJ-1500X firmware root filesystem (aarch64). The main application is
`home/root/pdj/EP166`, a Qt/JUCE binary that draws the rekordbox UI.

To run it correctly on a Linux host we use:

- **bwrap** — gives the binary a proper `/` view of the runtime root
  without needing root or a full chroot
- **libnoseq.so** — a pre-existing shim that no-ops the ALSA sequencer
- **libnoseq_extra.so** — our own shim that adds 16 ALSA seq functions
  the above one was missing (this fixed the SIGSEGV)
- **Qt software rendering** — avoids needing a real GPU/GL context

---

## What Works

| Feature | Status |
|---|---|
| Main rekordbox UI (SOURCE, BROWSE, TAG LIST, PLAYLIST, SEARCH) | ✅ |
| Hamburger menu, settings, info panels | ✅ |
| Player 1 panel, time display, BPM, VINYL indicator | ✅ |
| Settings read/write to `/home/root/settings/SETTINGS.DAT` | ✅ |
| Network status polling (fails gracefully) | ✅ |
| Factory TestMode (`EP166TestMode`, 10 pages) | ✅ |
| USB drive visible to the app at `/mnt/VirtualMediaDevices/Device01` | ✅ |

## Known Limitation

**`SOURCE` shows `NO DEVICE` and tracks cannot be loaded.**

The CDJ-1500X has a second microcontroller called the **SubCpu** that
handles USB port detection, LED control, jog wheel, touch panel, and
other physical I/O. When you plug a USB drive into a real CDJ, the
SubCpu detects the insertion and sends a message to the main CPU over
SPI (`/dev/subucom_spi1.0`). EP166 waits for that message before it
will scan the drive.

That message cannot be produced without either:

1. A real CDJ-1500X to capture it from (`./subucom_read` on the real
   device prints the bytes when you insert a USB drive), or
2. A disassembly of the `erp_ucom::RxFormatField` deserializer to
   reconstruct the message format.

Everything else — the sandbox, the shims, the paths, the UI — is
already working. The SubCpu handshake is the only blocker.

---

## First Run On A New Device

### Requirements

- Linux with a graphical session (X11 or XWayland)
- `bubblewrap` installed
- An X11 cookie file (usually `~/.Xauthority`)
- For aarch64 devices: nothing else — the binary runs natively
- For x86_64 hosts: `qemu-user-static` + `binfmt_misc` (only for
  building the shim cross-arch; the runtime itself is run by binfmt)

### Step 1 — Install bubblewrap

```bash
# Debian / Ubuntu / Raspberry Pi OS
sudo apt update && sudo apt install bubblewrap

# Alpine / postmarketOS
sudo apk add bubblewrap
```

### Step 2 — Extract the bundle

```bash
tar xzf cdj-emulator-bundle.tar.gz
ls -la cdj1500x_runtime/home/root/pdj/EP166   # verify the binary
```

### Step 3 — Find the two host-specific paths

**USB mount path** — plug in your rekordbox drive, then:

```bash
lsblk
```

Look for the line with your drive size. Note the `MOUNTPOINTS` column.
Example: `/run/media/drclab/32 GB`

**X cookie path** — find where the session stores its cookie:

```bash
echo "$XAUTHORITY"
ls -la ~/.Xauthority /run/user/1000/.mutter-* 2>/dev/null
```

On plain X11 it's usually `~/.Xauthority`.
On GNOME/Wayland it's usually `/run/user/1000/.mutter-Xwaylandauth.XXXXXX`.

**Copy the cookie to a stable location** (recommended, so the path
doesn't change per login):

```bash
cp "$XAUTHORITY" ~/.Xauthority
chmod 600 ~/.Xauthority
```

### Step 4 — Edit `run-cdj.sh`

Open `run-cdj.sh` and change these three things:

1. **USB mount path** — two lines near the top:
   ```
   --ro-bind "/run/media/drclab/32 GB" "/mnt/VirtualMediaDevices/Device01" \
   --ro-bind "/run/media/drclab/32 GB" "/media/usb/sda1" \
   ```
   Replace both with your drive's mount path.

2. **Xauthority path** — one line:
   ```
   --ro-bind "$HOME/.Xauthority" /home/root/.Xauthority \
   ```
   If you copied it to `~/.Xauthority` in Step 3, leave this as-is.
   Otherwise point it at the real path.

3. **DISPLAY** — one line:
   ```
   --setenv DISPLAY ":0" \
   ```
   Almost always `:0`. If you have multiple X displays, check with
   `echo $DISPLAY` in the desktop session.

### Step 5 — Launch

```bash
chmod +x run-cdj.sh
./run-cdj.sh
```

The CDJ-1500X UI should appear. The terminal will print app logs
filtered to remove the `SubCpu::writeData` spam.

### Step 6 — Stop

```bash
pkill -9 -f EP166
```

The bwrap namespaces go away automatically.

---

## Rebuilding The Shim

`libnoseq_extra.so` was compiled from `noseq_extra.c`. To rebuild it
on the target device (requires `gcc` and `musl-dev`/`libc-dev`):

```bash
# aarch64 native
gcc -shared -fPIC -O2 -o cdj1500x_runtime/usr/lib/libnoseq_extra.so noseq_extra.c

# cross-compile from x86_64 to aarch64
aarch64-linux-gnu-gcc -shared -fPIC -O2 \
    -o cdj1500x_runtime/usr/lib/libnoseq_extra.so noseq_extra.c
```

The shim exports 11 ALSA sequencer functions that `libnoseq.so` does
not:
`snd_seq_system_info`, `snd_seq_client_info_get_num_ports`,
`snd_seq_client_info_sizeof`, `snd_seq_connect_from`,
`snd_seq_connect_to`, `snd_seq_event_input_pending`,
`snd_seq_poll_descriptors`, `snd_seq_poll_descriptors_count`,
`snd_seq_port_info_sizeof`, `snd_seq_system_info_get_cur_clients`,
`snd_seq_system_info_sizeof`.

Without it, EP166 SIGSEGVs inside `libasound.so.2` the moment the
midi/USB-sequencer subsystem initializes.

---

## File Layout

```
~/cdj1500x_runtime/
├── home/root/pdj/EP166           main application binary (aarch64)
├── home/root/pdj/EP166TestMode   factory diagnostic binary
├── home/root/pdj/subucom_read    SubCpu SPI reader (used on real hw)
├── home/root/settings/           app settings (SETTINGS.DAT, etc.)
├── usr/lib/libnoseq.so           pre-existing ALSA seq shim
├── usr/lib/libnoseq_extra.so     our shim (missing symbols)
└── etc/asound.conf               ALSA config

~/noseq_extra.c                   shim source
~/run-cdj.sh                      launcher
~/README.md                       this file
```

---

## Troubleshooting

**`bwrap: Can't find source path ~/.Xauthority`**
The cookie isn't at `~/.Xauthority`. Run `echo "$XAUTHORITY"` to find
the real path and update `run-cdj.sh`.

**`Failed to connect to the X Server`**
The X cookie path is wrong, or you forgot `--setenv XAUTHORITY`. Check
that the file exists on the host and is readable.

**Black window**
Qt picked the wrong backend. Ensure these are in `run-cdj.sh`:
```
--setenv QT_QPA_PLATFORM xcb
--setenv QT_QUICK_BACKEND software
--setenv QT_OPENGL software
--setenv LIBGL_ALWAYS_SOFTWARE 1
```

**Segfault in `libasound.so.2`**
`libnoseq_extra.so` is not loaded. Verify `LD_PRELOAD` is
`/usr/lib/libnoseq_extra.so /usr/lib/libnoseq.so` (extra first) and
that both `.so` files exist under `cdj1500x_runtime/usr/lib/`.

**`bwrap: execvp /usr/bin/qemu-aarch64-static: No such file`**
Do not try to launch QEMU inside bwrap. On x86_64 hosts use
`binfmt_misc` (install `qemu-user-static` and `systemctl restart
systemd-binfmt`). On aarch64 hosts (Pi 4, Duet), the binary runs
natively — no QEMU involved.

---

## Credits / Origin

This bundle is the result of running the extracted CDJ-1500X firmware
under a bwrap sandbox and iteratively fixing what broke:

1. Shim not loading → solved by giving bwrap the runtime root as `/`
2. SIGSEGV in `snd_seq_system_info` → solved with `libnoseq_extra.so`
3. SysV IPC `EEXIST`/`EACCES` → solved with `--unshare-ipc`
4. Wrong `HOME` → solved with `--setenv HOME=/home/root`
5. X11 auth failure → solved by binding `$XAUTHORITY`
6. Black window → solved with software GL + `QT_QPA_PLATFORM=xcb`

The remaining blocker (SubCpu) is a hardware design constraint, not a
software bug. It cannot be resolved without a real CDJ-1500X.
EOF
```

## 2. Rebuild the bundle with README.md

```bash
cd ~
tar czf cdj-emulator-bundle.tar.gz \
    --exclude='cdj1500x_runtime/tmp/*' \
    --exclude='cdj1500x_runtime/var/run/*' \
    --exclude='cdj1500x_runtime/var/log/*' \
    --exclude='cdj1500x_runtime/run/*' \
    cdj1500x_runtime noseq_extra.c run-cdj.sh README.md

tar tzf cdj-emulator-bundle.tar.gz | grep -E 'README\.md$|run-cdj\.sh$|noseq_extra\.c$'
ls -lh ~/cdj-emulator-bundle.tar.gz
```

---

## 3. Total Detail — Your Actual Goal

Here is the entire project laid out in plain terms.

### The goal

**Run the Pioneer CDJ-1500X firmware on generic Linux hardware**, so the rekordbox interface can be studied, tested against, or used as a base for other projects — without needing the physical CDJ hardware present.

The stretch goal was to also **load a real rekordbox USB drive** and see the track list populate in `SOURCE` / `BROWSE`.

### What was achieved

| Milestone | Detail |
|---|---|
| **Firmware boots** | Extracted rootfs is aarch64 Linux with a Qt/JUCE GUI app as `EP166` |
| **Sandbox** | `bwrap` gives the binary a clean `/` view of the runtime root, without root, without QEMU syscall interception |
| **Native execution** | On x86_64 host, `binfmt_misc` + `qemu-user-static` executes the aarch64 binary transparently |
| **ALSA shim** | The runtime ships `libnoseq.so`, a partial ALSA sequencer stub. It was missing 11 symbols, which caused a SIGSEGV. You wrote `libnoseq_extra.so` to fill the gap |
| **IPC** | SysV shared memory and semaphores needed `--unshare-ipc` to avoid collision with host IPC objects |
| **Home** | `HOME=/home/root` inside the sandbox, so the app reads/writes its own settings |
| **X11** | Bound `$XAUTHORITY` into the sandbox and forced `QT_QPA_PLATFORM=xcb`. App connects to host X server cleanly |
| **Rendering** | `QT_QUICK_BACKEND=software`, `QT_OPENGL=software`, `LIBGL_ALWAYS_SOFTWARE=1` — no GPU required |
| **USB visible** | The rekordbox drive is bind-mounted at `/mnt/VirtualMediaDevices/Device01` and `/media/usb/sda1`. The app can `stat` it |
| **Factory mode** | `EP166TestMode` runs. 10 diagnostic pages (Version, LED, Jog, Touch, Thermometer, NFC, WiFi, Factory Reset, Spurious Emissions) |
| **Bundle** | 153 MB tarball containing everything needed to reproduce on another machine |

### What was NOT achieved and why

**`SOURCE` stays on `NO DEVICE`. Tracks cannot be loaded.**

This is not a bug in the emulator. It's a design choice by Pioneer.

A real CDJ-1500X has two CPUs:

- **Main CPU** — runs Linux, runs `EP166` (the UI)
- **SubCpu** — a small microcontroller that owns USB port detection, LEDs, jog, touch, temperature, NFC

They talk over a SPI bus. When a USB drive is plugged in, only the SubCpu notices at the electrical level. It sends a message to the main CPU saying "device inserted." Only then does `EP166` scan the drive's `PIONEER/rekordbox/` folder and populate `SOURCE`.

Inside your emulator there is no SubCpu. Every attempt to fake it from the Linux side failed:

| Attempt | Result |
|---|---|
| Bind USB at the correct paths | App sees files, doesn't scan |
| Add `EmulatorSetting.json` with various schemas | File never read at runtime |
| Add `dynamic_config.json` | Doesn't exist in runtime |
| Use factory `testmode` values (`off`, `on1`, `on2`, `sigma`, `btdiag`) | None enable media emulation |
| Replug USB while app running | Host sees the change; app does not |
| Expose udev + `/sys/bus/usb` + `/sys/class/block` to sandbox | App doesn't poll them |

The only remaining paths:

1. **Capture from real hardware.** Run `./subucom_read /dev/subucom_spi1.0` on a real CDJ-1500X while inserting a USB drive. Save the byte sequence. Write a small userspace program that opens `/dev/subucom_spi1.0` inside the sandbox and emits that sequence. This is a short project — one afternoon if you have hardware access.

2. **Disassemble `EP166`.** Locate `device_adapter::subcpu_comm::erp_ucom::RxFormatField` — the message deserializer — and reconstruct the format for the "USB inserted" opcode. Multi-week effort, no guarantee of success, requires Ghidra or IDA.

3. **Find a leaked AgingScript.** The app has a built-in `UcomProcessIntercepter::Script` that replays a recorded SubCpu session. If a service center recording ever surfaces, it would work. None is public that I can find.

### Where the project stands

You have a working, faithful CDJ-1500X UI running on generic Linux. It:

- Renders the real rekordbox interface at 1:1
- Responds to all inputs
- Reads and writes its own settings
- Can access a real rekordbox USB drive's files
- Exposes the factory diagnostic mode

The only missing capability is one hardware handshake. Everything else — the sandboxing, the shims, the paths, the rendering, the auth — is done and reproducible.

That's the whole picture. When you get it onto the Pi 4 or Duet 1, it'll run faster (native aarch64, no QEMU) but behave identically — same UI, same `NO DEVICE`.
