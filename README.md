> [!IMPORTANT]
> This fork pairs with **[ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)** (manager package `com.resukisu.resukisu`) instead of KernelSU. The bundled `ksud` is a shim that defers to ReSukiSU's `libksud.so`.

# DFRoot [DirtyFrag (CVE-2026-43284)]

Fork to root my emerald aka Poco M6 Pro (non-Samsung). Not tested on other devices.

The core of this code is credited to others. This fork combines those pieces and adds a few small
improvements/features.

Credits:
- Original PoC and various code: https://github.com/lsposed/lspromise
- Selinux Permissive kernel modules and various code: https://github.com/polygraphene/DFReroot
- Unprivileged XFRM socket method: https://github.com/combeng6th/DirtyInit

## Features

- Start on Boot
- Automatic soft reboot 
- RO Partition Protection
- Hide Selinux Modifications in KSU
- No Shizuku dependency — root can be regained without WiFi

> [!WARNING]
> I am not responsible for any damage to your device.

## Supported Devices

Ephemeral root for Samsung devices (and possibly others) w/ locked bootloaders vulnerable to DirtyFrag (CVE-2026-43284) 

**Verified on a non-Samsung device:** Poco M6 Pro (emerald, MT6789), HyperOS, kernel `6.12.30-android16` — DEFEX kprobes silently no-op on non-Samsung kernels; everything else is generic GKI.

| KMI Version | Verified |
|---|---|
| android12-5.10 | Yes ( Nothing Phone 2 ) |
| android13-5.10 | Untested |
| android13-5.15 | Untested |
| android14-5.15 | Untested |
| android14-6.1 | Untested |
| android15-6.6 | Yes (Samsung) |
| android16-6.12 | Yes (Samsung + Poco M6 Pro / Re:SU) |
| android17-6.18 | Untested |

## How it works

The Android kernel decrypts AES-CBC ESP packets directly into the page cache of files open for `splice()`. By crafting `IV = AES_ECB_DEC(key, current_content) ⊕ desired_content`, any 16-byte-aligned block in a mapped shared library can be overwritten without write permission and without copy-on-write.

The exploit uses this primitive to patch shellcode into `libc++.so` in the kernel's page cache. The next privileged call to those functions runs the shellcode and loads the DirtyFrag kernel module, which brings up ReSukiSU.

### Exploit chain

1. **IpSec transform** — App allocates a `UdpEncapsulationSocket` + SPI and builds an AES-CBC/HMAC-SHA256 ESP transform via `IpSecManager`.

2. **splicehelper → crash_dump64** — helper binary spliced into `/apex/com.android.runtime/bin/crash_dump64` via the CBC primitive. `crash_dump64` can be called by unprivileged app with `type_transform` and gives read access to vendor library pages and splices them into a pipe so the parent can compute correct IVs. 

3. **dirtyfrag.ko → libbinderdebug.so** — The kernel module is written into `/vendor/lib64/libbinderdebug.so` with `vendor_file` label that can be modprobe'd

4. **libc++ hook** (runs in init, uid=0, tid=1) — entrypoint via createorphanprocess. Patched with shellcode that forks, sets the child's SELinux exec context to `u:r:vendor_modprobe:s0`, and directly execs `/vendor/bin/insmod` with the patched vendor lib as the module (vendor_modprobe is allowed to `finit_module`).

5. **dirtyfrag.ko init** — resolves `kallsyms_lookup_name` via the kprobe trick, sets `selinux_state` to permissive, and registers DEFEX-bypass kprobes (Samsung-only; silently skipped on non-Samsung kernels like emerald). It then `call_usermodehelper`s `/system/bin/sh -c "<staged ksud> late-load ... "` and self-unloads by returning `-E2BIG` from init.

6. **ksud shim → ReSukiSU** — the staged `ksud` is a shell shim (see `app/src/main/assets/ksud`): it locates the ReSukiSU manager's `libksud.so` under `/data/app` and execs `ksud late-load --kmi <androidXX-Y.Y>`, which loads the ReSukiSU-built `kernelsu.ko` (LKM) and starts the daemon. Touches `/dev/dfm0` (success) or `/dev/dfm1` (failure) from the real `late-load` exit code.

7. **Cleanup** — libc++ patches are restored and crash_dump64 is fadvised out of cache.

## Usage

Install the **[ReSukiSU manager](https://github.com/ReSukiSU/ReSukiSU/releases)** (`com.resukisu.resukisu`) — required. The shim uses its `libksud.so` + `kernelsu.ko`.

```sh
./build.sh
adb install -r dirtyfrag.apk
```

Open the app → **Root Device** → open the ReSukiSU manager to see root. Reboot clears `/dev/df` and re-enables the button (markers live on tmpfs `/dev`).
