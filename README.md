# Omarchy Mac ISO

Live-boot / installer-environment counterpart to
[omarchy-mac](https://github.com/omarchy-mac/omarchy-mac), mirroring how
[omacom-io/omarchy-iso](https://github.com/omacom-io/omarchy-iso) relates to
upstream Omarchy on x86_64. This repo owns the live boot environment,
installer entrypoint, installation orchestration, and VM/build/test
harnesses. It does not own target-system setup, Apple Silicon/Asahi hardware
configuration, or package adaptations — that's `omarchy-mac`'s job.

## M0

M0 proves that a reproducible ARM64 build crosses the AArch64 UEFI boundary
under QEMU, reaches Linux userspace, and emits a deterministic readiness
signal.

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

M0 does not prove bare-metal Apple Silicon support, and does not touch disk
partitioning, bootloaders, target-system installation, or anything
Asahi/m1n1/u-boot specific — see `omarchy-mac` for all of that.

## Requirements

- macOS (Apple Silicon or Intel) or Linux, x86_64 or aarch64
- [`qemu`](https://www.qemu.org/) on `PATH` — on macOS via Homebrew this also
  provides the AArch64 UEFI firmware
  - macOS: `brew install qemu`
  - Debian/Ubuntu: `apt install qemu-system-arm qemu-efi-aarch64`
  - Arch: `pacman -S qemu-system-aarch64 edk2-armvirt`
- `curl`, `tar`, `cpio`, `gzip`, `shasum` — ship with macOS and virtually
  every Linux distro already
- No root privileges and no virtual disks: M0 boots entirely from RAM

Hardware acceleration is used when available (HVF on Apple Silicon, KVM on
Linux aarch64 hosts with `/dev/kvm`) and falls back to software emulation
(TCG) otherwise.

## Build

```
./bin/omarchy-mac-iso-make
```

Downloads pinned, checksummed upstream artifacts (cached under
`~/.cache/omarchy-mac-iso/`), and writes
`release/omarchy-mac-iso-arm64/{vmlinuz,initramfs.img,BUILD_INFO}`.

## Boot

```
./bin/omarchy-mac-iso-boot
```

Boots interactively (`Ctrl-A X` to quit). Expect the UEFI firmware banner,
kernel boot log, `==> OMARCHY_MAC_ISO_READY`, and a shell prompt.

## Test

```
./test/unit    # fast, no network, no VM
./test/smoke   # full build + boot + marker detection + teardown
```

On failure, `test/smoke` prints the tail of the serial console log and
preserves the run directory for debugging.

## Architecture / Scope

- M0's artifact is a kernel + initramfs, not a production ISO — Alpine's
  `linux-virt` kernel is itself a UEFI EFI-stub PE binary, so QEMU boots it
  directly without an ESP, bootloader, or disk image.
- Alpine aarch64 is temporary M0 live content, not the eventual installer
  distro decision; a later milestone swaps in Arch Linux ARM once
  pacman-driven install work starts.
- QEMU's generic `virt` machine + edk2 firmware is a stand-in AArch64 UEFI
  boundary, not Apple Silicon emulation.
- `omarchy-mac-iso` owns boot/install orchestration; `omarchy-mac` owns
  target-system Apple Silicon / Omarchy setup.

## Non-goals

- Package management of any kind — no pacman in this image
- Disk encryption, production installer UX, or hardware auto-detection
- chroot setup or user creation — that's `omarchy-mac`'s job
