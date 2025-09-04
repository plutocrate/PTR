# Raspberry Pi Pico W SDK on Arch (detailed step‑by‑step)

This is a **detailed, copy‑ready** guide that records everything I did to make the Pico SDK + examples work on Arch (Hyprland). It explains *why* each step matters, the exact commands I ran, common failure modes and how to fix them, and a short troubleshooting checklist you can use later.

> Read it in a reader later — it’s written to be self‑contained.

---

# TL;DR (one‑liner summary)

1. Install the toolchain & build tools.
2. Clone `pico-sdk` and `pico-examples`.
3. Export `PICO_SDK_PATH` in your shell.
4. Initialize SDK submodules (`git submodule update --init --recursive`).
5. `cmake -DPICO_BOARD=pico_w ..` and `make -C pico_w/blink`.
6. Copy `blink.uf2` to the Pico device (`/run/media/$USER/RPI-RP2/`) while holding BOOTSEL.

If you see “Skipping Pico W examples” in CMake output, **submodules were missing** — see the detailed steps below.

---

# 1) Packages / prerequisites (Arch)

Minimum packages I installed (adjust if you already have some):

```bash
sudo pacman -S --needed base-devel cmake git python arm-none-eabi-gcc openocd
```

Useful extras (recommended):

```bash
sudo pacman -S picocom minicom gdb-multiarch
```

Notes:
- On Arch the GCC cross‑toolchain binary appears as `arm-none-eabi-gcc` (the package name can vary across distributions; on Debian/Ubuntu it is usually `gcc-arm-none-eabi`).
- `openocd` only matters if you plan to debug via SWD (and have a hardware probe). For simple UF2 flashing it's not required.

---

# 2) Create working directory and clone repos

I prefer `~/pico` as the workspace:

```bash
mkdir -p ~/pico
cd ~/pico
git clone https://github.com/raspberrypi/pico-sdk.git
git clone https://github.com/raspberrypi/pico-examples.git
```

Why two repos?
- `pico-sdk` is the SDK (drivers, headers, CMake glue).
- `pico-examples` contains small example projects that use the SDK.

Both must be available for the examples to configure and build correctly.

---

# 3) Tell CMake where the SDK lives (PICO_SDK_PATH)

Set the environment variable **once** so CMake finds the SDK:

```bash
# add to your shell rc so it persists
echo 'export PICO_SDK_PATH="$HOME/pico/pico-sdk"' >> ~/.bashrc
# or for zsh: >> ~/.zshrc
source ~/.bashrc

# quick check
echo $PICO_SDK_PATH
# expected output: /home/<youruser>/pico/pico-sdk
```

Why this matters: CMake will look for `pico_sdk_import.cmake` via `PICO_SDK_PATH`. If this isn't set CMake will not configure examples properly.

---

# 4) **Critical**: initialize SDK submodules

When the SDK repo is cloned, several optional components live in submodules. These are required for Pico W support (Wi‑Fi, USB stacks, etc.).

From the SDK directory run:

```bash
cd ~/pico/pico-sdk
git submodule update --init --recursive
```

What this pulls in (examples):
- **tinyusb** — USB device/host stack (used by many examples and some host features).
- **pico_cyw43_driver** + **lwip** + **btstack** — wireless and networking support that Pico W examples need.

If submodules are missing, CMake will warn with messages like:
```
TinyUSB submodule has not been initialized; USB support will be unavailable
Skipping Pico W examples as support is not available
```
This is exactly the message we observed — the fix is the `git submodule` command above.

---

# 5) Configure and build the examples (Pico W)

**Important:** always create a separate `build/` directory and point CMake at the source root (`..`).

```bash
cd ~/pico/pico-examples
rm -rf build        # start clean when debugging
mkdir build
cd build
cmake -DPICO_BOARD=pico_w ..
```

**What to look for in CMake output (diagnosis):**
- `Using PICO_SDK_PATH from environment ('/home/you/pico/pico-sdk')` — good.
- `Target board (PICO_BOARD) is 'pico_w'.` — good.
- Warnings about TinyUSB, cyw43, or LWIP — indicates submodules missing.
- `Skipping Pico W examples as support is not available` — shows CMake could not enable W examples.

If config stage looks good, build the specific example (faster):

```bash
# build only the pico_w blink example
make -C pico_w/blink -j$(nproc)
```

Or build everything (slower):

```bash
make -j$(nproc)
```

Expected artifact after the blink build:
```
build/pico_w/blink/blink.uf2
build/pico_w/blink/blink.elf
build/pico_w/blink/blink.bin
```

---

# 6) Flashing the UF2 to the Pico W

1. Hold the **BOOTSEL** button on the Pico W and plug it into USB.
2. The board should enumerate as a mass storage device called `RPI-RP2` (on most desktops it is auto‑mounted at `/run/media/$USER/RPI-RP2`).
3. Copy the UF2 file onto the device:

```bash
cp pico_w/blink/blink.uf2 /run/media/$USER/RPI-RP2/
```

4. The Pico will reboot and run the newly flashed firmware. If the example is the Pico W blink example (which uses the cyw43 driver), the onboard LED should blink.

**If the device is not auto‑mounted:**

- Check kernel messages: `dmesg | tail`
- List block devices: `lsblk` (look for a small USB mass storage device)
- If it appears as `/dev/sdX`, you can mount it manually (rare because udisks typically auto‑mount on desktops):

```bash
sudo mount /dev/sdX1 /mnt
cp pico_w/blink/blink.uf2 /mnt/
sudo umount /mnt
```

Replace `/dev/sdX1` with the actual device node.

---

# 7) What I saw and how I diagnosed it (walk‑through of the log you posted)

The important lines from your CMake output were:

```
CMake Warning: TinyUSB submodule has not been initialized
CMake Warning: cyw43-driver submodule has not been initialized
Skipping Pico W examples as support is not available
```

Interpretation:
- The SDK needs several submodules for Pico W. Because those submodules were not checked out, `cmake` intentionally *skipped* the Pico W specific examples (including the blink example under `pico_w/`).

Action I recommended (and you ran):

```bash
cd ~/pico/pico-sdk
git submodule update --init --recursive
```

After that: reconfigure (`cmake -DPICO_BOARD=pico_w ..`) and build the blink example.

---

# 8) Picotool behaviour and how to avoid repeated builds

CMake may download/build a local copy of `picotool` (a tiny helper used to postprocess UF2s) into `build/_deps/` when it can't find an installed `picotool` on your system. This is fine; but if you want to avoid rebuilding it every project, do one of the following:

**Option A — Install picotool globally**

```bash
cd ~/pico
git clone https://github.com/raspberrypi/picotool.git
cd picotool
mkdir build && cd build
cmake ..
make -j$(nproc)
# then copy the binary somewhere in PATH
sudo cp picotool /usr/local/bin/
```

After that CMake will find the system `picotool` and won't rebuild it for each project.

**Option B — Reuse a shared directory by setting** `PICOTOOL_FETCH_FROM_GIT_PATH` in your environment. (Advanced; not covered here.)

---

# 9) Minimal helpful scripts (copy/paste)

**Quick setup script (review before run)**

```bash
# ~/pico/setup_pico_arch.sh
#!/usr/bin/env bash
set -e
sudo pacman -S --needed base-devel cmake git python arm-none-eabi-gcc openocd
mkdir -p ~/pico
cd ~/pico
[ -d pico-sdk ] || git clone https://github.com/raspberrypi/pico-sdk.git
[ -d pico-examples ] || git clone https://github.com/raspberrypi/pico-examples.git
# export SDK path
grep -q PICO_SDK_PATH ~/.bashrc || echo 'export PICO_SDK_PATH="$HOME/pico/pico-sdk"' >> ~/.bashrc
source ~/.bashrc
# init sdk submodules
cd ~/pico/pico-sdk
git submodule update --init --recursive
# build blink only
cd ~/pico/pico-examples
rm -rf build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -C pico_w/blink -j$(nproc)
```

Make it executable:

```bash
chmod +x ~/pico/setup_pico_arch.sh
# run it after reviewing content
# ~/pico/setup_pico_arch.sh
```

This script is deliberately conservative — it does not overwrite your existing repos if they exist.

---

# 10) How to create (or re‑create) a minimal Pico W blink project yourself

If you want to make a *small* project (instead of using `pico-examples`) you can follow the SDK template approach. The short recipe is:

1. Create a project folder (e.g. `~/pico/my-blink`).
2. Add `CMakeLists.txt` that imports the SDK and calls `pico_sdk_init()`.
3. Write `main.c` and link with the appropriate pico libraries.

**Note:** for Pico W you normally need the cyw43 architecture layer (the examples handle this for you). The easiest, least error‑prone route is to copy the `pico-examples/pico_w/blink` example as your starting point.

---

# 11) Useful troubleshooting checklist

- `echo $PICO_SDK_PATH` — must point to SDK root.
- `cd pico-sdk && git submodule status` — shows which submodules are initialized.
- Recreate a clean build dir (`rm -rf build && mkdir build && cd build`).
- `cmake -DPICO_BOARD=pico_w ..` — read its output carefully for "Skipping" messages.
- `make -C pico_w/blink -j1` to see a linear error log if parallel build hides errors.
- If UF2 doesn't appear: check `ls build/pico_w/blink/` and `dmesg | tail` after plugging in the Pico.

---

# 12) Extra notes & caveats

- **VM/Ubuntu (Vagrant)**: If you use your Ubuntu VM for building, you must enable USB passthrough in VirtualBox (and install the Oracle Extension Pack) for the Pico to be visible to the VM when you hold BOOTSEL. When using a VM, flashing inside the VM is easiest: copy the UF2 from within the VM to the mounted `RPI-RP2` device.

- **Pico vs Pico W LED wiring**: non‑W Pico has the LED on GPIO 25 (simple). Pico W controls the board LED via the CYW43439 wireless chip — that's why Pico W examples rely on extra drivers/submodules.

- **OpenOCD & debugging**: if you plan to do SWD debugging, you need a hardware SWD probe (or a second Pico running picoprobe firmware) and possibly udev rules for the probe.

---

# 13) Final notes — why this is worth learning on Arch

Doing this manually on Arch is slightly more work, but it pays off:
- you understand the SDK/submodule layout and why components are optional
- you can reproduce and debug environment problems yourself
- when you move to a server/CI or build scripts later, you won't be surprised by implicit dependencies

If you prefer convenience, the Ubuntu VM route (official `pico_setup.sh`) gives a frictionless, book‑aligned environment. But since you want to stick to Arch — this guide contains everything you need.

---

If you want I can:
- Add a tiny `main.c` + `CMakeLists.txt` example for Pico W (copyable), or
- Produce a shortened one‑page cheat sheet with only the commands you will run often.

Tell me which and I’ll add it to the same file.

