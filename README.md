

---

# piframe

## Intro

I'm turning a regular USB-powered photo frame into a Google Photos frame using a Raspberry Pi Zero W. The Pi presents itself to the frame as a USB flash drive, so the frame reads it like any stick.

The Pi's SD card is small (4 GB, about 1.8 GB free after a Raspberry Pi OS Lite install), so the photos live on my NAS. The workflow:

1. Add photos to a Google Photos album.
2. Overnight, rclone syncs that album to a FAT32 disk image (`piusb.bin`) stored on the NAS.
3. The Pi exposes that image to the frame as a USB drive.

This could read from a NAS folder directly, but Google Photos is far more convenient for managing the photos.

## Hardware and prerequisites

- Raspberry Pi Zero W with Raspberry Pi OS Lite
- A microSD card (4 GB or larger)
- A NAS with an SMB/CIFS share (this setup doesn't use Samba on the Pi)
- A USB cable from the Pi's **data** port (the inner micro-USB, labelled USB) to the frame
- Power for the Pi from the **PWR IN** port. Don't power it from the frame.
- rclone 1.53 or newer (the `apt` package is too old for Google Photos support; see below)

Base setup follows the [MagPi guide](https://magpi.raspberrypi.org/articles/pi-zero-w-smart-usb-flash-drive) up to step 10, skipping the Samba section. During `sudo raspi-config`, enable **Wait for network at boot**.

## Part 1: Turn the Pi into a USB gadget

Enable the USB gadget driver:

```bash
sudo nano /boot/config.txt
```

Append at the bottom:

```
dtoverlay=dwc2
```

Then:

```bash
sudo nano /etc/modules
```

Add `dwc2` on a new line after `i2c-dev`. Save both files (CTRL+O, Enter, CTRL+X), then shut down:

```bash
sudo halt
```

## Part 2: Create the image and mount it

### Create the folders

```bash
sudo mkdir -p /mnt/fotosnas /mnt/fotos
```

- `/mnt/fotosnas` holds the NAS share, which contains the `.bin` file.
- `/mnt/fotos` shows the contents of the `.bin` file.

### Mount the NAS

NAS credentials are in `/home/pi/.nascreds`, in the standard `username=` / `password=` format. Restrict its permissions:

```bash
chmod 600 /home/pi/.nascreds
sudo mount -t cifs -o credentials=/home/pi/.nascreds,vers=2.0,uid=pi //NAS/others/piframe /mnt/fotosnas
```

### Create the disk image

This writes about 29 GiB to the NAS and can take a long time. Start with a smaller `count` to test.

```bash
sudo dd bs=1M if=/dev/zero of=/mnt/fotosnas/piusb.bin count=30000 status=progress
sudo mkfs.vfat -F 32 -I /mnt/fotosnas/piusb.bin
```

`mkdosfs` is an alias for `mkfs.vfat`, so either name works.

### Make the mounts persistent

Add these lines to **`/etc/fstab`** (not `/mnt/fstab`):

```
//NAS/others/piframe /mnt/fotosnas cifs credentials=/home/pi/.nascreds,vers=2.0,uid=pi,gid=pi,_netdev,nofail 0 0
/mnt/fotosnas/piusb.bin /mnt/fotos vfat users,umask=000,nofail,x-systemd.requires=/mnt/fotosnas 0 0
```

- `_netdev` and `nofail` stop the boot from hanging if the NAS is offline.
- `x-systemd.requires` ensures the NAS is mounted before the image is loaded.

Then mount everything:

```bash
sudo mount -a
```

## Part 3: Sync Google Photos to the image

### Configure rclone

Download the latest ARM build from [rclone.org](https://rclone.org/downloads/) and unzip it in your home folder, then run:

```bash
~/rclone-v1.53.1-linux-arm/rclone config
```

Follow the prompts. The remote name is used throughout this document, so name it `piframe`.

**Note:** Google's current API doesn't let rclone modify albums you created in the Google Photos app. The workaround is to create the album with rclone itself:

```bash
~/rclone-v1.53.1-linux-arm/rclone mkdir piframe:album/piframe0
```

Add a few test photos to that album in the Google Photos app, then run an initial sync:

```bash
~/rclone-v1.53.1-linux-arm/rclone sync piframe:album/piframe0 /mnt/fotos
```

**Warning:** `sync` makes the destination match the source, so it deletes files from `/mnt/fotos` that have been removed from the album. If you'd rather keep old photos, use `copy` instead.

### Expose the image to the frame

```bash
sudo modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1
```

The frame should detect a USB drive and start showing photos after a few seconds.

## Part 4: Update the photos

Every update follows the same sequence:

1. Detach the image from the frame, so the frame stops reading it.
2. Unmount the image, so the Pi finishes all writes.
3. Sync from Google Photos.
4. Reattach the image.

Save this as `/home/pi/piframe-update.sh`:

```bash
#!/bin/bash
set -e

RCLONE=/home/pi/rclone-v1.53.1-linux-arm/rclone
RCLONE_CONF=/home/pi/.config/rclone/rclone.conf

# 1. Detach from the frame (ignore the error if it isn't attached)
sudo modprobe -r g_mass_storage 2>/dev/null || true

# 2. Make sure the NAS and image are mounted
mountpoint -q /mnt/fotosnas || sudo mount /mnt/fotosnas
mountpoint -q /mnt/fotos || sudo mount /mnt/fotos

# 3. Sync the album into the image
"$RCLONE" --config "$RCLONE_CONF" sync piframe:album/piframe0 /mnt/fotos

# 4. Flush writes and unmount so the image is consistent
sync
sudo umount /mnt/fotos

# 5. Reattach to the frame
sudo modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1
```

Make it executable and test it:

```bash
chmod +x /home/pi/piframe-update.sh
/home/pi/piframe-update.sh
```

To run it nightly, add this to the **pi** user's crontab (`crontab -e`, not `sudo crontab -e`):

```
0 3 * * * /home/pi/piframe-update.sh >> /home/pi/piframe-update.log 2>&1
```

Running it as `pi` matters: if root ran it, rclone would look for its config in `/root/.config/rclone` and fail. The script uses `sudo` only for the mount and gadget commands, which Raspberry Pi OS allows without a password for the `pi` user.

## Part 5: Attach the image at boot

After a reboot, the NAS and image mount automatically through fstab. The USB gadget doesn't, so create a systemd unit:

```bash
sudo nano /etc/systemd/system/piframe-gadget.service
```

```ini
[Unit]
Description=Expose piframe image to the digital frame
After=mnt-fotos.mount
Requires=mnt-fotos.mount

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/modprobe g_mass_storage file=/mnt/fotosnas/piusb.bin stall=0 ro=1

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable piframe-gadget.service
```

## Optional: trigger the sync from Home Assistant

If you run Home Assistant, you can trigger a sync on demand. Set up SSH key access from Home Assistant to the Pi, then add a `shell_command` to `configuration.yaml`:

```yaml
shell_command:
  piframe_update: "ssh -i /config/.ssh/id_piframe pi@piframe /home/pi/piframe-update.sh"
```

You can then call `shell_command.piframe_update` from a button or automation.

## Troubleshooting

- **Frame shows no photos:** Check that the image mounts (`mountpoint /mnt/fotos`) and that the gadget is loaded (`lsmod | grep g_mass_storage`).
- **Boot hangs or takes a long time:** Usually the NAS is unreachable. The `nofail` and `_netdev` options should prevent this, but check `journalctl -b` for mount errors.
- **Sync fails under cron:** Check `/home/pi/piframe-update.log`. The most common cause is running the script as root.
- **Corrupted image:** Always run the update script rather than syncing manually while the gadget is attached.

## More info about rclone

- [rclone Google Photos documentation](https://rclone.org/googlephotos/)

rclone's documentation is thorough. If the commands above make sense to you, the rest is mostly configuration.
