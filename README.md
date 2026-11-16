# Arch Linux Installation Lab

Group coursework documenting a manual Arch Linux installation in a virtual machine.
Read the [installation notes](arch_installation.md), written primarily in Swedish.

Topics covered include UEFI detection, disk partitioning, LUKS encryption, ext4,
package installation, users and sudo, systemd networking and DNS, swap, encrypted
secondary storage, crypttab/fstab, and a GNOME/Wayland desktop.

## Work in progress

These are the group's original working notes, not a complete, tested installation
guide. The notes retain placeholders and review comments. GRUB/initramfs setup for
booting encrypted root is not documented, and the scheduled Restic backup section
is unfinished. There is no claim that backup automation or a reproducible installation
has been completed. No VM or installation commands were executed during publication.

The disk formatting commands erase the selected device. These lab notes assume an
isolated disposable VM; device names must be checked for that VM before use.

The recorded documentation commit is by Johan Sjöberg. This repository represents
group coursework participation and credits the source in [ATTRIBUTION.md](ATTRIBUTION.md).
