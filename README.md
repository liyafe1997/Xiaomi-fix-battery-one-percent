# English

## Background

Some Xiaomi devices with PM8150 (qcom gen4) power management IC, mainly Snapdragon 865/870 devices including Redmi K30Pro/POCO F2 Pro, K30S Ultra, K40, Xiaomi 10. (Xiaomi 10 Pro is not included because it uses BQ27Z561 PMIC), there is a chance the battery level may stuck at 1%, even charging(actually it can be charged, but still displaying 1%) or reboot doesn't recover it. 

If you run into this situation, the only way to escape is trying to "reset" the PMIC, one way is just to let the battery drain, so the motherboard and PMIC would be completely powered off. Another way is by pressing the Volume Down & Power button for 60s+, power cycle into FASTBOOT around 10+ times, then the PMIC could be reset.

## What caused this

The qualcomm's PMIC driver `qpnp-fg-gen4.c`, it has a strategy called "rapid_soc_dec", when the battery is under-voltage(usually caused by an aging battery or/and low temperature with high current and low battery level), it will immediately report the battery level (aka. `soc` in the code) to 0% for asking the upper-level software to power off the system. (Actually that is bad, this is why some other phones sometimes suddenly power off when the battery is low or in a low-temperature environment). **There is no chance of exiting this state because the original design logic is powering off the device! This is the root reason for this "bug"!!**

However, Xiaomi Implemented a "smooth strategy" in the driver, that can let the battery level change "smoothly" without suddenly jumping to a very high or low value. So when it has entered "rapid_soc_dec" state, it will slowly and smoothly move to 1% (not reporting 0% because there are some other conditions that need to be met), then always stuck there, not powering off, and no chance of exiting this "rapid_soc_dec" state. Charging the battery is also not able to exit this state.

Even, rebooting/power cycling also doesn't work, because for entering "rapid_soc_dec" state, the parameters have been written to the PMIC register, so it reports 0% `soc` from the hardware, not the kernel/software thing. Even there is a "fg_gen4_shutdown" callback that seems to try to restore the value, but I don't know why it doesn't work on many devices, at least not my device. Some people say it can be solved by rebooting/power cycling the device, which means this callback working on these devices.

## The fix
Quite simple, just add an exit strategy of "rapid_soc_dec" state. Checking the battery voltage when it > 3700mV, then exit this state. 

Just check this commit, it is the code of the fix: https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25

Why did I choose the value 3700mV? I don't know exactly, It just seems for a battery that is at a relatively low charge (Like 20+%), still easy to go back to >3700mV when in a low load(current). If you connect to a charger surely it should go back to >3700.

This fix just makes sure 1. The battery/kernel accidentally entered that state, in a "enough charge level" like 20+ present. 2. At least it can escape this state when you charge the battery. If your battery voltage is really low and can not go back to 3700, it is better to just stay in this "rapid_soc_dec" state, reports 1% to you for asking you to charge the battery as soon as possible :)

# About this repository
This repository is forked from Redmi K30 Pro / POCO F2 Pro's official kernel (https://github.com/MiCode/Xiaomi_Kernel_OpenSource/tree/lmi-q-oss), and I just committed this commit for showing how does the fix works: https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25.

If you are building a third-party kernel you can refer to this commit to fix this problem. Highly recommend you do that for your kernel to avoid this annoying 1% battery bug for the end-users.

# 简体中文
写上面的累了，有空再写吧，或者哪位老哥帮写下，可以提交个PR，万分感谢！