# spacemonkey_root
# Process for rooting Space Monkey device

## Minimal Procedure
- Open device
    - Power off device
    - Remove label and screw from bottom of device
    - Pry apart plastic shell with much force (ideally without injuring yourself or others)
- Create bootable disk
    - Partition a new disk
        - gpt is ok
        - Linux can use virtually anything as a root FS, but UBoot can boot from ext3 (perhaps not ext4?)
    - Install a root FS
        - default UBoot attempts to boot from `/dev/sda1`. UBoot probably enumerates USB devices starting at `/dev/sdb`, but requires the kernel to be on IDE 0.
        - mount root partition at `/tmp/smroot`
        - `sudo debootstrap --arch armel --foreign sid /tmp/smboot/`
    - set up a chroot
        - `sudo apt-get install qemu-user-static`
        - `sudo cp /usr/bin/qemu-arm-static /tmp/smboot/usr/bin/`
        `- sudo chroot /tmp/smboot`
    - finish bootstrap in chroot
        - `/debootstrap/debootstrap --second-stage`
    - set a root password in chroot
        - `passwd`
    - install some things in chroot
        - `apt-get install u-boot-tools openssh-server`
    - set a hostname in /etc/hostname
    - enable DHCP on eth0 and set the correct MAC address (it’s on the sticker from the bottom of the device)
        - Put the following in /etc/systemd/network/eth0.network:
                [Match]
                Name=eth0
                
                [Network]
                DHCP=both
                
                [Link]
                MACAddress=<your MAC here>
    - Enable networkd
        - `systemctl enable systemd-networkd.service`
    - enable root login over ssh (just for now; you can disable it later)
        - `sed -i /etc/ssh/sshd_config -re 's/^(PermitRootLogin)[[:space:]]+.*/\1 yes/'`
    - exit chroot
        - `exit`
- Install a kernel at /boot/uImage on the root FS
    - `sudo wget https://github.com/Ft1Byh0hl/spacemonkey_data/blob/master/uImage-3.19?raw=true -O /tmp/smboot/boot/uImage`
- Connect bootable disk and network, boot system, find IP address from DHCP
    - check your router/DHCP server for new leases
- login to the system with ssh using the password you set previously
    - `ssh root@<IP address>`
- Get original disk passkey from UBoot environment
    - put the following in /etc/fw_env.config:
            # Configuration file for fw_(printenv/setenv) utility.
            # MTD Device Offset Size Sector Size
            /dev/mtd0 0xE0000 0x20000 0x20000
    - `export hdd_password=$(fw_printenv  | sed -nre 's/hdd_password=(.*)/\1/p')`
        - write this down!
- Get root on original FS with passkey
    - connect original disk to another machine
        - screws holding rubber pads are ~ 7/64” hex (~3mm)
        - this connection probably has to be direct SATA or eSATA; I doubt ATA security commands work over USB-SATA adapters.
    - unlock disk with passkey from above (stored in hdd_password shell environment variable), assuming original Space Monkey disk is `/dev/sdb`.
        - `hdparm --security-unlock $hdd_password /dev/sdb`
    - Set a new root password, add another user with sudo, add your ssh key, etc.
    - Connect original Space Monkey disk back to Space Monkey device
- Enjoy root!

## Optional

- Serial access - super handy!
    - https://imgur.com/0EVtcnw
- Compile kernel
    - ...
    - remember to set MAC address in dts file; otherwise, you’ll get 70:93:f8:00:09:7b
    - ...

## UBoot environment
bootargs=${console} root=/dev/sda1  
bootcmd=run reset_button_test; run bootstrap_test; run upgrade_test; run hdd_boot; run nand_boot; run led_fail;  
bootdelay=0  
bootstrap_test=if test ${sm_bootstrap}_ = true_; then run nand_boot; fi  
clear_sm_upgrade=setenv sm_upgrade false; saveenv  
console=console=ttyS0,115200  
ethact=egiga0  
hdd_boot=ide reset; run hdd_unlock; run hdd_load_kernel hdd_set_args hdd_boot_cmd  
hdd_boot_cmd=bootm ${kernel_addr}  
hdd_load_kernel=ext2load ide 0 ${kernel_addr} boot/uImage  
hdd_set_args=setenv bootargs ${console} ${mtdparts} root=/dev/sda1  
hdd_unlock=if test ${hdd_password}_ != _; then if ide unlock ${hdd_password}; then ide freeze; fi; fi  
initrd_addr=0x1100000  
kernel_addr=0x800000  
led_fail=led green off; led blue off; while true; do led red on; sleep 1; led red off; sleep 1; led red on; sleep 1; led   red off; sleep 1; led red on; sleep 1; led red off; sleep 3; done  
mfg_location=Compeq  
mfg_timestamp=Sun Sep 15 08:44:07 2013 UTC  
mtdids=nand0=orion_nand  
mtdparts=mtdparts=orion_nand:1M(uboot),4M(uImage),10M(recovery),1M(secure),-(data)  
nand_boot=run nand_load_kernel nand_set_args nand_boot_cmd  
nand_boot_cmd=bootm ${kernel_addr}  
nand_load_kernel=nand read ${kernel_addr} nand0,1  
nand_set_args=setenv bootargs ${console} ${mtdparts} root=/dev/mtdblock2 ro   
rootfstype=jffs2  
reset_button_test=if resetbutton; then run nand_boot; fi  
silent=true  
stderr=serial  
stdin=serial  
stdout=serial
upgrade_boot=run clear_sm_upgrade nand_load_kernel upgrade_set_args nand_boot_cmd  
upgrade_set_args=setenv bootargs ${console} ${mtdparts} root=/dev/mtdblock2 ro rootfstype=jffs2 smupgrade=true  
upgrade_test=if test ${sm_upgrade}_ = true_; then run upgrade_boot; fi  
sm_image_timestamp=Sat Sep 21 21:59:20 UTC 2013  
ethaddr=70:93:f8:00:09:7b  
hdd_password=QhzO_C8bkUPuBECBoQNh7YoTO7OTErIN  
sm_preprovision_timestamp=Sat Sep 21 21:59:22 UTC 2013  
sm_bootstrap=false  