---
title: Developing bootsecure
date: 2026-04-19
description: development of bootsecure and ideas behind it
draft: false
tags:
toc: false
authors:
  - spezialk
---
# Introduction to bootsecure

[Bootsecure](https://github.com/SpezialK-dev/bootsecure) is a small rust application that runs on every system login for the user and detects if the computer has been used without logging in. And notifies the User if that has been the case. 
This uses a feature of the UEFI Spec so it is relatively portable. I found this feature while working on my next blog post about UEFI variables. And I thought that this could make an interesting security application for people worried that their hardware is interacted with while they are away from it. Hence, I build this application.
Currently, It's still quite new, so bugs can happen since it was build in a single weekend, so I am happy if people report bugs with it or contribute to improve this. 

# Design considerations 

I choose Rust simply because I wanted to learn it. For now the application is Linux only and highly dependent on Linux though nothing hinders it from working under Windows[^1] or other OSes as longs as the UEFI interface is there. 

The application is quite small and the parsing of the UEFI var is done without the use of a lib since it was not needed. Also, some libraries don't parse the MTC variable because it's quite a hidden in the Spec[^2]. Besides, It can have different names on different systems. So far I have encountered MTC and Monotonic Counter, but there might be more that I am not aware hence I wanted my own parser to handle this. 
# Why it works

The Monotonic counter consists of two 32 bit the higher one being persistent across reboots while the lower one gets wiped each reboot. There are multiple reasons for the higher one to get counted up, but the main reason is because of a system reset the lower one could also overflow to cause the higher to count up, but I have not encountered that.
This allows one to see the total amount of system resets and thus detects events like visiting the bios or running another EFI binary can cause a reset and trigger the counter to count upwards. 
This allows me to detect interactions with the system, by just storing the old value and seeing if the difference is larger than one. Though this works on the assumption that this is a single user computer and that a user boots the computer up and logs in every time. Both which I would argue are quite reasonable today. Though multiuser can be archived by having the config in a shared location and some modifications to the source something for future revisions possibly.   

Though It might happen on UEFI firmware that the counter increases more than once, though I have not encountered those, but theoretically it's possible since there are other reasons for the Monotonic counter to increase, but I have not seen them and would need more hardware to test that. If that is the case some small modifications would need to happen to the program. 

As always feedback for the application and the posts in general is always appreciated. 



[^1]: macOS might also be possible, but I have no idea if the MTC variable even exists there so I can't test it there. 

[^2]: https://uefi.org/specs/UEFI/2.11/08_Services_Runtime_Services.html#getnexthighmonotoniccount
