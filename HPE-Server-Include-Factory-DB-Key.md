HPE servers require their DB key to be added to boot with Secure Boot enabled. Unlike other devices the Microsoft keys should not be required.
On a HPE DL380 Gen9 this key is *"HP UEFI Secure Boot 2013 DB key"*. It may be different on other hardware.

The following steps will allow you to install your own keys and then append the factory db and dbx entries from HPE. Then finally you can remove all db entries other than your key and the required HPE key.

1. Reboot and enter the BIOS.  Go to *Server Security > Secure Boot Settings > Reset all keys to platform defaults*.  This ensures you have only the factory keys present. Then reboot back into Arch.

2. Run the following commands as root to extract the existing EFI variables. Only the db and dbx will be needed but you can save the PK and KEK as well.  You will need the `efitools` package installed.

```bash
efi-readvar -v PK -o factory_PK.esl
efi-readvar -v KEK -o factory_KEK.esl
efi-readvar -v db -o factory_db.esl
efi-readvar -v dbx -o factory_dbx.esl
```

3. Reboot back to the BIOS and delete all the keys by going to *Server Security > Secure Boot Settings > Delete all keys (PK, KEK, DB, DBX)*. Then reboot back into Arch.

```bash
# Run the following commands as root

# 4. Install sbctl
pacman -S sbctl

# 5. Create Secure Boot keys
sbctl create-keys

# 6. Enroll the keys.  
# If you do not have a TPM in the server you will need to use 
# --yes-this-might-brick-my-machine
sbctl enroll-keys

# 7. Temporarily remove the Platform Key so you return to Setup mode.
cd /usr/share/secureboot
efi-updatevar -d 0 -k keys/PK/PK.key PK

# 8. Append the factory db and dbx you extracted earlier.
# Note the -a flag which appends to the EFI variable.
# All dbx entries appear to be signed by Microsoft and since we
# are removing the Microsoft db key this is optional.
# But there is no harm in including them.
efi-updatevar -a -e -f factory_db.esl db
efi-updatevar -a -e -f factory_dbx.esl dbx

# 9. Generate a PK.auth so we can reinstall our Platform Key
cert-to-efi-sig-list -g "$(< GUID)" keys/PK/PK.pem PK.esl
sign-efi-sig-list -g "$(< GUID)" -k keys/PK/PK.key -c keys/PK/PK.pem PK PK.esl PK.auth

# 10. Move the generated files into the PK keys folder and restrict permissions
chmod 400 PK.esl PK.auth
mv PK.esl PK.auth keys/PK/

# 11. Finally reinstall the platform key. 
# Run the chattr command if you get a permission denied error
chattr -i /sys/firmware/efi/efivars/{PK,KEK,db}*
efi-updatevar -f keys/PK/PK.auth PK
```

12. If everything was successful running `sbctl status` should show setup mode as disabled. This indicates the platform key is present.

13. Sign your kernel and any required files. Check `sbctl verify` to confirm.
```bash
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/vmlinuz-linux
```

14. Once everything is signed, reboot and enable Secure Boot in the BIOS. *Server Security > Secure Boot Settings > Secure Boot Enforcement > Enabled*.

15. Confirm you are able to boot the system and run `sbctl status` which should show Secure Boot as enabled but with the Microsoft vendor keys.

16. Now we will want to remove any db keys that we don't explicitly require to boot the system. Run `efi-readvar` and look at the output under the db section. You should see your db key listed first and additonal factory keys below. Make note of these entries and the owner guid for each. In the case of the DL380 Gen9 we will be removing both Microsoft keys and the SUSE Linux key listed below. We need to retain the HPE key with the guid of `f5a96b31-dba0-4faa-a42a-7a0c9832768e` to load required option roms. If it is removed Secure Boot will fail to boot.

```
db: List 1, type X509
    Signature 0, size 1420, owner f5a96b31-dba0-4faa-a42a-7a0c9832768e
        Subject:
            O=Hewlett-Packard Company, OU=Long Lived CodeSigning Certificate, CN=HP UEFI Secure Boot 2013 DB key
        Issuer:
            C=US, O=Hewlett-Packard Company, CN=Hewlett-Packard Printing Device Infrastructure CA
db: List 2, type X509
    Signature 0, size 1572, owner 77fa9abd-0359-4d32-bd60-28f4e78f784b
        Subject:
            C=US, ST=Washington, L=Redmond, O=Microsoft Corporation, CN=Microsoft Corporation UEFI CA 2011
        Issuer:
            C=US, ST=Washington, L=Redmond, O=Microsoft Corporation, CN=Microsoft Corporation Third Party Marketplace Root
db: List 3, type X509
    Signature 0, size 1515, owner 77fa9abd-0359-4d32-bd60-28f4e78f784b
        Subject:
            C=US, ST=Washington, L=Redmond, O=Microsoft Corporation, CN=Microsoft Windows Production PCA 2011
        Issuer:
            C=US, ST=Washington, L=Redmond, O=Microsoft Corporation, CN=Microsoft Root Certificate Authority 2010
db: List 4, type X509
    Signature 0, size 1296, owner 2879c886-57ee-45cc-b126-f92f24f906b9
        Subject:
            CN=SUSE Linux Enterprise Secure Boot Signkey, C=DE, L=Nuremberg, O=SUSE Linux Products GmbH, OU=Build Team, emailAddress=build@suse.de
        Issuer:
            CN=SUSE Linux Enterprise Secure Boot CA, C=DE, L=Nuremberg, O=SUSE Linux Products GmbH, OU=Build Team, emailAddress=build@suse.de
```

18. Reboot to the BIOS one last time and go to *Server Security > Secure Boot Settings > Advanced Secure Boot Options > Delete Signature (Allowed DB)*. Then remove the signatures not required listed below. Only your db key and the HPE key should be remaining. Compare against the output you took from `efi-readvar` above to confirm.
```
77fa9abd-0359-4d32-bd60-28f4e78f784b
77fa9abd-0359-4d32-bd60-28f4e78f784b
2879c886-57ee-45cc-b126-f92f24f906b9
```

19. That's it. Reboot the system and let it start up normally. Run `sbctl status` and confirm you have Secure boot enabled but the vendor keys entry should be gone.
