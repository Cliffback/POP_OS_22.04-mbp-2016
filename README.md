# POP_OS_22.04-mbp-2016
 A repackaged POP!_OS distro with drivers for the MacBook Pro 13,3 (Late 2016).
 
# Need to format this text better 


## Sources:
https://dev.to/cmiranda/linux-on-macbook-pro-2016-1onb
https://github.com/Dunedan/mbp-2016-linux
https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7


## RESIZE PARTITIONS
The POP_OS installer apparently uses a different block size than the macOS disk, at least in my case, making the partitions being listed as 8 times smaller than their real size. This is not an issue after the installation, as the sizes are listed correctly there. However it forces you to make an EFI partition of around 6 GB, which is unfortunate.

I solved this by doing the following.
Source: https://superuser.com/a/1289122


Ideally this should be done from a live usb

    Mount the ESP, if it's not mounted already:

     # mount /dev/nvm0n1p5 /mnt # replace nvm0n1p5 with ESP

    Make a backup of its contents:

     # mkdir ~/esp
     # rsync -av /mnt/ ~/esp/

    Unmount the ESP:

     # umount /mnt

    Delete and recreate the ESP:

     # gdisk /dev/nvm0n1 # replace nvm0n1 with disk containing ESP
     p (list partitions)
     (ensure the ESP is the first partition)
     d (delete partition)
     1 (select first partition)
     n (create partition)
     Enter (use default partition number, should be 1)
     Enter (use default first sector, should be 2048)
     Enter (use default last sector, should be all available space)
     EF00 (hex code for EFI system partition)
     w (write changes to disk and exit)

    Format the ESP:

     # partprobe /dev/nvm0n1
     # mkfs.fat -F32 /dev/nvm0n1p5

    Restore the ESP's contents:

     # mount /dev/nvm0n1p5 /mnt
     # rsync -av ~/esp/ /mnt

    Update EFI entry in /etc/fstab

     # blkid | grep EFI
     # nano /etc/fstab
     PARTUUID=XXXX-XXXX # Replace with PARTUUID of EFI partition from blkid




## WIFI
Download: https://gist.githubusercontent.com/cristianmiranda/6f269797b62076c3414c3baa848dda67/raw/6508ff1f7ae10f45756d1c7437619a529f0a00ad/brcmfmac43602-pcie.txt

cp brcmfmac43602-pcie.txt /lib/firmware/brcm

## AUDIO
https://github.com/davidjo/snd_hda_macbookpro
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
#run the following command as root or with sudo
./install.cirrus.driver.sh


## TOUCHBAR
https://github.com/PatrickVerner/macbook12-spi-driver

echo -e "\n# applespi\napplespi\nspi_pxa2xx_platform\nintel_lpss_pci" >> /etc/initramfs-tools/modules

apt install dkms
git clone https://github.com/PatrickVerner/macbook12-spi-driver.git /usr/src/applespi-0.1
dkms install -m applespi -v 0.1

### add rebinding usb drivers each boot
https://github.com/roadrunner2/macbook12-spi-driver/issues/42

#42
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

Essentially, what you need is to write a file at /etc/systemd/system and add the contents of #42 (comment) to it. (For example, I've named mine /etc/systemd/system/macbook-quirks.service.)

Then, enable your unit with systemctl enable macbook-quirks.service (assuming you named the file macbook-quirks.service). This should hopefully fix your issues.


## ADJUST TRACKPAD SENSITIVITY

sudo nano /usr/share/libinput/local-overrides.quirks

Add content:
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


then:
sudo systemd-hwdb update


## PALM REJECTION: https://gist.github.com/peterychuang/5cf9bf527bc26adef47d714c758a5509


## ALTERNATIVE TRACKPAD: https://int3ractive.com/blog/2018/make-the-best-of-macbook-touchpad-on-ubuntu/


## SWITCH GPU


sudo su

# Blacklist amdgpu
echo "blacklist amdgpu" > /etc/modprobe.d/blacklist-amdgpu.conf

# Switch to integrated GPU
cd && git clone https://github.com/0xbb/gpu-switch
cd gpu-switch
sudo ./gpu-switch -i


### Turn off AMD GPU-stuff (broke installation last time I tried):
https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286168538
https://github.com/Dunedan/mbp-2016-linux/issues/6#issuecomment-286092107
but in this loaction: sudo nano /usr/share/X11/xorg.conf.d/20-intel.conf

commands:
gpu-manager
sudo modprobe amdgpuworks
command lspci | grep "VGA

vgaswitcheroo missing: https://askubuntu.com/questions/57059/sys-kernel-debug-vgaswitcheroo-missing

### GPU BENCHMARK
Use to check which GPU is active

sudo apt-get install glmark2
glmark2

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


Lastly:
gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_ctrl_win']"

Tips:
view options
setxkbmap -print -verbose 10

remove options
setxkbmap -option


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
  
  

