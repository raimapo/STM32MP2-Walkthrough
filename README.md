# STM32MP257 — OpenSTLinux Build & Flash Guide

**Developer Documentation**

A complete guide to fetching, building, and flashing a Yocto-based Linux image on the STM32MP257F-EV1 evaluation board using the **Scarthgap** release.

- Ubuntu 22.04 LTS (recommended)
- Yocto Scarthgap
- Tag: `openstlinux-6.6-yocto-scarthgap-mpu-v26.02.18`

## Table of contents

- [Prerequisites](#prerequisites)
- [1. Install build dependencies](#1-install-build-dependencies)
- [2. Create the workspace directory](#2-create-the-workspace-directory)
- [3. Initialise repo and sync sources](#3-initialise-repo-and-sync-sources)
- [4. Configure the build environment and build](#4-configure-the-build-environment-and-build)
- [Build troubleshooting](#build-troubleshooting)
- [5. Install STM32CubeProgrammer](#5-install-stm32cubeprogrammer)
- [6. Configure USB and udev rules](#6-configure-usb-and-udev-rules)
- [7. Flash the board](#7-flash-the-board)
- [Adding, removing, and inspecting packages](#adding-removing-and-inspecting-packages)

## Prerequisites

Before starting, ensure you have the following available:

- A machine running **Ubuntu 22.04 LTS** (Ubuntu 24.04 works but may show warnings)
- At least **100 GB** of free disk space for the full Yocto build
- At least **8 GB RAM** (16 GB recommended for parallel builds)
- Fast internet connection — the `repo sync` step fetches several GB
- An **STM32MP257F-EV1** evaluation board with a USB cable
- An STMicroelectronics account (required to download STM32CubeProgrammer)

> ℹ️ All commands below assume you are running as a regular (non-root) user with `sudo` privileges. Do not run the Yocto build as root.

---

## 1. Install build dependencies

Update your package lists, then install all required host tools. These cover compilers, Python utilities, source-control helpers, and misc Yocto prerequisites.

```bash
sudo apt update
sudo apt-get install -y \
    gawk wget git diffstat unzip texinfo \
    gcc-multilib build-essential chrpath socat cpio \
    python3 python3-pip python3-pexpect \
    xz-utils debianutils iputils-ping \
    python3-git python3-jinja2 \
    libegl-dev libsdl1.2-dev pylint xterm \
    lz4 zstd curl repo bsdmainutils git-lfs \
    libegl1-mesa libgmp-dev libmpc-dev libssl-dev
```

### What each group installs

| Package(s) | Purpose |
|---|---|
| gawk, diffstat, unzip, texinfo | Text processing tools required by Bitbake |
| gcc-multilib, build-essential | Host compiler toolchain |
| chrpath, socat, cpio | ELF rpath patching, socket relay, archive utilities |
| python3, python3-pip, python3-pexpect, python3-git, python3-jinja2 | Python runtime and Bitbake Python dependencies |
| libegl-dev, libsdl1.2-dev | OpenGL/EGL and SDL development headers for Weston |
| lz4, zstd, xz-utils | Compression tools for image packaging |
| repo, git-lfs | Google Repo manifest tool; Git Large File Storage |
| libgmp-dev, libmpc-dev, libssl-dev | Math and crypto libraries used by the cross-compiler build |

---

## 2. Create the workspace directory

All build artefacts and source layers live inside a single directory. Create it in your home folder and change into it.

```bash
mkdir ~/stm32mp2-yocto && cd ~/stm32mp2-yocto
```

> ℹ️ You can choose a different path — just replace `~/stm32mp2-yocto` consistently in every subsequent command.

---

## 3. Initialise repo and sync sources

The STMicroelectronics OE manifest repo groups all required Yocto meta-layers into one versioned reference. Initialise it at the correct tag, then sync.

```bash
# Initialise the manifest at the v26.02.18 tag
repo init \
    -u https://github.com/STMicroelectronics/oe-manifest.git \
    -b refs/tags/openstlinux-6.6-yocto-scarthgap-mpu-v26.02.18

# Download all layer sources (may take 10–30 minutes)
repo sync
```

> ⚠️ **Do not interrupt `repo sync`.** Partial syncs can leave the tree in an inconsistent state. If it fails, re-run `repo sync` — it is safe to resume.

---

## 4. Configure the build environment and build

Source the ST environment setup script to initialise Bitbake with the correct machine and distro, then kick off the image build.

```bash
# Source the environment (must be done in every new shell session)
MACHINE=stm32mp25-eval DISTRO=openstlinux-weston \
    source layers/meta-st/scripts/envsetup.sh
```

This command changes your working directory to the build folder. Now choose which image to build:

```bash
# Minimal console image (faster, no Weston compositor)
bitbake st-image-core

# Full Weston Wayland GUI image
bitbake st-image-weston
```

> ℹ️ The first full build compiles a cross-compiler from scratch and can take **3–8 hours** depending on your CPU and disk speed. Subsequent builds are fast thanks to Bitbake's shared-state cache.

> ✅ Build output images are written to:
> `build-openstlinuxweston-stm32mp25-eval/tmp-glibc/deploy/images/stm32mp25-eval/`

---

## Build troubleshooting

### Error: locale / UTF-8 issue

Bitbake requires a UTF-8 locale. If you see locale-related errors, fix them with:

```bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

Add the two `export` lines to your `~/.bashrc` to persist across sessions.

### Error: unprivileged user namespaces (Ubuntu 24.04)

Ubuntu 24.04 ships with AppArmor restrictions that block the unprivileged user namespaces that Bitbake's pseudo uses. Disable the restriction:

```bash
# Apply for the current session
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0

# Persist across reboots
echo 'kernel.apparmor_restrict_unprivileged_userns=0' | \
    sudo tee /etc/sysctl.d/99-userns.conf
```

> ⚠️ This setting slightly reduces the kernel's AppArmor surface. It is acceptable on a dedicated build machine. Review your organisation's security policy before applying on shared systems.

---

## 5. Install STM32CubeProgrammer

STM32CubeProgrammer is the official ST tool for writing firmware and partition images to STM32 devices over USB (DFU) or JTAG.

1. **Download the installer** — Log in to [st.com](https://www.st.com), search for *STM32CubeProgrammer*, and download the Linux 64-bit installer ZIP (e.g. `SetupSTM32CubeProgrammer_linux_64.zip`).
2. **Install runtime dependencies** — The installer requires Java and libusb.

   ```bash
   sudo apt install libusb-1.0-0 openjdk-11-jre-headless unzip
   ```

3. **Extract and run the installer** — Unzip to a local directory, make the binary executable, and run it.

   ```bash
   unzip SetupSTM32CubeProgrammer_linux_64.zip -d STM32CubeProgrammer-setup
   cd STM32CubeProgrammer-setup
   chmod +x SetupSTM32CubeProgrammer-2.22.0.linux
   ./SetupSTM32CubeProgrammer-2.22.0.linux
   ```

   Follow the GUI installer. It installs to `~/STMicroelectronics/STM32Cube/STM32CubeProgrammer/` by default.

4. **Add the CLI tool to your PATH** — So you can call `STM32_Programmer_CLI` from any directory.

   ```bash
   echo 'export PATH=$HOME/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin:$PATH' \
       >> ~/.bashrc
   source ~/.bashrc
   ```

5. **Verify the installation**

   ```bash
   STM32_Programmer_CLI --version
   # Expected output: STM32CubeProgrammer v2.22.0  (or similar)
   ```

---

## 6. Configure USB and udev rules

Without udev rules, accessing the STM32 device in DFU mode requires root. Install the bundled rules to allow your user to access it directly.

```bash
# Copy the bundled udev rules
cd ~/STMicroelectronics/STM32Cube/STM32CubeProgrammer/Drivers/rules
sudo cp *.rules /etc/udev/rules.d/

# Reload udev so the rules take effect immediately
sudo udevadm control --reload-rules
sudo udevadm trigger
```

> ℹ️ You may need to **unplug and re-plug** the USB cable after reloading udev rules for the new permissions to apply to the already-connected device.

---

## 7. Flash the board

Before flashing, copy the TSV flash layout file into the images directory so the programmer can resolve relative paths to all partition binaries.

### Copy the flash layout file

```bash
BUILD=~/stm32mp2-yocto/build-openstlinuxweston-stm32mp25-eval
DEPLOY=$BUILD/tmp-glibc/deploy/images/stm32mp25-eval

cp $DEPLOY/flashlayout_st-image-core/optee/\
FlashLayout_emmc_stm32mp257f-ev1-optee.tsv \
$DEPLOY/
```

### Boot the board into DFU mode

Set the boot switches on the STM32MP257F-EV1 to **USB DFU mode** (consult the board schematics or ST's hardware reference), then connect it to your host PC via the USB-C cable.

### Run the programmer

```bash
STM32_Programmer_CLI \
  -c port=usb1 \
  -w $DEPLOY/FlashLayout_emmc_stm32mp257f-ev1-optee.tsv \
  -binpath $DEPLOY
```

> ✅ Flashing typically takes 3–8 minutes. When complete, reset the boot switches to eMMC boot mode and power-cycle the board. Linux should boot within 30 seconds.

> ⚠️ Replace `port=usb1` with `port=usb0` if the programmer reports it cannot find the device. Use `STM32_Programmer_CLI -l usb` to list detected DFU devices.

---

## Adding, removing, and inspecting packages

The Yocto image content is controlled through Bitbake variables, not a package manager like `apt`. Changes are made before the build, and the image is rebuilt.

### Inspect what's in the current image

```bash
# Full package manifest of the built image
cat $DEPLOY/st-image-core-stm32mp25-eval.manifest

# Check what IMAGE_INSTALL currently resolves to
bitbake -e st-image-core | grep ^IMAGE_INSTALL
```

### Add packages via local.conf

The quickest way to add packages is to append to `IMAGE_INSTALL` in your build's `local.conf`:

```bash
# Note the leading space inside the quotes
IMAGE_INSTALL:append = " htop nano python3-numpy openssh"
```

### Remove packages

```bash
# Remove a package even if pulled in as a dependency
IMAGE_INSTALL:remove = " packagename"
PACKAGE_EXCLUDE = "packagename"
```

### Search for available recipe names

```bash
# Search all available recipes by keyword
bitbake-layers show-recipes 2>/dev/null | grep -i <keyword>

# Find recipe .bb files by keyword
find layers/ -name "*.bb" | xargs grep -l "SUMMARY.*<keyword>" 2>/dev/null
```

> ℹ️ After editing `local.conf`, re-source the environment and run `bitbake st-image-core` again. Bitbake will only rebuild what changed — subsequent builds are fast.
