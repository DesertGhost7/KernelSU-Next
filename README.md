# KernelSU-Next — Samsung 5.10 Fork

A specialized fork of [KernelSU-Next](https://github.com/rifsxd/KernelSU-Next) with Samsung kernel compatibility, userspace soft-reboot, and a custom-signed Manager APK.

> **⚠️ Important:** You **must** install the Manager APK from this repository's [Releases](https://github.com/sarabpal-dev/KernelSU-Next/releases) page. The upstream KernelSU-Next Manager will **not** detect the LKM module because it is signed with a different key. The userspace soft-reboot feature requires matched signatures between the Manager and the kernel module to communicate over ioctl.

---

## Features

### Samsung Kernel Support
- Full compatibility with **Samsung KDP** (Knox Defender Platform), **DEFEX**, and **RKP**
- Kernel-level bypasses for Samsung security hardening on Linux 5.10 kernels
- Dynamic symbol resolution and CFI bypass for restricted kernel environments

### Userspace Soft Reboot
- Instant module reload via `ksud soft-reboot` — no full device reboot required
- Process session isolation (`setsid`) to survive Manager app teardown
- Environment scrubbing (`ZYGISK_ENABLED`) for clean ReZygisk/Zygisk restarts
- Automatic cleanup of lingering module daemons (ptrace monitors)
- DAC permission sanitization for `/data/adb` module directories

### Manager App
- Soft reboot action integrated directly into the Manager UI
- Adapted labels and controls for late-load / temp-root LKM mode
- Certificate signature auto-update in kernel Kbuild on APK rebuild

---

## Installation

### Step 1 — Install the Manager APK

Download and install **KernelSU_Next_v3.3.0-release.apk** from the [Releases](https://github.com/sarabpal-dev/KernelSU-Next/releases) page.

> **Do not use the upstream KernelSU-Next Manager.** This fork uses a custom signing key. The upstream Manager will not recognize the LKM module, and userspace features (soft reboot, ioctl communication) will not work.

### Step 2 — Load the Kernel Module

Download **kernelsu-android12-5.10.ko** from the [Releases](https://github.com/sarabpal-dev/KernelSU-Next/releases) page and load it:

```sh
# Push the module to the device
adb push kernelsu-android12-5.10.ko /data/local/tmp/

# Load via insmod (requires root)
adb shell su -c "insmod /data/local/tmp/kernelsu-android12-5.10.ko"
```

> Standard loading methods or flashing ZIPs designed for upstream KernelSU will **not** work with this fork due to dynamic memory patching at insertion time.

### Step 3 — Open the Manager

Launch the KernelSU-Next Manager app. It should detect the loaded module and show root status.

---

## Building from Source

If you want to build everything yourself with your own signing key, all build scripts are included.

### Prerequisites

- Android NDK with `cargo-ndk` installed
- Rust toolchain with `aarch64-linux-android` target
- Java 17+ (for Manager APK)
- Python 3 (for certificate extraction)
- DDK or kernel build environment (for LKM)

### Build the Manager APK

```sh
# Generate your own signing key (one-time setup)
cd manager && ./setup.sh

# Build everything: ksud binary + Manager APK + certificate hash update
./build_manager.sh --release
```

This will:
1. Cross-compile `ksud` for ARM64
2. Build and sign the Manager APK with your key
3. Automatically extract the APK certificate hash and update `kernel/Kbuild`

### Build the LKM Kernel Module

```sh
cd kernel

# Build for android12-5.10 (default)
./build-all.sh

# Or specify a target explicitly
./build-all.sh android12-5.10
```

> **Important:** After building the Manager APK with a new key, you **must** rebuild the LKM so it contains the updated certificate hash. The module verifies the Manager's signature at runtime.

### Using Your Own Key

1. Run `manager/setup.sh` — it generates `key.jks` and writes credentials to `gradle.properties`
2. Build the Manager APK with `./build_manager.sh --release`
3. The build script automatically runs `scripts/extract_apk_cert.py --update` to patch `kernel/Kbuild` with your certificate hash
4. Rebuild the LKM with `kernel/build-all.sh`

Both the Manager and LKM will now use your key, and signature verification will pass.

---

## Project Structure

```
KernelSU-Next/
├── kernel/                    # LKM kernel module source
│   ├── compat/                # Samsung KDP, DEFEX compatibility
│   ├── build-all.sh           # LKM build script
│   └── Kbuild                 # Contains certificate hash for Manager verification
├── manager/                   # Android Manager app (Kotlin/Compose)
│   ├── setup.sh               # Generate signing key
│   └── app/                   # App source
├── userspace/ksud/            # ksud daemon (Rust)
│   ├── src/soft_reboot.rs     # Userspace soft-reboot implementation
│   ├── src/cleanup.rs         # Module process cleanup
│   └── build.sh               # ksud cross-compilation script
├── scripts/
│   └── extract_apk_cert.py    # Extract APK certificate for Kbuild
└── build_manager.sh           # One-command Manager build + sign + install
```

---

## License

This project is licensed under the [GPL-3.0 License](LICENSE).
