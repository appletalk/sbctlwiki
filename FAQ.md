# Option ROM
(Work in progress)

Option ROM is firmware that resides on expansion cards on the system which is loaded during boot. These files can contain firmware for graphics cards, storage devices and other PCI cards. UEFI includes these files as part of the Secure Boot chain and any failure to validate this ROM file is going to prevent loading the given hardware.

These files are usually signed by the `Microsoft Corporation UEFI CA 2011`.

The image below displays the process in a simplified fashion.


![](https://edk2-docs.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-M5spcXCRu82SfMZIemU%2F-M5sphCby0TUlHRNiNkZ%2F-M5spoo4UIOYinNkvqZQ%2Fimage2.png?generation=1587944071800213&alt=media)
(From: https://edk2-docs.gitbook.io/understanding-the-uefi-secure-boot-chain/secure_boot_chain_in_uefi/uefi_secure_boot)

The effect of this, depending on the hardware, is essentially "soft bricking" the device. If you don't have any iGPU but your nvidia card has Option ROM that fails to validate, you might not have any way to display graphics. This would prevent you from turning off secure boot.

One can figure out if these files are part of your bootchain by utilizing the Trusted Platform Module (TPM). It provides an eventlog which should record all of the Option ROM loaded during boot. These are recorded as `EFI_BOOT_SERVICE_DRIVER`.

```
$ cp /sys/kernel/security/tpm0/binary_bios_measurements eventlog
$ tpm2_eventlog eventlog | grep "BOOT_SERVICES_DRIVER"
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
  EventType: EV_EFI_BOOT_SERVICES_DRIVER
```

If there is Option ROM in your bootchain there are two ways one can solve this:
* Enroll the `Microsoft Corporation UEFI CA 2011` file.
* Read the checksums from the TPM eventlog.

These entries from the TPM Evetlog also has a checksum that corresponds to the firmware, and the UEFI signature database (the `db` variable) allows x509 certificates and checksum, one can take the checksums from the log and enroll them along with any self-created Platform keys. However this should be considered experimental at best.

The safe option is to just enroll the Microsoft CA, despite the [valid criticisms](https://github.com/pbatard/rufus/wiki/FAQ#Why_do_I_need_to_disable_Secure_Boot_to_use_UEFINTFS
) against the hold Microsoft has over the Secure Boot platform keys on modern computers.

`sbctl` will strive to guide the user and implement any workarounds to the Microsoft CA where possible.
