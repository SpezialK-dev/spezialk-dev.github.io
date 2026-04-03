---
title: Disabling FN+4 under Lenovo
date: 2026-03-16
description: Disabling the  FN+4 Shortcut sleep shortcut
draft: false
tags:
toc: false
authors:
  - spezialk
---

# My Problem 

Lenovo has this feature that allows you to press `FN+4` to put your laptop to sleep. Though this clashes with the switching of the FN and the Ctrl key. Because it seems that there is a bug in the Lenovo Bios of my E14 G6, that when I switched the keys now both keys are recognized as FN only for the sleep shortcut.  All Other FN behaviour works as expected. 

From a quick look around it does not appear like one can simply disable the shortcut in the Bios [^1]. 
It seems that there is an option to remapp it in your OS to simply do nothing[^2].
But most advice is only for Windows and thus not applicable on Linux. 


# Exploring  

Firstly this keyboard shortcut only works when using the laptop keyboard not when using another connected keyboard. So I assume it is might have something to do with ACPI[^3] / is a hardware feature on the laptop keyboard, to send a ACPI command. 

While pocking around `/sys/firmware/acpi/interrupts/` I found `ff_slp_btn`.  Looking at the [documentation](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-firmware-acpi) it turns out that this is the interrupt for a sleep button.

```shell
cat /sys/firmware/acpi/interrupts/ff_slp_btn
       0         invalid      unmasked
```
Looking the output from cat it turns out that the counter does not change. Though the counter of the powerbutton also didn't change so it might be a problem at my end. 

So I looked deeper into some of the managers for ACPI events and stumbled upon [acpid](https://wiki.archlinux.org/title/Acpid) which also included the nice little info to how to find out what event was triggered when a button was pressed, as well as what would be executed on that press. One thing I was still wondering was if lid close would also trigger a sleep button press or if it was handled differently. 

And so I went on with some testing using the following command: 
```shell
journalctl -f
```
When pressing the sleep button I get the following output in the log 
```
Mar 15 19:09:30 archlinux root[46265]: SleepButton pressed
```
While when closing the lid a lid closed event is sent as well as some undefined hotkey.

```
Mar 15 19:16:32 archlinux root[47419]: ACPI group/action undefined: ibm/hotkey / LEN0268:00
[...]
Mar 15 19:16:32 archlinux root[47421]: LID closed
```
Though bot put the laptop into sleep.
This is also backed up by the output of `acpi_listen`. Where the sleep button appears seperatly as `button/sleep SBTN 00000080 00000000 K` and the lid close events are handled differently. 

# Finding a Solution 

Thankfully the [Arch wiki](https://wiki.archlinux.org/title/Acpid#Define_event_action) also includes a way to redefine these events. 

Firstly I tried by adding following script under `/etc/acpi/events/sleep-button`
```shell
event=button/sleep.*
action=<drop>
```
Though this didn't archive the output I wanted, since `FN+4` still put the laptop to sleep. But now it didn't show up in `acpi_listen` anymore.  Looking further into the handling of acpi events I found out that Systemd also brings its own deamon to handle these[^4] [^5]. 


To disable the sleep key simply add the following line to `/etc/systemd/logind.conf` and restart your system. 
```
HandleSuspendKey=ignore
```
Atleast this worked in my case though if your desktop enviroment or some other service handles Sleep you might have to change it there. The reason why changing the acpi config most likely didn't work is because systemd-logind was handling the sleep button and not acpid.


[^1]: https://old.reddit.com/r/thinkpad/comments/ro657m/can_i_disable_the_fn4_hotkey/

[^2]: https://old.reddit.com/r/LenovoLegion/comments/spwja4/disabling_fn_4/

[^3]: https://en.wikipedia.org/wiki/ACPI

[^4]: https://www.dotlinux.net/blog/how-to-handle-acpi-events-on-linux/#handling-acpi-events-with-systemd-logind

[^5]: https://github.com/systemd/systemd/issues/22936
