# nixos-consume

If your here then your like me and can't stand the thought of having to imperatively build out your
VPS's configuration in a foreign Linux. You would much rather prefer to have a provably correct
custom configuration, fully tuned to your needs, that you can apply with a single command and
rollback capablities if needed i.e. NixOS. If this is you then this script is your jam. The intent
is to have a oneliner curl bash approach to convert your existing freshly deployed minimal Ubuntu
based VPS system over to a clean minimal NixOS on which you can then apply your own custom NixOS
configuration.

### Disclaimer
This is a completely destructive approach to installing NixOS on an existing Ubuntu Linux. Use at
your own risk; knowing that ***nixos-consume*** will entirely consume your existing Ubuntu Linux
system leaving it potentially unusable if you don't have console or otherwise access to recover if
things go wrong. Luckily you can typically use VPS provider tools to simply wipe it and start over,
but there is inherit risk in overwriting a system in place.

This project borrows heavily from the awesome [nixos-infect](https://github.com/elitak/nixos-infect).
I wanted something that supports the new kexec NixOS unstable way of doing things. Oh and this is
purely flake based as well.

### Quick links
- [Getting started](#getting-started)
  - [Convert Ubuntu 24.04 server](#convert-ubuntu-24.04-server)
  - [Environment variables](#environment-variables)
  - [Tested Linux Distros and VPS providers](#tested-linux-distros-and-vps-providers)
- [Details](#details)
  - [For those that like to know](#for-those-that-like-to-know)
  - [Prerequisites](#prerequisites)
    - [Disk space](#disk-space)
    - [RAM](#ram)
    - [SSH Keys](#ssh-keys)
- [Development](#development)
  - [Running unpublished changes](#running-unpublished-changes)
  - [Testing with a local quickemu VM](#testing-with-a-local-quickemu-vm)

## Getting started
Ensure your host is configured with at least ***2GB of RAM*** and a ***20GB or larger disk*** see
[prerequisites](#prerequisites).

### Convert Ubuntu 24.04 server

1. Provision your host using Ubuntu Server 24.04

2. Ensure your SSH authorized key is in `/root/.ssh/authorized_keys`
   ```bash
   scp ~/.ssh/authorized_keys root@<host>:/tmp
   ssh root@<host>
   install -m 600 -o root -g root /tmp/authorized_keys /root/.ssh/authorized_keys
   ```

3. Configure nameserver resolution if your server didn't come with it provisioned
   1. Edit the netplan `vim /etc/netplan/50-cloud-init.yaml` and add the `nameservers` section
      ```yaml
      network:
        version: 2
        ethernets:
          eth0:
            addresses:
              - 15.14.13.12/24
            routes:
              - to: default
                via: 15.14.13.1
            nameservers:
              addresses:
                - 1.1.1.1
                - 8.8.8.8
      ```
   2. Apply the netplan changes and check the dns resolution
      ```bash
      netplan apply 
      dig raw.githubusercontent.com
      ```

4. Run the script the script as `root`
   ```bash
   sudo su
   curl https://raw.githubusercontent.com/phR0ze/nixos-consume/master/consume | NIXPKGS=nixos-unstable bash

   # Example output
   consume - convert this host to NixOS in place
    - Host:      racknerd-1234567 (Ubuntu 24.04 LTS, Linux 6.8.0-31-generic)
    - CPU:       Intel(R) Xeon(R) CPU E5-2680 v2 @ 2.80GHz, 2 core(s)
    - RAM:       2.1G
    - Disk:      /dev/vda2 (34G, ext4)
    - Boot:      BIOS (legacy), GRUB device /dev/vda
    - Network:   eth0 (Static)
    - Swap:      existing device /dev/vda3 carried over (zramSwap disabled on target); temporary 1G swapfile for the build (removed afterward)
    - SSH keys:  1 authorized key(s) from /root/.ssh/authorized_keys carried over
    - NixOS:     nixos-unstable (system.stateVersion 25.11)
    - Options:   STATIC_IP=y FALLBACK_SWAP=y MEM_TUNING=y KEXEC=y SERIAL_CONSOLE=n

   This will DESTROY the above disk and replace it with NixOS. Continue? [y/N]
   ```
   press `y`

5. The script will report `warning: installing Nix as root is not supported by this script!`. This is
   normal and to be expected. Your SSH session to the original OS will drop the moment `kexec` runs
   (it kills the whole process tree). Reconnect with the same key after a few seconds — you'll land
   in the installer. Once logged back in you can watch progress through `journalctl -u consume-install -f`.
   Once that completes you'll loose your connection again as it reboots into the final system.

6. Install your age key if needed, note `root` is the only that exists in the new system via the
   previously configured public key. This also means there will be new SSH fingerprint to accept.
   ```bash
   scp ~/.config/sops/age/keys.txt root@<host>:/tmp
   ssh root@<host>
   mkdir -p ~/.config/sops/age/
   install -m 600 -o root -g root /tmp/keys.txt /root/.config/sops/age/keys.txt
   ```
7. Deploy your configuration. here's my personal example
   1. Create the nix shell that has your deps
      ```bash
      nix-shell -p git sops
      ```
   2. Clone my target configuration
      ```bash
      cd /etc/nixos
      rm *
      rm .*
      git clone https://github.com/phR0ze/nixos-config
      ```
   3. Apply your config typically `nixos-rebuild switch --flake "${CONFIG_DIR}#${HOST}"`
      ```bash
      # my own automation
      ./clu update system vps1
      ```

### Environment variables

| Variable                | Default       | Description                                        |
| ----------------------- | ------------- | -------------------------------------------------- |
| `UNATTENDED=y`          | unset         | Skip confirmation prompt                           |
| `NIXPKGS`               | `25.11`       | Nixpkgs flake ref; also sets stateVersion          |
| `NIXOS_FLAKE=<url>`     | unset         | Custom flake.nix, fetched instead of generated     |
| `NIXOS_CONFIG=<url>`    | unset         | Custom configuration.nix (fetched, not generated)  |
| `STATIC_IP=y`           | auto          | Force static network config (else auto-detected)   |
| `FALLBACK_SWAP=n`       | `y`           | Skip the target's fallback disk swapfile           |
| `MEM_TUNING=n`          | `y`           | Skip sysctl/oomd memory tuning on target           |
| `KEXEC=n`               | `y`           | Stop before kexec; leaves configs for inspection   |
| `NIX_INSTALL_URL=<url>` | nixos.org URL | Override Nix installer URL                         |
| `SERIAL_CONSOLE=y`      | unset         | Add serial console kernel params                   |

### Tested Linux Distros and VPS providers
Feel free to open a PR if you've managed to get this working on other linux hosting combinations.

| Distro            | Flavor          | Version   | Hosting             | Firmware |
| ----------------- | --------------- | --------- | ------------------- | -------- |
| Ubuntu            | Server          | 24.04     | RackNerd            | BIOS     |
| Ubuntu            | Server          | 24.04     | KVM/QEMU            | EFI      |
| Ubuntu            | Server          | 24.04     | KVM/QEMU            | BIOS     |

## Details

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
Plan **at least 2GB of RAM** — even that is tight. The kexec installer uses `netboot-minimal.nix`
(network install, no offline channel copy) instead of the larger `netboot-base.nix` which failed
outright on a 2GB test VM (`EINVAL` / kernel page fault). Typically target VPS systems will always
have limited RAM due to the cost which is why the defaults are to set following:
```nix
# Default NixOS recommend value, set as the highest swap priority
zramSwap = { enable = true; memoryPercent = 50; priority = 100; algorithm = "zstd"; };

# Cheap fallback insurance to use at a lower priority only after zramSwap is full
swapDevices = [{ device = "/var/swapfile"; size = 2048; priority = 5; }];
```

#### SSH Keys
Ensure that your freshly provisioned host has the root account properly configured to be able to SSH
into the system using `/root/.ssh/authorized_keys`. The resulting NixOS system will have no accounts
other than root and no password. SSH'ing in using your ssh key will be the only mechanism to get back
into the system.

## Development

### Running unpublished changes
If your testing against a not-yet-pushed copy of the script, copy it over and run it
directly instead of pulling from GitHub:
```bash
scp consume <user>@<host>:/tmp/consume
ssh <user>@<host>
sudo NIXPKGS=nixos-unstable bash -x /tmp/consume
```

### Testing with a local quickemu VM
Given the disclaimer above, don't iterate against a real host - test against a disposable
local VM instead (see [quickemu](https://github.com/phR0ze/tech-docs/tree/master/src/virtualization/virtual_machines/quickemu)
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
