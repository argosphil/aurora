# The Google Pixel Watch 2 (WiFi only)

* Codenamed `aurora` (`eos` is the version with an LTE modem ... or maybe the other way around?).
* Tapping Settings > About > Build number to activate developer mode doesn't work
* however, after pairing with an Android phone (which needs to use the Google app store), activating developer mode and enabling OEM unlock does work

# Reboot into fastboot

Hold both buttons for 20 seconds. Yes, that long. Then hit the top-left and bottom-right edges of the touch screen simultaneously. You have a window of about half a second to do so. Retry until that works out. It's as finicky as it was on the `r11`. Actually, you can now hit the side button instead of doing the top-left/bottom-right tap! That's convenient.

# Kernel config

Google doesn't provide what appears to be a working kernel configuration, so I had to extract one using the firmware update ZIP, mkbootimg, and extract-ikconfig. That still needed some work because (of course) it depends on Google-specific files not provided anywhere, mounted in random locations, but ultimately built.

# booting

The first few attempts at booting the kernel weren't successful, so I switched to a built-in ramdisk and enabled the USB serial console gadget. That prevented the immediate reboot, at least, so I guess it's doing something?

* I disabled PSCI restarts, which cannot provide ramdumps
* I ran "fastboot oem ramdump usb" to try to enable ramdumps

# adb prompt

I now get an adb prompt using a heavily-modified kernel (mostly adding as built-ins most of the 92 (!) kernel modules Android uses).

# kernel size

For some reason, kernels I built myself are very large, so there are problems exceeding the 64 MB size limit for the kernel + ramdisk.
