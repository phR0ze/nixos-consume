# NixOS Consume

The intent of this project is to convert a running foreign Linux into a minimal NixOS in place. It is
intended to be run against a freshly installed new minimal foreign Linux system.

## Claude directives
* DON'T git commit or push without user approval
* DON'T include user personal information into the repo

## Project Goals
* Use modern nixos-unstable methods for installing NixOS as a replacement for an existing non-NixOS Linux.
* Usage should be driven by a single curl bash script
* Support carrying over and using the existing ssh authorized_keys
* Script should detect all hardware details and convert it into the appropriate nixos equivalents
* Script should detect and configure all appropriate kernel settings and use in nixos equivalents
* Allow the user to verify the gather details and approach look sound before continuing
* Final results of the script run should be a fully working clean minimal standard NixOS system
* No cruft from the prior foreign Linux should remain
* Resulting system should be flake based and usable as normal from `/etc/nixos`

## Test VMs
Local quickemu VMs for testing live outside this repo at `~/Projects/vms/`:
* `ubuntu-server1` (EFI) — with a single snapshot, ssh port 2222
* `ubuntu-server2` (BIOS) — with a single snapshot, ssh port 2223
Both are Ubuntu Server 24.04, user `admin`/password `foobar`. Revert to the snapshot, boot,
seed `/root/.ssh/authorized_keys`, then copy over and run `consume` (see README's
"Testing with a local quickemu VM"). Never test against a real host.

## Non-obvious gotcha
The generated `kexec-boot` script (from `config.system.build.kexecTree`) does NOT bundle its
own `kexec` binary — the host's own `kexec` (apt package `kexec-tools`) must be on PATH, or
the jump fails with "kexec not found: please install kexec-tools". This is easy to get wrong
by reading nixpkgs source, since the *separate* `boot.kexec`/`kexec.nix` systemd module does
wrap `path = [ pkgs.kexec-tools ]` — that's a different code path and doesn't apply here.


