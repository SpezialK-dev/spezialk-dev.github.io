---
title: UEFI Vars
date: 2026-04-05
description: A complete List of UEFI Vars
draft: false
tags:
toc: false
authors:
  - spezialk
exclude_from_rss: true
---


# Variables 

A comprehensive List of UEFI variables 
asd
## UEFI 
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
- MTC (Monotonic Counter)

## SHIM 

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

## Systemdboot
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

## Lenovo

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

## AMD


- AMD_PBS_SETUP
- AmdSetup
- AmdAcpiVar

## Windows 

- BuiltAsSecuredCorePC

## Tcg2

- Tcg2PhysicalPresence
- Tcg2PhysicalPresenceFlags
- Tcg2PhysicalPresenceConfig
### Tgc2-adjacent

- PhysicalPresence
- PhysicalPresenceFlags