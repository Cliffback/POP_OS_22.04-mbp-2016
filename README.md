# POP_OS_22.04-mbp-2016
 A repackaged POP!_OS distro with drivers for the MacBook Pro 13,3 (Late 2016). Not tested on other MacBooks, but will probably work on most Apple T1 devices. 

This was done on macOS Monterey, alongside a Bootcamp partition, resulting in a pretty neat triple boot envirmoent. 

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

### WiFi

[Source](https://dev.to/cmiranda/linux-on-macbook-pro-2016-1onb)

Download [brcmfmac43602-pcie.txt](https://gist.githubusercontent.com/cristianmiranda/6f269797b62076c3414c3baa848dda67/raw/6508ff1f7ae10f45756d1c7437619a529f0a00ad/brcmfmac43602-pcie.txt)  

```
cp brcmfmac43602-pcie.txt /lib/firmware/brcm
```

### Audio

[Source](https://github.com/davidjo/snd_hda_macbookpro)

```
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
#run the following command as root or with sudo
./install.cirrus.driver.sh
```

### TouchBar

[Source](https://github.com/PatrickVerner/macbook12-spi-driver)

Get and install the drivers

```
echo -e "\n# applespi\napplespi\nspi_pxa2xx_platform\nintel_lpss_pci" >> /etc/initramfs-tools/modules

apt install dkms
git clone https://github.com/PatrickVerner/macbook12-spi-driver.git /usr/src/applespi-0.1
dkms install -m applespi -v 0.1
```

Therr is a bug with the USB drivers, that apparently overrides the TouchBar drivers after booting. This can be mitigated by rebinding usb drivers each boot

[Source](https://github.com/roadrunner2/macbook12-spi-driver/issues/42)

```
sudo nano /etc/systemd/system/macbook-quirks.service
```

Add the following contents to the file

```
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

```
systemctl enable macbook-quirks.service 
```

### Adjust Trackpad Sensitivity

Make file

```
sudo nano /usr/share/libinput/local-overrides.quirks
```

Add content:

```
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

```
sudo systemd-hwdb update
```

## Configurations Not Implemented Yet

### Swap Ctrl and Command

[Source](https://askubuntu.com/a/501660)

```
sudo nano /usr/share/X11/xkb/symbols/altwin
```

Add the following mapping abow swap_alt_win

```
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

```
sudo nano /usr/share/X11/xkb/rules/evdev
```

Insert the following line under the option = symbols section in

```
  altwin:swap_ctrl_win  =       +altwin(swap_ctrl_win)
```

Edit this file

```
sudo nano /usr/share/X11/xkb/rules/evdev.lst
```

Add the new option  under the section "! option":

```
  altwin:swap_ctrl_win  Ctrl is swapped with Win
```

Register the rule

```
gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_ctrl_win']"
```

And now it should work!

Helpful commands

```
setxkbmap -print -verbose 10 # See registered options
setxkbmap -option # Remove options currently registered

```

### Stuff I haven't looked into yet

#### Palm Rejection

[Source](https://gist.github.com/peterychuang/5cf9bf527bc26adef47d714c758a5509)

#### Alternative Trackpad 

[Source](https://int3ractive.com/blog/2018/make-the-best-of-macbook-touchpad-on-ubuntu/)




# Generate ISO
Used this guide: https://nixaid.com/linux-on-macbookpro/

Remember to add CASPER_GENERATE_UUID=1 before each line updating initramfs. Example:
CASPER_GENERATE_UUID=1 update-initramfs -u


Updated text: 
sudo xorriso -as mkisofs \
  -r -V "POPOS4MAC" -R -l -o popos4mac.iso \
  -c isolinux/boot.cat -b isolinux/isolinux.bin \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot -isohybrid-gpt-basdat customiso/

  




NEOFETCH
sudo install neofetch
from home: 
sudo nano .bashrc 

if during generating iso, it is in:
/etc/skel/.bashrc

Enable force_color_prompt
Add neofetch to end of .bashrc







SWAP CTRL AND COMMAND

	sudo nano /usr/share/X11/xkb/symbols/altwin
	
	// Add the following mapping abow swap_alt_win
	
	// Control is SWAPPED with Win-keys 
	partial modifier_keys
	xkb_symbols "swap_ctrl_win" {
	    key <LWIN> { [ Control_L ] };
	    key <RWIN> { [ Control_R ] };
	    key <LCTL> { [ Super_L ] };
	    modifier_map Control { <LWIN>, <RWIN> };
	    modifier_map Mod4 { <LCTL> };
	};


	Insert the following line under the option = symbols section in /usr/share/X11/xkb/rules/evdev (disregard the warning on the first line):
	
	  altwin:swap_ctrl_win  =       +altwin(swap_ctrl_win)


	Add the new option to /usr/share/X11/xkb/rules/evdev.lst under the section "! option":
	
	  altwin:swap_ctrl_win  Ctrl is swapped with Win



	gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_ctrl_win']"

Tips:
view options
setxkbmap -print -verbose 10

remove options
setxkbmap -option


generate iso
https://nixaid.com/linux-on-macbookpro/

Remember to add CASPER_GENERATE_UUID=1 before each line updating initramfs. Example:
CASPER_GENERATE_UUID=1 update-initramfs -u


Updated text: 
sudo xorriso -as mkisofs \
  -r -V "POPOS4MAC" -R -l -o popos4mac.iso \
  -c isolinux/boot.cat -b isolinux/isolinux.bin \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot -isohybrid-gpt-basdat customiso/

  





## Tips After Successfull Installation

### Switch GPU to the internal

Blacklist the AMD GPU

	sudo su
	echo "blacklist amdgpu" > /etc/modprobe.d/blacklist-amdgpu.conf 

Switch to integrated GPU

	cd && git clone https://github.com/0xbb/gpu-switch
	cd gpu-switch
	sudo ./gpu-switch -i

Reboot

### Turn off AMD GPU

Turn off the AMD card properly. This didn't work for me, as I don't have a vgaswitcheroo folder. If someone knows how I can work around this, please enlighten me!

Could be a solution here perhaps: https://askubuntu.com/questions/57059/sys-kernel-debug-vgaswitcheroo-missing

```
sudo su
gpu-manager | grep 'amdgpu loaded? no' && sudo modprobe amdgpu || echo 'AMD GPU already loaded'
echo OFF > /sys/kernel/debug/vgaswitcheroo/switch
```

Some of the following instructions broke my installation when I tried, so proceed with caution (and Timeshift backups)

https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286168538
https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286092107

Note that in the last link, the location is different in this installation, so use the following command instad

	sudo nano /usr/share/X11/xorg.conf.d/20-intel.conf

Usefull commands to get GPU information

	gpu-manager
	sudo modprobe amdgpuworks
	command lspci | grep "VGA

To see which GPU is active, use the glmark2 benchmark tool

Install

	sudo apt-get install glmark2

Run the benchmark

	glmark2

