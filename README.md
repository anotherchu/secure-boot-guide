# Secure Boot with rEFInd using MOK on ArchLinux

When attempting to setup Secure Boot on my desktop machine, I ran into a Verification failed:(0x1A)Security Violation screen when trying to
load rEFInd even though I was sure the proper keys were already enrolled in MOK Manager. The cause seems to be from a lacking .sbat section for rEFInd for shim>=15.3.

This article will attempt to workaround this by installing an earlier version of shim.

## Pre-requisites

* [A compatible shimx64.efi binary for rEFInd](http://launchpadlibrarian.net/502909051/shim-signed_1.45+15+1552672080.a4a1fbe-0ubuntu2_amd64.deb)
* [sbsigntools](https://archlinux.org/packages/?name=sbsigntools)

## Steps

Extract the shimx64.efi binary from the shim-signed package from Ubuntu launchpad:

```console
mkdir shim-signed && cd shim-signed && \
curl http://launchpadlibrarian.net/502909051/shim-signed_1.45+15+1552672080.a4a1fbe-0ubuntu2_amd64.deb --output shim-signed_1.45.deb && \
ar -xv shim-signed_1.45.deb && tar -xvf data.tar.xz && mv usr/lib/shim/shimx64.efi.dualsigned shimx64.efi
```

Use the refind-install script to install the new shim binary and sign the drivers with local keys:

```console
refind-install --usedefault /dev/sdXY --shim shimx64.efi --localkeys
```

where sdX is the device and Y is the partition number for the ESP

Sign the kernel with the same keys:

```console
sbsign --key /etc/refind.d/keys/refind_local.key --cert /etc/refind.d/keys/refind_local.crt --output /boot/vmlinuz-linux-zen /boot/vmlinuz-linux-zen
```

You should replace `vmlinuz-linux-zen` with your kernel image.

Then, enroll the key in MOK Manager the next time you boot. It can be found in *ESP*/EFI/BOOT/refind/keys/refind_local.cer

## Automatically sign the kernel when it updates

You may have to create these folders if they don't exist:

```console
mkdir /etc/pacman.d/hooks
mkdir -p /usr/local/share/libalpm/scripts
```

Copy these files:

```console
cp /usr/share/libalpm/hooks/90-mkinitcpio-install.hook /etc/pacman.d/hooks/90-mkinitcpio-install.hook
cp /usr/share/libalpm/scripts/mkinitcpio-install /usr/local/share/libalpm/scripts/mkinitcpio-install
```

In `/etc/pacman.d/hooks/90-mkinitcpio-install.hook`, replace:

`Exec = /usr/share/libalpm/scripts/mkinitcpio-install`

with

`Exec = /usr/local/share/libalpm/scripts/mkinitcpio-install`


In `/usr/local/share/libalpm/scripts/mkinitcpio-install`, replace:

`install -Dm644 "{line}" "/boot/vmlinuz-${pkgbase}`

with

`sbsign --key /etc/refind.d/keys/refind_local.key --cert /etc/refind.d/keys/refind_local.crt --output "/boot/vmlinuz-{pkgbase}" "${line}"`

## References

1. [Unified_Extensible_Firmware_Interface/Secure_Boot#Signing_the_kernel_with_a_pacman_hook](Unified_Extensible_Firmware_Interface/Secure_Boot#Signing_the_kernel_with_a_pacman_hook)
2. [A solution to rEFInd unable to load using shim when Secure Boot is enabled](https://dev.to/hollowman6/a-solution-to-refind-unable-to-load-using-shim-when-secure-boot-is-enabled-1e8l)
3. [REFInd#Secure_Boot](https://wiki.archlinux.org/title/REFInd#Secure_Boot)

