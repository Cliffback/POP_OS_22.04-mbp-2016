# POP_OS_22.04-mbp-2016
A repackaged POP!_OS distro with drivers for the MacBook Pro 13,3 (Late 2016). Not tested on other MacBooks, but will probably work on most Apple T1 devices. 

This was done on macOS Monterey, alongside a Bootcamp partition, resulting in a pretty neat triple boot envirmoent. 

I hope that this will save someone from a lot of work, googling around, and reading up on forum posts and discussions. I've gathered everything I have learnt in the following document. I also take no credit for the methods presented below, and have linked to the source below each heading. 

## Installation

To install, unpack the latest ISO from the releases, and make a bootable USB, using balenaEtcher, rufus or similar tools.

Shrink your macOS partition with Disk Utility.

Hold down the alt key while booting, and choose the USB (should be appear as a yellow drive named EFI Boot).

Choose custom installation, and make a root partiton and a boot partition that's at least 500 MB. If the installer lists the partitons with the wrong size, see the "Resize Partition" heading below. 

If someone knows of a way to mitigate this problem, or update the ISO to use macOS block size, I would be happy to know!


## Resize Partition
The POP_OS installer apparently uses a different block size (4096 bytes) than the macOS disk (512 bytes), at least in my case, making the partitions being listed as eight times smaller than their real size. This is not an issue after the installation, as the sizes are listed correctly there. However it forces you to make an EFI partition of around 6 GB, which is unfortunate. After the installation is finished you can manually resize the partition.

Source: https://superuser.com/a/1289122

Boot to the live USB you just created, and open the terminal

Mount the ESP, if it's not mounted already:

    mount /dev/nvm0n1p5 /mnt # replace nvm0n1p5 with ESP

Make a backup of its contents:

    mkdir ~/esp
    rsync -av /mnt/ ~/esp/

Unmount the ESP:

     umount /mnt

Delete and recreate the ESP:

    gdisk /dev/nvm0n1 # replace nvm0n1 with disk containing ESP
    
    p (list partitions)
    d (delete partition)
    5 (select the EFI partition to be resized)
    n (create partition)
    Enter (use default partition number )
    Enter (use default first sector)
    Enter (enter first sector + the euqivalent of 600 MB)
    EF00 (hex code for EFI system partition)
    w (write changes to disk and exit)

Format the ESP:

    partprobe /dev/nvm0n1
    mkfs.fat -F32 /dev/nvm0n1p5

Format the ESP:

    partprobe /dev/nvm0n1
    mkfs.fat -F32 /dev/nvm0n1p5

Restore the ESP's contents:

     mount /dev/nvm0n1p5 /mnt
     rsync -av ~/esp/ /mnt

Get PARTUUID for the new EFI partition

     blkid | grep EFI

Update EFI entry in /etc/fstab

    nano /etc/fstab

Replace PARTUUID with the new one.

```
PARTUUID=XXXX-XXXX # Replace with PARTUUID of EFI partition from blkid
```

Save the file and you should be good to go

```
Ctrl+O
Enter
Ctrl+X
```

Reboot

## The Changes I've Done

These are the modifications I've done with the ISO.

Useful links:

https://github.com/Dunedan/mbp-2016-linux

https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7#gistcomment-2164350

### WiFi

[Source](https://dev.to/cmiranda/linux-on-macbook-pro-2016-1onb)

Download [brcmfmac43602-pcie.txt](https://gist.githubusercontent.com/cristianmiranda/6f269797b62076c3414c3baa848dda67/raw/6508ff1f7ae10f45756d1c7437619a529f0a00ad/brcmfmac43602-pcie.txt)  

```bash
cp brcmfmac43602-pcie.txt /lib/firmware/brcm
```

### Audio

[Source](https://github.com/davidjo/snd_hda_macbookpro)

```bash
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
#run the following command as root or with sudo
./install.cirrus.driver.sh
```

### TouchBar

[Source](https://github.com/PatrickVerner/macbook12-spi-driver)

Get and install the drivers

```bash
echo -e "\n# applespi\napplespi\nspi_pxa2xx_platform\nintel_lpss_pci" >> /etc/initramfs-tools/modules

apt install dkms
git clone https://github.com/PatrickVerner/macbook12-spi-driver.git /usr/src/applespi-0.1
dkms install -m applespi -v 0.1
```

There is a bug with the USB drivers, that apparently overrides the TouchBar drivers after booting. This can be mitigated by rebinding usb drivers each boot

[Source](https://github.com/roadrunner2/macbook12-spi-driver/issues/42)

```bash
sudo nano /etc/systemd/system/macbook-quirks.service
```

Add the following contents to the file

```bash
[Unit]
Description=Re-enable MacBook 14,3 TouchBar
Before=display-manager.service
After=usbmuxd.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/unbind"
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/bind"
RemainAfterExit=yes
TimeoutSec=0

[Install]
WantedBy=multi-user.target
```

Then, enable the service

```bash
systemctl enable macbook-quirks.service 
```

### Adjust Trackpad Sensitivity

Make file

```bash
sudo nano /usr/share/libinput/local-overrides.quirks
```

Add content:

```bash
[MacBook(Pro) SPI Touchpads]
MatchName=*Apple SPI Touchpad*
ModelAppleTouchpad=1
AttrTouchSizeRange=200:150
AttrPalmSizeThreshold=1100

[MacBook(Pro) SPI Keyboards]
MatchName=*Apple SPI Keyboard*
AttrKeyboardIntegration=internal

[MacBookPro Touchbar]
MatchBus=usb
MatchVendor=0x05AC
MatchProduct=0x8600
AttrKeyboardIntegration=internal
```

Register the file

```bash
sudo systemd-hwdb update
```

### Neofetch

Just to get a nice looking terminal, with useful information.

```bash
sudo install neofetch
```

Then edit file from the home directory

```bash
sudo nano .bashrc 
```

If done during generating iso, the home folder don't exist, so we have to do it here instead

```bash
sudo nano /etc/skel/.bashrc
```

In the file, uncomment

```bash
force_color_prompt
```

and add

```bash
neofetch
```

at the end of the file

## Recommended Configurations (not implemented)

### Swap Ctrl and Command

[Source](https://askubuntu.com/a/501660)

```bash
sudo nano /usr/share/X11/xkb/symbols/altwin
```

Add the following mapping abow swap_alt_win

```bash
// Control is SWAPPED with Win-keys 
partial modifier_keys
xkb_symbols "swap_ctrl_win" {
    key <LWIN> { [ Control_L ] };
    key <RWIN> { [ Control_R ] };
    key <LCTL> { [ Super_L ] };
    modifier_map Control { <LWIN>, <RWIN> };
    modifier_map Mod4 { <LCTL> };
};
```

Edit the following file

```bash
sudo nano /usr/share/X11/xkb/rules/evdev
```

Insert the following line under the option = symbols section in

```bash
  altwin:swap_ctrl_win  =       +altwin(swap_ctrl_win)
```

Edit this file

```bash
sudo nano /usr/share/X11/xkb/rules/evdev.lst
```

Add the new option  under the section "! option":

```bash
  altwin:swap_ctrl_win  Ctrl is swapped with Win
```

Register the rule

```bash
gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_ctrl_win']"
```

And now it should work!

Helpful commands

```bash
setxkbmap -print -verbose 10 # See registered options
setxkbmap -option # Remove options currently registered
```

### Palm Rejection (not tested)

[Source](https://gist.github.com/peterychuang/5cf9bf527bc26adef47d714c758a5509)

### Alternative Trackpad (not tested)

[Source](https://int3ractive.com/blog/2018/make-the-best-of-macbook-touchpad-on-ubuntu/)

## Tips After Successfull Installation

### Switch GPU to the internal

Be sure to spoof the machine into thinking it's booting into macOS to get access to the iGPU. The easiest method is to use rEFind.

Uncomment the "spoof_osx_version" line in refind.conf and you'll get access to the iGPU when booting into Pop!_OS.

Other methods can be found [here](https://dev.to/cmiranda/linux-on-macbook-pro-2016-1onb)


Then blacklist the AMD GPU

```bash
sudo su
echo "blacklist amdgpu" > /etc/modprobe.d/blacklist-amdgpu.conf 
```

Switch to integrated GPU

```bash
cd && git clone https://github.com/0xbb/gpu-switch
cd gpu-switch
sudo ./gpu-switch -i
```

Reboot

### Turn off AMD GPU

Turn off the AMD card properly. This didn't work for me, as I don't have a vgaswitcheroo folder. If someone knows how I can work around this, please enlighten me!

Could be a solution here perhaps: https://askubuntu.com/questions/57059/sys-kernel-debug-vgaswitcheroo-missing

```bash
sudo su
gpu-manager | grep 'amdgpu loaded? no' && sudo modprobe amdgpu || echo 'AMD GPU already loaded'
echo OFF > /sys/kernel/debug/vgaswitcheroo/switch
```

Some of the following instructions broke my installation when I tried, so proceed with caution (and Timeshift backups)

https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286168538
https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286092107

Note that in the last link, the location is different in this installation, so use the following command instad

```bash
sudo nano /usr/share/X11/xorg.conf.d/20-intel.conf
```

Usefull commands to get GPU information

```bash
gpu-manager
sudo modprobe amdgpuworks
command lspci | grep "VGA
```

To see which GPU is active, use the glmark2 benchmark tool

Install

```bash
sudo apt-get install glmark2
```

Run the benchmark

```bash
glmark2
```

## How to Repackage the ISO Yourself

[Source](https://nixaid.com/linux-on-macbookpro/)

Download the image file (I used Pop!_OS 22.04 LTS), and extract the contents. You could even use my ISO to modify it further.

```bash
cd ~/Downloads
sudo mkdir /mnt/iso
sudo mount ubuntu-20.04.2.0-desktop-amd64.iso /mnt/iso
mkdir customiso
rsync -a --exclude=casper/filesystem.squashfs /mnt/iso/ customiso/
sudo unsquashfs /mnt/iso/casper/filesystem.squashfs
sudo umount /mnt/iso
```

Chroot to the image so that we can update it from the inside

```bash
sudo mount --bind /dev squashfs-root/dev/
sudo chroot squashfs-root/
PS1="(chroot) $PS1"
LC_ALL=C
HOME=/root
export PS1 HOME LC_ALL

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devpts none /dev/pts
```

Update the image

```bash
mv /etc/resolv.conf /etc/resolv.conf.bak
echo 'nameserver 8.8.8.8' | tee /etc/resolv.conf

apt-get update
apt-get -y dist-upgrade
apt-get -y autoremove
```

Then do wanted configurations 

Remember to add CASPER_GENERATE_UUID=1 before each line updating initramfs. This makes sure that we can extract the UUID when mastering the ISO. 

Examples:

```bash
CASPER_GENERATE_UUID=1 update-initramfs -u
CASPER_GENERATE_UUID=1 dkms install -m applespi -v 0.1
```

Then exit the chroot enviroment

```bash
rm /var/lib/dbus/machine-id
rm /etc/resolv.conf
mv /etc/resolv.conf.bak /etc/resolv.conf
umount /dev/pts
umount /sys
umount /proc
rm /root/.bash_history
unset HISTFILE
exit
sudo umount squashfs-root/dev/
```

And update the kernel

```bash
sudo cp squashfs-root/boot/vmlinuz customiso/casper/vmlinuz
sudo cp squashfs-root/boot/initrd.img customiso/casper/initrd
```

Then build remastered image

```bash
sudo rm customiso/casper/filesystem.squashfs
sudo mksquashfs squashfs-root customiso/casper/filesystem.squashfs

unmkinitramfs customiso/casper/initrd /tmp/z
sudo cp /tmp/z/main/conf/uuid.conf customiso/.disk/casper-uuid-generic
rm -rf /tmp/z

cd customiso
sudo rm md5sum.txt
sudo find -type f -print0 | xargs -0 sudo md5sum | grep -Ev "./md5sum.txt|./isolinux/" | sudo tee md5sum.txt
cd ..

sudo apt -y install xorriso isolinux

rm ubuntu4mac.iso
sudo xorriso -as mkisofs \
  -r -V "POPOS4MAC" -R -l -o popos4mac.iso \
  -c isolinux/boot.cat -b isolinux/isolinux.bin \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot -isohybrid-gpt-basdat customiso/

sudo chown $(id -u):$(id -g) ubuntu4mac.iso
```
