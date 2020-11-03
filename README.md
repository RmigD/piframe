# piframe
## Intro

These are my personal notes, but I'll create an intro to help anyone else.

I'm taking a regular photo frame (with a USB port), adding a Raspberry Pi Zero W and integrate them with Google Photos. The RPi will be interpreted as a USB stick by the digital photo frame. Since the SD card in the RPi has only 4 GB (1.8 GB free after a Raspberry Pi OS lite install), I will need my NAS to keep the photos that are synced with the Google Photos album. I can then add photos to my Google Photos album, and overnight synchronize them with the frame. This could easily be adapted to read photos from a folder in my NAS, but managing these via Google Photos is much more convenient.

I've setup my RPi Zero W according to this page: https://magpi.raspberrypi.org/articles/pi-zero-w-smart-usb-flash-drive - until step 10. I've skipped the Samba setup.

By following the guide above, we can see that I need to create a container (the .bin file) which will contain the images. This .bin file will act like the content of the USB stick that the frame will "see". 32GB of images obviously won't fit within the meager 4 GB SD card, so I'll save the .bin file in the NAS. In order for the RPi to "see" this file as if it was inside the SD card, I will mount the NAS folder as a local folder in the RPi. We will then need to mount the .bin file so that the contents are readable (and we can manage the photos inside via [rclone](www.rclone.org)).

## Turning the RPi Zero W into a "USB stick"

This is done by following the instructions in the magpi url above. We start by burning the system image onto the SD card, and setting it up for network access and SSH access. After this, in short, we do the following:

> Next, we need to enable the USB driver which provides the gadget modes, by editing two configuration files.
>>	sudo nano /boot/config.txt
> Scroll to the bottom and append the line below:
>>	dtoverlay=dwc2
> Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
>>	sudo nano /etc/modules
> Append the line below, just after the i2c-dev line:
>>	dwc2
> Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
> Now shut down the Pi Zero W with the command:
>>	sudo halt
