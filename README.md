# piframe
## Intro

These are my personal notes, but I'll create an intro to help anyone else.

I'm taking a regular photo frame (with a USB port), adding a Raspberry Pi Zero W and integrate them with Google Photos. The RPi will be interpreted as a USB stick by the digital photo frame. Since the SD card in the RPi has only 4 GB (1.8 GB free after a Raspberry Pi OS lite install), I will need my NAS to keep the photos that are synced with the Google Photos album. I can then add photos to my Google Photos album, and overnight synchronize them with the frame. 
This could easily be adapted to read photos from a folder in my NAS, but managing these via Google Photos is much more convenient. 
 
I've setup my RPi Zero W according to this page: https://magpi.raspberrypi.org/articles/pi-zero-w-smart-usb-flash-drive - until step 10. I've skipped the Samba setup. 
 
By following the guide above, we can see that I need to create a container (the .bin file) which will contain the images. This .bin file will act like the content of the USB stick that the frame will "see". 32GB of images obviously won't fit within the meager 4 GB SD card, so I'll save the .bin file in the NAS. In order for the RPi to "see" this file as if it was inside the SD card, I will mount the NAS folder as a local folder in the RPi. I will then mount the .bin file as well, so that the contents are readable (and we can manage the photos inside it via [rclone](www.rclone.org)). 

## Setting it up from scratch
### Turning the RPi Zero W into a "USB stick"

> sudo raspi-config 
Activate "wait for network" at boot 

The following is done by following part of the instructions in the magpi url above. We start by burning the system image onto the SD card, and setting it up for network access and SSH access. After this, in short, we do the following: 

> Next, we need to enable the USB driver which provides the gadget modes, by editing two configuration files. 
>
>>	sudo nano /boot/config.txt 
>
> Scroll to the bottom and append the line below:
>
>>	dtoverlay=dwc2
>
> Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
>
>> sudo nano /etc/modules 
>
> Append the line below, just after the i2c-dev line:
>> dwc2
>
> Press CTRL+O followed by Enter to save, and then CTRL+X to quit. 
> Now shut down the Pi Zero W with the command: 
>> sudo halt 

### Setting up the image synchronization 
Part of these instructions are included in the magpi url above.
### Create folder structure and mount content
Create folder /mnt/fotosnas (will contain the .bin file) in the RPi 
>
> mkdir /mnt/fotosnas 
>
Create folder /mnt/fotos (will show the contents of the .bin file): 
>
> mkdir /mnt/fotos 

The NAS access credentials are stored inside the file /home/pi/.nascreds. Let's mount the NAS folder where we will keep the .bin file. It will show in the RPi as folder /mnt/fotosnas: 
> sudo mount -t cifs -o credentials=/home/pi/.nascreds,vers=2.0,uid=pi //NAS/others/piframe /mnt/fotosnas 
>
Create the .bin file inside /mnt/fotosnas (this will create the file in the NAS. It will take a while, depending on the size. Start small to test. This one will have almost 30 GB). 
>
> sudo dd bs=1M if=/dev/zero of=/mnt/fotosnas/piusb.bin count=30000 
>
Format the .bin file: 
> sudo mkdosfs /mnt/fotosnas/piusb.bin -F 32 -I 
>
Setup fstab to mount the .bin file in folder /mnt/fotos by adding the following to file /mnt/fstab: 
>
> /mnt/fotosnas/piusb.bin /mnt/fotos vfat users,umask=000,nofail 0 2 
>
Mount all content as specified inside fstab: 
>
> sudo mount -a 

#### Synchronize the desired Google Photos album with directory /mnt/fotos (rclone usage)
This will get the images from the Google Photos album called "piframe0" into the folder /mnt/fotos (which is actually the .bin file). 

Setting up rclone is beyond this documentation, but it's really easy. See the bottom of this file for more info. After installing it (or unzipping the files), run: 
> ~/rclone-v1.53.1-linux-arm/rclone config
...and follow the instructions.
##### Notes: 
1. Installing rclone via apt-get installed an older version of rclone, without the option to sync Google Photos. I just downloaded the latest 32 bit ARM version from rclone and unzipped it into the home folder. 
2. Google won't let rclone touch the current albums in Google Photos via the current version of its API. This is why you need to create an album with rclone itself. See next step. 

Create the album in Google Photos, via rclone: 
>
> ~/rclone-v1.53.1-linux-arm/rclone mkdir remote:album/piframe0 
>
You can then add a couple of photos to the album for testing purposes. 
>
Run the synchronization command (pull images from Google Photos into /mnt/fotos): 
>
> ~/rclone-v1.53.1-linux-arm/rclone sync piframe:album/piframe0 /mnt/fotos 
>
Mount the .bin file as mass storage so that the digital photo frame interprets it as a USB stick that was just connected: 
>
> sudo modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1 
>
Enjoy the photos in your digital frame. They might take a few seconds to load.. 

## Setting it up after a reboot
If you reboot the RPi you will need to run a few commands. *This can be optimized!* One of the improvements to be done is to integrate the mount command within fstab, but I still haven't managed to do it. Another improvement is to turn put these into a bash file and execute at boot time.

> sudo mount -t cifs -o credentials=/home/pi/.nascreds,vers=2.0,uid=pi //192.168.1.27/others/piframe /mnt/fotosnas
>
> sudo mount -a # fstab was read during the boot process, but /mnt/fotosnas was not yet mounted, so it didn't work. Let's try again with this command.
> 
> sudo modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1 # "attach the USB storage" to the photo frame.

In summary: we mount the NAS folder that contains the .bin file in /mnt/fotosnas. Then we mount the .bin file as /mnt/fotos. Finally, we simulate attaching the USB storage into the digital photo frame. 
Mounting /mnt/fotos at boot time (so before we mount the .bin file) does not make sense. I'm hoping I can include the first mount command into fstab, eventually.

## Updating images

This can be turned into a bash file and run daily or whenever you want, via a cron job. Or manually, each time someone makes changes to the album.

> sudo modprobe -r g_mass_storage # Simulate the removing of the USB storage from the digital frame. Pictures will stop playing.
> 
> ~/rclone-v1.53.1-linux-arm/rclone sync piframe:album/piframe0 /mnt/fotos #Sync photos with Google Photos
> 
> sync # Not sure this is needed but it won't hurt
> 
> sudo modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1 # Simulate the attachment of the USB storage to the Digital Frame. It should start playing the new photos after a few seconds (depends on the frame, I assume).

## More info about rclone:
https://rclone.org/googlephotos/ 
rclone has very good documentation, so it should be relatively easy to set up that part. If you understood the commands above, you're practically there already.
