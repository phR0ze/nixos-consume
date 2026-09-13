# nixos-consume

If your here then your like me and can't stand the thought of having to imperatively build out your
VPS's configuration in a foreign Linux. You would much rather prefer to have a provably correct
custom configuration, fully tuned to your needs, with rollback capablities i.e. NixOS. Then this script,
is your jam. The intent is to have a oneliner curl bash approach to convert your existing freshly
deployed minimal Ubuntu based VPS system over to a clean minimal NixOS on which you can then apply
your own custom NixOS configuration.

### Disclaimer
This is a completely destructive approach to installing NixOS on an existing foreign Linux. Use at
your own risk; knowing that ***nixos-consume*** will entirely consume your existing foreign Linux
system leaving it potentially unusable if you don't have console or otherwise access to recover if
things go wrong. Luckily you can typically use VPS provider tools to simply wipe it and start over,
but there is inherit risk in overwriting a system in place.

This project borrows heavily from the awesome [nixos-infect](https://github.com/elitak/nixos-infect).
I wanted something that supports the new kexec NixOS unstable way of doing things.

### Quick links
- [Overview](#overview)
  - [For those that like to know](#for-those-that-like-to-know)
  - [Prerequisites](#prerequisites)
    - [Disk space](#disk-space)
    - [RAM](#ram)
    - [SSH Keys](#ssh-keys)
  - [Tested Linux Distros and VPS providers](#tested-linux-distros-and-vps-providers)
- [Getting started](#getting-started)
- [Development](#development)
  - [Running unpublished changes](#running-unpublished-changes)
  - [Testing with a local quickemu VM](#testing-with-a-local-quickemu-vm)
- [Environment variables](#environment-variables)

## Overview

### For those that like to know
***nixos-consume*** will interogate the currently running Linux to determine key system details such
as: disk layout, hostname, SSH keys and networking among other things. Using these details consume
will then build a ephemeral NixOS installer it will then `kexec` directly into — no reboot necessary.
From the ephemeral installer it will then format the original root partition write out the target
configuration and run the `nixos-install` and reboot when finished.

### Prerequisites

#### Disk space
Plan on **at least a 20GB disk**. The ephemeral kexec installer has to be built into `/nix/store` on
the *original* filesystem, alongside the foreign Linux , before the `kexec` jump ever happens. In
testing, a 10GB disk ran out of space mid-build with `nix build` failing outright. If you hit `error:
... note: build failure may have been caused by lack of free disk space`, this is why.

#### RAM
Plan **at least 2GB of RAM**, and be aware that even 2GB is tight. The ephemeral kexec
installer is built from `netboot-minimal.nix` rather than `netboot-base.nix` specifically
because of this: `netboot-base.nix` bundles a full offline nixpkgs channel copy, pushing the
kexec payload past 1GB — large enough that both `kexec_load` and `kexec_file_load` failed
outright on a 2GB test VM (the former returned `EINVAL`, the latter crashed the kernel with a
page fault). `netboot-minimal.nix` keeps the payload small enough to load reliably, at the
cost of needing working network access immediately after the `kexec` jump (no offline channel
copy).

#### SSH Keys
Ensure that your freshly provisioned host has the root account properly configured to be able to SSH
into the system using `/root/.ssh/authorized_keys`. The resulting NixOS system will have no accounts
other than root and no password. SSH'ing in using your ssh key will be the only mechanism to get back
into the system.

### Tested Linux Distros and VPS providers
Feel free to open a PR if you've managed to get this working on other linux hosting combinations.

| Distro      | Flavor  | Version | Hosting       | Firmware |
| ----------- | ------- | ------- | ------------- | -------- |
| Ubuntu      | Server  | 24.04   | KVM/QEMU      | EFI      |
| Ubuntu      | Server  | 24.04   | KVM/QEMU      | BIOS     |

## Getting started

1. Ensure your host is configured with at least ***2GB of RAM*** and a ***20GB or larger disk***
2. Provision your host using Ubuntu Server 24.04
3. Ensure your SSH authorized key is in `/root/.ssh/authorized_keys` 
4. Run the script, either straight from GitHub:
   ```bash
   curl https://raw.githubusercontent.com/phR0ze/nixos-consume/master/consume | NIXPKGS=nixos-25.11 bash -x
   ```
5. Your SSH session to the original OS will drop the moment `kexec` runs (it kills the whole
   process tree). Reconnect with the same key after a few seconds — you'll land in the

## Development

### Running unpublished changes
If your testing against a not-yet-pushed copy of the script, copy it over and run it
directly instead of pulling from GitHub:
```bash
scp consume <user>@<host>:/tmp/consume
ssh <user>@<host>
sudo NIXPKGS=nixos-unstable bash -x /tmp/consume
```

### 
*NB*: This script **formats the target host's root partition** as part of the unattended
install. Any errors during phase 1 (before the `kexec` jump) halt execution safely — the
original OS is untouched at that point. Once `kexec` runs, there's no going back; run with
`bash -x` so you have a full log if something goes wrong.

### Testing with a local quickemu VM
Given the disclaimer above, don't iterate against a real host - test against a disposable
local VM instead (see [quickemu](../tech-docs/src/virtualization/virtual_machines/quickemu)
if you need one set up). The workflow that's been used to develop and test this script:

1. Provision a fresh Ubuntu Server 24.04 VM (at least 20G disk, 2G RAM - see
   [Prerequisites](#prerequisites)) and get it to a normal logged-in state (a non-root user
   with a password, `openssh-server` installed). For a BIOS-firmware VM, Ubuntu's
   [autoinstall](https://ubuntu.com/server/docs/install/autoinstall) can build this
   unattended by booting the ISO's `casper/vmlinuz`+`casper/initrd` directly with
   `-append "autoinstall ds=nocloud; ..."` and a small NoCloud seed ISO, bypassing the
   interactive installer entirely.
2. Take a `qemu-img snapshot -c <tag>` of that baseline *before* ever running `consume`
   against it, so you can revert and retest repeatedly without re-provisioning.
3. Before each test run: revert to the baseline snapshot
   (`qemu-img snapshot -a <tag> disk.qcow2`), boot the VM, and seed
   `/root/.ssh/authorized_keys` (a manual prerequisite `consume` itself checks for - see
   [SSH Keys](#ssh-keys)):
   ```bash
   ssh <user>@<host> "sudo mkdir -p /root/.ssh && sudo tee -a /root/.ssh/authorized_keys" < ~/.ssh/id_ed25519.pub
   ```
4. Copy over and run the script per [Running unpublished changes](#running-unpublished-changes)
   above, adding `UNATTENDED=y` to skip the confirmation prompt for a hands-off run.
5. Watch for the SSH session dropping (the `kexec` jump), reconnect, and watch
   `journalctl -u consume-install -f` in the ephemeral installer until it reboots.
6. Verify the result: `systemctl --failed` (expect none), `systemctl is-system-running`
   (expect `running`), and that a full `reboot` survives cleanly.
7. Revert to the step-2 snapshot again before the next test run - `consume` is destructive
   and there's no in-place undo once `kexec` has run.

This same procedure works for exercising the BIOS/GRUB code path specifically (a separate
baseline VM built with `boot="legacy"` instead of the default EFI firmware) and the
`STATIC_IP=y` path (force it with the env var even on a DHCP VM, and check the rendered
`networking.nix` looks sane - this path doesn't currently have a dedicated test VM).

5. Your SSH session to the original OS will drop the moment `kexec` runs (it kills the whole
   process tree). Reconnect with the same key after a few seconds — you'll land in the
   ephemeral installer's own sshd. Watch progress with:
   ```
   journalctl -u consume-install -f
   ```
   If the unattended install fails, that unit shows `failed` in `systemctl status` and the
   ephemeral installer's sshd stays up (it does **not** auto-reboot on failure) so you can
   debug and re-run `nixos-install --root /mnt ...` by hand.
6. On success, the install unit reboots the host straight into the finished NixOS system.

## Environment variables

* `UNATTENDED=y` — skip the confirmation prompt after the system summary is printed.
  * Default: unset (prompts for confirmation before proceeding)
* `NIXPKGS` — the `nixpkgs` flake ref to build against (e.g. `nixos-25.11`, `nixos-unstable`).
  Also used to derive `system.stateVersion`: when `NIXPKGS` matches `nixos-XX.YY`, `stateVersion` is
  set to `XX.YY`.
  * Default: `25.11`
* `NIXOS_FLAKE=<url>` — supply your own `flake.nix`, fetched in place of the generated one.
  * Default: unset (generate one)
* `NIXOS_CONFIG=<url>` — supply your own `configuration.nix`
  * Default: unset (generate one)
* `STATIC_IP=y` — generate static network config (`networking.nix`) from the currently
  active network settings, applied to both the ephemeral installer and the final target.
  This is auto-detected and not needed typically but allows for overriding default behavior.
  * Default: unset (auto-detected)
* `NO_SWAP=y` — skip creating a temporary swapfile before building the kexec tree.
  * Default: unset (a temporary swapfile is created, unless swap is already active on the host)
* `NO_KEXEC=y` — stop right before the `kexec` jump; generated configs are left under
  `/etc/consume/target` and `/etc/consume/kexec` for inspection, and you can
  trigger the install manually via `/etc/consume/kexec/result/kexec-boot`.
  * Default: unset (proceeds through the `kexec` jump)
* `NIX_INSTALL_URL=<url>` — override the Nix installer URL used to bootstrap Nix on the
  original OS.
  * Default: `https://nixos.org/nix/install`
* `SERIAL_CONSOLE=y` — add `console=tty1 console=ttyS0,115200` to `boot.kernelParams` on both
  the ephemeral installer and the final target system. Useful as a last-resort debugging aid
  when a display or SSH-based console shows nothing (e.g. a hung/broken boot) — you can then
  read the hypervisor's serial socket directly instead (e.g. `virsh console <vm>`, or
  `socat - unix-connect:<vm>-serial.socket` for a quickemu VM).
  * Default: unset (no serial console output)
