---
title: Understanding Uefi Vars
date: 2026-03-07
description: A deep dive into UEFI variables and their meanings.
draft: false
tags:
toc: false
authors:
  - spezialk
---
# What are UEFI Vars

UEFI vars are variables that are stored on your motherboard. They store things like boot options, SHIM settings, and other information that have to be independent of your hard drive.

Under the Linux system they are mapped under `/sys/firmware/efi/efivars/` using the EFI vars file system [^2].

Under Windows there are some special permissions needed for an app to read them out[^4]. Though system calls exist to ask for specific variables [^5]. 

## My motivation for looking at them

I find them interesting because they reveal a bunch of information about a system that is normally only accessible during startup.  
Also, they are not something that most people interact with in their daily lives, even though they play a crucial role in how their system works. Besides, quite a few of them are not really documented. And I haven't been able to find a single source that aggregates information about a lot of them. This is a first step to try and aggregate as much information as possible about them and consolidate it in a single place. With either writing what I found or directly linking to their documentation.  

## What their Format looks like under Linux

The variable format is as follows: its `Name + GUID. 
With the GUID being the same across systems, as far as I can tell.

The internal format of each variable is as follows:
`4 bytes of attributes + efivar data` [^3].


# What Information is stored in EFI vars 

While I tried to give as much info as I could find this is not a complete list and only shows the ones that are shown in user space via the Linux file system for them or where they are mentioned in documentation. 
Hence, I wanted to write an article giving some insights into some of them, what they do, and how they impact your system.

The best way to read them out from the Linux file system is via a hex-editor like xxd.
A Path where one can find them is given in the command below.
```
xxd /sys/firmware/efi/efivars/<efivar in fs>
```
No special permissions should be needed to read them out. 
Altering them is not recommended unless you know what you want to archive and know their format. 

Alternatively, tooling like [Efivar](https://man.archlinux.org/man/efivar.1.en) exists and can be used.  That gives additional information about each variable. 

A fully sorted list can be found in the [Addendum](#Addendum). 
For the ones I that I found noteworthy or where only very little documentation exists, I expanded with what I could find, as well as some speculation based on surrounding evidence. 

## UEFI Spec

A list of all from the UEFI spec specified codes can be found [here](https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html#globally-defined-variables).
Most of them are fairly clear already based on the above-mentioned document. 

### devAuthBoot

Is a interesting variable because its not exposed to the OS(at least to my experience), but also serves a security critical role in the boot process. It toggles if devices have to be verified in the boot process[^8]. 
As far as I understand the spec, all external devices, all PCI devices, as well as all attached devices on specific ports are verified by this.  The document states it mostly verifies the identity and if it is a known device, not if the certificate is the newest one or if it has the latest firmware.

But as I understand the spec, theoretically one could also add their own revocations to the revocation list and thus block booting if certain devices are installed. Meaning one could block known bad devices from being inserted before boot (If their certificate is knownen). 

#### EDKII implementation of the Feature 

The code to check if verification is activated is located in `SecurityPkg/DeviceSecurity/SpdmSecurityLib/SpdmSecurityLib.c`[^9].

The actual validation happens in the following code.
```c
  if (((SecurityPolicy->AuthenticationPolicy & EDKII_DEVICE_AUTHENTICATION_REQUIRED) != 0) && (IsDeviceAuthBootEnabled ())) {
    Status = DoDeviceAuthentication (SpdmDeviceContext, &AuthState, SlotId, IsValidCertChain, RootCertMatch, SecurityState);
    if (EFI_ERROR (Status)) {
      DEBUG ((DEBUG_ERROR, "DoDeviceAuthentication failed - %r\n", Status));
      goto Ret;
    } else if ((AuthState == TCG_DEVICE_SECURITY_EVENT_DATA_DEVICE_AUTH_STATE_FAIL_NO_SIG) ||
               (AuthState == TCG_DEVICE_SECURITY_EVENT_DATA_DEVICE_AUTH_STATE_FAIL_INVALID))
    {
      goto Ret;
    }
  }
```
[Source](https://github.com/tianocore/edk2/blob/2bfde627ff858f1ca1a3a7809d8cbe968e21b6cc/SecurityPkg/DeviceSecurity/SpdmSecurityLib/SpdmSecurityLib.c#L108)
****
The `IsDeviceAuthBootEnabled()` function checks for the `devAuthBoot` variable and returns a Boolean based on that. 
All of this checking is housed in `SpdmDeviceAuthenticationAndMeasurement`.
So this is all done via the [SPDM protocol](https://www.dmtf.org/standards/spdm) as is alluded by the function name.


## SHIM

The Shim brings its own list of interesting variables..

As taken from their [documentation](https://github.com/rhboot/shim/blob/main/MokVars.txt).
Generally speaking all of the RT variables are ones that are copies of the value to make them available to the kernel.

### MokSBState / MokSBStateRT

These two variables are responsible for storing the information about whether the SHIM is in insecure mode or not. Insecure means that to the EFI, secure boot is still active, so the UEFI variable is still set, but SHIM does not validate signatures.

If the variable is set to 1 the insecure mode is activated.

With the Mokutils checking the RT variable, the other state variable is not exposed to the operating system.[^1]. 
```c
if (efi_get_variable (efi_guid_shim, "MokSBStateRT", &data, &data_size,
			      &attributes) >= 0) {
		moksbstate = 1;
		free (data);
	}
```
[Source](https://github.com/lcp/mokutil/blob/9ae73db13e2bf0c610b0dcced5410fd6dc61f68a/src/mokutil.c#L1451)

This variable is set when one runs `mokutil --disable-validation` and then disables the validation in Shim Screen. The flow to do that is described in this [article](https://wiki.ubuntu.com/UEFI/SecureBoot/DKMS). 

The flow works as follows:
firstly `MokSB` is set with the following packet [^6] :
```c
typedef struct {
        UINT32 MokSBState;
        UINT32 PWLen;
        CHAR16 Password[PASSWORD_MAX];
} __attribute__ ((packed)) MokSBvar;
```

This then gives you the new option to disable or enable Secure Boot (depending on your current settings). The logic to disable or enable all that is handled by `mok_sb_prompt`[^7]
This then sets the variable `MokSBStateRT` and `MokSBState`.

Interestingly enough, whatever password one has set with `MokPW` seems to have no impact on this setting, so there is no way to prevent an operating system from setting this setting (as long as a user has the needed user rights in that operating system). This lines up with my personal experience using this feature.

In my playing around with this, I also found out that the `MokSBStateRT` variable disappears from the file system once set to 0.

## Systemdboot specific ones

It also stores what it should boot, as well as some info if a system fails to boot.  
I won't repeat their [documentation,](https://systemd.io/BOOT_LOADER_INTERFACE/) but if anyone is interested, it can be easily found.  
There is nothing overly unexpected for a bootloader to find.  
Though settings like `LoaderTpm2ActivePcrBanks` are interesting and could be interesting to look at if measured boot becomes more commonplace in Linux environments.

## Lenovo Specific ones 

The following ones are most likely Lenovo specific UEFI variables. 

Though there are some more UEFI Vars, I will only attribute those to Lenovo where I am certain that they are not from my System. 

###  BootCurrent  and BootOrder

This seems to be the current selection of BootOrder and what is currently booted. 
I am quite sure they are Lenovo specific, since they are not mentioned in the EDKII source code, nor are the mentioned in the UEFI spec (at least to my research).

Though some more playing around might solve this in the future.

The format of `BootCurrent` is quite simple it stores the hex number of the current `Boot####` entries elected. 

And `BootOrder` simply has a list of Hex values separated by zeros

### LastBootOrder and LastBootCurrent 

These appear similar to the `BootCurrent` and `BootOrder` but probably only save some previous value. Though their format is the same. 


### PlatformLang and PlatformLangCodes

These denote some language codes / what language is selected. 
Though these are quite interesting. Because the laptop this was run on has a German keyboard and is clearly intended for the German market. 
But simply looking at the language codes this is not really identifiable. 

```shell
$ xxd PlatformLang-8be4df61-93ca-11d2-aa0d-00e098032b8c
00000000: 0700 0000 656e 2d55 5300                 ....en-US.
```

```shell
xxd PlatformLangCodes-8be4df61-93ca-11d2-aa0d-00e098032b8c
00000000: 0600 0000 656e 2d55 533b 6a61 2d4a 503b  ....en-US;ja-JP;
00000010: 6672 2d46 523b 6b6f 2d4b 5200            fr-FR;ko-KR.
```

I maybe understand the JP code, because all the certificates that are stored in the `db` that are from Lenovo have their location either set to North Carolina or Kanagawa Yokohama.  Which does not explain the fr-FR one. 


## AMD specific ones

For all the other AMD specific ones I could not really make sense of what they would store
### AMD_PBS_SETUP and AmdSetup

This seems to be related to AMD PBS which as far as I could tell from other motherboard manuals seems to be some sort of menu for AMD specific settings [^10].
This might store configuration information or some other info potentially. Since it's quite a huge blob.  `AmdSetup` also might hold some more configuration information. Though once again I could not find anything concrete. Hence, this is all speculation. 

## Windows / Microsoft specific ones

Since Windows used to be installed on this laptop, there are some windows specific ones.

### BuiltAsSecuredCorePC

This is probably a way for windows to find out if the current PC is a [Secured-core PC](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-highly-secure-11)  or not.  I don't know if the pure existence means its a secured Core PC or if there has to be some value in this variable. 

And currently I don't really have a way to verify it either.


## Tgc2

This has to do with TPM-2 physical presence[^11].  As far as I understand it this seems to be some [standart](https://trustedcomputinggroup.org/wp-content/uploads/PC-Client-Platform-Physical-Presence-Interface-Specification-V1.4-R15_22July22.pdf) by the Trusted computing group. 

### Tcg2PhysicalPresence and Tcg2PhysicalPresenceFlags

Are visible from user land with `Tcg2PhysicalPresence` serving as a way to communicated between OS and firmware, the format is described in the standard. The Flags variable simply indicates the set operation flags. 

Information on both can be found in section 5.1 and 5.2 respectfully in the [standard](https://trustedcomputinggroup.org/wp-content/uploads/PC-Client-Platform-Physical-Presence-Interface-Specification-V1.4-R15_22July22.pdf).

### Tcg2PhysicalPresenceConfig

Is not visible from user land but is mentioned in the standard under section 5.3. Hence, I wanted to include it 
### adjacent Flags

These two flags are not mentioned in the standard, but their naming suggest that they are adjacent to the entire TPM2 setup. Though I have no clue how they work they are most likely a Lenovo thing, but I don't know.
# Final words

I think quite a lot of this is still up for exploration, and there are even some UEFI variables that I did not list here because I could not find out what they are from or what meaning they could have. 
If someone has more resources / knows more I am more than happy to learn more about them because I think this is very interesting. 

# Addendum
there are certainly some missing, but I did not want to add somewhere I was not sure from what system they were. 

## Addendum-UEFI 
(the #### are replaced with hexadecimal numbers)

- AuditMode
- Boot####
- BootCurrent
- BootNext
- BootOrder
- BootOptionSupport
- ConIn
- ConInDev
- ConOut
- ConOutDev
- dbDefault
- dbrDefault
- dbtDefault
- dbxDefault
- DeployedMode
- devAuthBoot
- devdbDefault
- Driver####
- DriverOrder
- ErrOut
- ErrOutDev
- HwErrRecSupport
- KEK
- KEKDefault
- Key####
- OsIndications
- OsIndicationsSupported
- OsRecoveryOrder
- PK
- PKDefault
- PlatformLangCodes
- PlatformLang
- PlatformRecovery####
- SignatureSupport
- SecureBoot
- SetupMode
- SysPrep####
- SysPrepOrder
- Timeout
- VendorKeys

## Addendum-SHIM 

- MokPW
- MokSB
- MokDB
- MokDBState
- MokNew
- MokAuth
- ShimRetainProtocol
- MokList
- MokListRT
- MokListX
- MokListXRT
- MokSBState
- MokSBStateRT
- MokDBState
- MokIgnoreDB
- MokPWStore
- MokListTrusted
- HSIStatus

## Addendum- Systemdboot
- LoaderConfigTimeout
- LoaderDevicePartUUID
- LoaderEntries
- LoaderEntrySelected
- LoaderFeatures
- LoaderFirmwareInfo
- LoaderFirmwareType
- LoaderImageIdentifier
- LoaderInfo
- LoaderSystemToken
- LoaderTimeExecUSec
- LoaderTimeInitUSec
- LoaderTimeMenuUSec
- LoaderTpm2ActivePcrBanks

## Addendum-Lenovo

- LastBootOrder
- LastBootCurrent
- LenovoAbtStatus
- LenovoBDG
- LenovoConfig
- LenovoEvtLogCapsuleUpdate
- LenovoFunctionConfig
- LenovoHiddenSetting
- LenovoLogging
- LenovoMfgProductID
- LenovoRuntimeConfig
- LenovoScratchData
- LenovoSecurityConfig
- LenovoSystemConfig
- LenovoWolInfo
- BootCurrent
- BootOptionSupport
- BootOrder
- BootOrderDefault
- PlatformLang
- PlatformLangCodes
- PlatformSetupCommon
- ProtectedBootOptions

## Addendum-AMD


- AMD_PBS_SETUP
- AmdSetup
- AmdAcpiVar

## Addendum-Windows 

- BuiltAsSecuredCorePC

## Addendum-Tcg2

- Tcg2PhysicalPresence
- Tcg2PhysicalPresenceFlags
- Tcg2PhysicalPresenceConfig
### Addendum-Tgc2-adjacent

- PhysicalPresence
- PhysicalPresenceFlags


[^1]: https://github.com/lcp/mokutil/blob/9ae73db13e2bf0c610b0dcced5410fd6dc61f68a/src/mokutil.c#L1451

[^2]: https://wiki.gentoo.org/wiki/Efivarfs/en

[^3]: https://docs.kernel.org/6.1/filesystems/efivarfs.html

[^4]: https://learn.microsoft.com/en-us/windows/win32/sysinfo/access-uefi-firmware-variables-from-a-universal-windows-app

[^5]: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getfirmwareenvironmentvariablea

[^6]: https://github.com/rhboot/shim/blob/main/MokVars.txt

[^7]: https://github.com/rhboot/shim/blob/c4665d282072df2ed8ab6ae1d5fa0de41e5db02f/MokManager.c#L1455

[^8]: https://uefi.org/specs/UEFI/2.11/32_Secure_Boot_and_Driver_Signing.html#device-authentication

[^9]: https://github.com/tianocore/edk2/blob/1fe2504afb6ed8a991d5cfe2e3cf1d39afb47b95/SecurityPkg/DeviceSecurity/SpdmSecurityLib/SpdmSecurityLib.c#L21

[^10]: https://download.gigabyte.com/FileList/Manual/mb_manual_amd800-bios_e_0822.pdf

[^11]: https://github.com/tianocore/edk2/blob/2bfde627ff858f1ca1a3a7809d8cbe968e21b6cc/SecurityPkg/Include/Guid/Tcg2PhysicalPresenceData.h
	
