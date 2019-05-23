# Fix Huawei Matebook X Pro Speakers on Linux

Out of the box Linux install only 2 out of 4 speakers are working on the Matebook X Pro.

**Here is a step-by-step guide to fix this issue and enabling all 4 speakers to work properly under Linux.**


1. Install needed packages

   `$ sudo pacman -S alsa-utils pulseaudio pulseaudio-alsa alsa-tools pavucontrol pulsemixer`

2. Use `hdajackretask` as root to apply to correct config

   `$ sudo hdajackretask`

3. Select `Realtek ALC256` codec on the top.

4. Check the `Show unconnected pins` and `Advanced overrides` options on right.

5. Apply the following configuration for `PIN ID: 0x14`

   ![2019-05-22-081324_1261x347_scrot](https://user-images.githubusercontent.com/9076747/58158269-9e414a00-7c69-11e9-824b-fc08df81903e.png)
   
6. Apply the following configuration for `PIN ID: 0x1b`

   ![2019-05-22-081708_1284x360_scrot](https://user-images.githubusercontent.com/9076747/58158467-042dd180-7c6a-11e9-9dca-8780d98f6a6a.png)
   
7. Select <kbd>Install boot override</kbd> on the bottom right.

8. (Optional) If you have full disk encryption you need to edit your `/etc/mkiniticpio.conf` with:

   `FILES=(/usr/lib/firmware/hda-jack-retask.fw)`
   
9. (Optional) Then recreate your initramfs:

   `sudo mkinitcpio -P`
   
10. Reboot

11. Open `pavucontrol`, navigate to `Configuration` and under `Profile` dropdown, select `Analog Surround 4.0 Output`.

    Or to achieve the same in terminal : `pactl set-card-profile 0 output:analog-surround-40`

([Source](https://www.reddit.com/r/MatebookXPro/comments/8z4pv7/fix_for_the_2_out_of_4_speakers_issue_on_linux/) & [Source](https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html))
