# Omarchy Mac ISO

Live-boot / installer-environment counterpart to
[omarchy-mac](https://github.com/omarchy-mac/omarchy-mac), mirroring how
[omacom-io/omarchy-iso](https://github.com/omacom-io/omarchy-iso) relates to
upstream Omarchy on x86_64. This repo owns the live boot environment,
installer entrypoint, installation orchestration, and VM/build/test
harnesses. It does not own target-system setup, Apple Silicon/Asahi hardware
configuration, or package adaptations — that's `omarchy-mac`'s job.

## Current milestone: M0

M0 proves one thing: a reproducible build produces an ARM64 artifact that
boots under real AArch64 UEFI to a live userspace and announces a
deterministic ready signal, entirely under QEMU, with no Apple Silicon
hardware involved.

```
source
  -> ./bin/omarchy-mac-iso-make
  -> release/omarchy-mac-iso-arm64/{vmlinuz,initramfs.img}
  -> ./bin/omarchy-mac-iso-boot
  -> AArch64 UEFI (edk2)
  -> Linux kernel + initramfs
  -> live userspace
  -> "OMARCHY_MAC_ISO_READY" on the serial console
```

**M0 does not prove bare-metal Apple Silicon support.** A successful QEMU
boot says nothing about M1/M2 hardware — that validation belongs to a later
Asahi-integration milestone. M0 also does not touch disk partitioning,
bootloaders, target-system installation, or anything Asahi/m1n1/u-boot
specific; see `omarchy-mac` for all of that.

### Why the artifact isn't an ISO

This host has no Docker/Podman/Lima, and upstream's build
(`docker run archlinux/archlinux ... mkarchiso`) needs one. Building a real
Arch Linux ARM archiso image is deferred to whichever milestone actually
needs pacman running inside the live environment. M0's artifact is a plain
kernel + gzip'd cpio initramfs — Alpine aarch64's `linux-virt` kernel is
itself a valid UEFI PE32+ application (EFI stub), so QEMU's
`-kernel`/`-initrd` flags route through the real edk2 UEFI boot manager
(`QemuKernelLoaderFsDxe` + the standard PE/COFF loader), not a bypass. No ESP
filesystem, no bootloader, no disk image needed to prove the UEFI boundary.

### Why the live content is Alpine, not Arch Linux ARM

`omarchy-mac` targets Asahi Alarm (Arch Linux ARM), but M0's live environment
never runs pacman or installs anything — it only needs to reach a live
userspace and print a marker. Alpine's aarch64 minirootfs is the smallest
correct content for that. This is a throwaway generic environment, not a
preview of the eventual installer environment; a later milestone will swap in
a real Arch Linux ARM rootfs once pacman-driven install work actually starts.

## Supported development environment

- macOS (Apple Silicon or Intel) or Linux, x86_64 or aarch64
- [`qemu`](https://www.qemu.org/) on `PATH` (provides both the emulator and,
  on macOS via Homebrew, the AArch64 UEFI firmware)
  - macOS: `brew install qemu`
  - Debian/Ubuntu: `apt install qemu-system-arm qemu-efi-aarch64`
  - Arch: `pacman -S qemu-system-aarch64 edk2-armvirt`
- Everything else (`curl`, `tar`, `cpio`, `gzip`, `shasum`) ships with macOS
  and virtually every Linux distro already.
- No root privileges required. No virtual disks are created — M0 boots
  entirely from RAM.

Hardware acceleration is used when available (HVF on Apple Silicon, KVM on
Linux aarch64 hosts with `/dev/kvm`) and falls back to software emulation
(TCG) otherwise, with a printed warning that boot will be slower.

An Apple Silicon Mac running this is a fast native-aarch64 QEMU *host* for
development. It is not a claim of bare-metal support for that Mac — see
the milestone notes above.

## How to build

```
./bin/omarchy-mac-iso-make
```

Downloads pinned, checksummed upstream artifacts (cached under
`~/.cache/omarchy-mac-iso/`, so repeat builds are fast and offline), and
writes `release/omarchy-mac-iso-arm64/{vmlinuz,initramfs.img,BUILD_INFO}`.

## How to boot

```
./bin/omarchy-mac-iso-boot
```

Prints the exact `qemu-system-aarch64` invocation, then boots interactively
(`Ctrl-A X` to quit). You should see the UEFI firmware banner, kernel boot
log, `==> OMARCHY_MAC_ISO_READY`, and a usable shell prompt.

Pass an explicit artifact directory to boot something other than the default:

```
./bin/omarchy-mac-iso-boot release/omarchy-mac-iso-arm64
```

## How to run the tests

```
./test/unit    # fast, no network, no VM
./test/smoke   # full build + boot + marker detection + teardown
```

`test/smoke` output:

```
==> Building ARM64 image
==> Booting with AArch64 UEFI
==> Waiting for live environment
==> OMARCHY_MAC_ISO_READY
PASS
```

On failure it prints the tail of the serial console log and preserves the
run directory for debugging, and exits nonzero.

## What M0 does NOT prove

- Apple Silicon bare-metal boot (any generation)
- Anything about Asahi's own boot chain (m1n1, u-boot, GRUB) — M0 boots
  under QEMU's generic `virt` machine and edk2 UEFI firmware, a stand-in
  hardware-abstraction boundary, not real Apple Silicon firmware
- Target-disk installation, chroot setup, user creation, or any
  `omarchy-mac` target-side setup — that's owned entirely by `omarchy-mac`
- Package management of any kind (no pacman in this image)
- Disk encryption, production installer UX, or hardware auto-detection
