The modern BitLocker implementation uses TPM2 to store the disk decryption key. This requires Secure Boot to be enabled for Windows to safely unseal the Bitlocker decryption key from the TPM. WIthout secure boot, you will be prompted for the long Bitlocker recovery key at every Windows boot.

Setting this up with `sbctl` is fairly straightforward. The steps follow from [this blog post](https://saligrama.io/blog/post/upgrading-personal-security-evil-maid/#enrolling-your-key-into-secure-boot).

1. Reboot into your UEFI interface and enable secure boot. Set the secure boot mode setting to “Setup mode,” which allows enrolling new keys. Then boot back into Arch.

```bash
# Execute the following instructions as root

# 2. Install sbctl
pacman -S sbctl

# 3. Create a keypair
sbctl create-keys

# 4. Enroll your keys while keeping Microsoft's keys.
sbctl enroll-keys --microsoft

# 5. Sign each of the EFI files that may appear somewhere
#    in the boot chain. The following files are specific
#    to my configuration, double check that you sign everything
#    you need to for your setup.
#    It is possible that double-signing the Microsoft entries
#    could be rejected by some firmwares, but this has not been
#    an issue for me. See the following issue:
#    https://github.com/Foxboron/sbctl/issues/137
sbctl sign -s /boot/EFI/Linux/linux.efi
sbctl sign -s /boot/EFI/Linux/fallback.efi
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/Boot/bootx64.efi
sbctl sign -s /boot/EFI/Microsoft/bootmgfw.efi
sbctl sign -s /boot/EFI/Microsoft/bootmgr.efi
sbctl sign -s /boot/EFI/Microsoft/memtest.efi

# 6. Verify that all the files you need are signed
sbctl list-files

# 7. Verify that the sbctl pacman hook works on a kernel upgrade.
#    Ensure that the string "Signing EFI binaries..." appears.
pacman -S linux
```

8. Reboot into the UEFI interface and ensure that Secure Boot is still enabled. Verify that the Secure Boot mode setting has changed to “User mode.”

9. Test booting into Arch, Arch fallback, and Windows. All should succeed without issues.

10. Boot into Windows and enable Bitlocker. Rebooting into Windows should work and you should not be prompted to enter a Bitlocker recovery key.% 