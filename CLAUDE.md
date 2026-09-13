# NixOS Consume

The intent of this project is to convert a running foreign Linux into a minimal NixOS in place. It is
intended to be run against a freshly installed new minimal foreign Linux system.

## Claude directives
* DON'T git commit or push without user approval

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
