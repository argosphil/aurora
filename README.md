# The Google Pixel Watch 2 (WiFi only)

* Codenamed `aurora` (`eos` is the version with an LTE modem ... or maybe the other way around?).
* Tapping Settings > About > Build number to activate developer mode doesn't work
* however, after pairing with an Android phone (which needs to use the Google app store), activating developer mode and enabling OEM unlock does work

# Reboot into fastboot

Hold both buttons for 20 seconds. Yes, that long. Then hit the top-left and bottom-right edges of the touch screen simultaneously. You have a window of about half a second to do so. Retry until that works out. It's as finicky as it was on the `r11`. Actually, you can now hit the side button instead of doing the top-left/bottom-right tap! That's convenient.

In fact, an easy way to get to fastboot appears to be to simply hold the side button for a very, very long time (until the "fastboot" message appears, or longer). This always appears to work.

# Bootloader issues

The bootloader accepts v4 abootimgs, which contain an uncompressed kernel image (compressed kernel images do not work). If they contain a ramdisk, it is not passed to the kernel properly (I suspect that /init is overwritten by one of the other ramdisks provided in the other boot partitions). The images do not contain dtbs, which are instead provided by the bootloader.

We can work around the ramdisk problem, kind of, by including the ramdisk in the kernel image (it will still be compressed). However, that gives rise to a chicken-and-egg problem, because we would like to include kernel modules (built after the kernel) in the ramdisk already included in the kernel.

One way around that is to use something like barebox as an intermediate boot loader, to decompress a kernel image and an initrd and boot into it. That also gives us the opportunity to apply DT overlays, and it appears to be working!

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

Any ideas on how to trace this down to a kernel config option? Presumably I activated something that resulted in the inclusion of about 20 MB of data. It's not the DWARF/BTF debug information.

Tracked it down. It was KFENCE + KASAN + UBSAN.

# USB problems

The USB connection sometimes goes wonky after booting into debug-ramdisk.

# RAM Oops

I haven't gotten USB ramoops to work, but storage ramoops does work:

* `fastboot oem ramdump storage`
* reboot after a failure
* `mount -t pstore pstore /sys/fs/pstore`
* look at `/sys/fs/pstore/console-ramoops-0`

# Frame buffer

It's turning out to be quite hard to get the display driver to work at all. Getting it to load is simple, but it turns out it is then in "continuous splash" mode (IIUC, all this means is the Google logo is still being displayed out of a fixed frame buffer set up by the bootloader), and the code paths to leave that mode don't seem to be working.

So I thought we could just pretend continuous splash mode is a thing that doesn't exist, but then we fail to set up the IOMMU to account for the fixed frame buffer, and the SMMU faults cause us to throw errors.

One thing we can do is map the initial fixed frame buffer as a simplefb. That works, kind of, except for caching issues since the simplefb is in RAM but accessed non-coherently by the display engine.

Right now what I'm doing is to forget about continuous splash mode as soon as the IOMMU's set up to include a mapping for the fixed frame buffer.

However, the screen's flickering, kind of like it's double-buffered but one of the buffer's all black.

Also, psplash doesn't work because it uses mmap.

(Actually, psplash does work on the simplefb, as do three of the expected four Tux logos).

# Minor things

* `adb reboot bootloader` works ... sometimes

# Android boot process

Just follow this diagram(https://source.android.com/docs/core/architecture/partitions/generic-boot).

Or don't. I'm not even sure which of the five applies to us.

In summary, init is now a multi-stage process, it's not clear what first-stage init does to make second-stage init not fail to boot and reboot the system instead.

# APEX

APEX is some nightmare idea of loopback-mounting essential system components from APK files. I have no idea how it starts up, and this link(https://source.android.com/docs/core/ota/apex) isn't much help. The problem is that there's an `apexd` that presumably mounts apex files, but it requires `libc.so`, which is... in an APEX file.
