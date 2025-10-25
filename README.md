# PIC K150 usb serial traffic sniffer using QEmu from WinXP VM on debian 13 

Prerequisite : Install Debian 13 on a **non virtual** host

## Verify device on linuxDevice for K150 programmer

Plug your K150 in the host

Verify that it is correctly seen by the OS :

    sudo dmesg

Look for the USB stuff, and the `pl2303` which is what the K150 used to use :

    [1373310.459289] usb 1-2: new full-speed USB device number 3 using xhci_hcd
    [1373310.608360] usb 1-2: New USB device found, idVendor=067b, idProduct=2303, bcdDevice= 3.00
    [1373310.608375] usb 1-2: New USB device strings: Mfr=1, Product=2, SerialNumber=0
    [1373310.608382] usb 1-2: Product: USB-Serial Controller
    [1373310.608387] usb 1-2: Manufacturer: Prolific Technology Inc.
    [1373310.614112] pl2303 1-2:1.0: pl2303 converter detected
    [1373310.615092] usb 1-2: pl2303 converter now attached to ttyUSB0

Verify what group is the owner of the `ttyUSB0` device

    stat /dev/ttyUSB0

Look for the `Gid` field :

    File: /dev/ttyUSB0
    Size: 0               Blocks: 0          IO Block: 4096   character special file
    Device: 0,5     Inode: 785         Links: 1     Device type: 188,0
    Access: (0660/crw-rw----)  Uid: (    0/    root)   Gid: (   20/ dialout)
    Access: 2025-10-17 15:30:15.251700585 +0200
    Modify: 2025-10-17 15:30:15.251700585 +0200
    Change: 2025-10-17 15:30:15.251700585 +0200
    Birth: 2025-10-17 15:30:15.223700774 +0200

Ensure that your user is part of that owning group

    sudo usermod -a -G dialout $USER

IMPORTANT: Logout and reconnect from your desktop or ssh session or the added group will not be applied and you will not be able to use the programming device

## Starting a Windows XM VM with Qemu and sharing the K150 with it

Credits :
- https://computernewb.com/wiki/QEMU/Guests/Windows_XP
- https://linux-kvm.org/page/Change_cdrom

Install required packages

    sudo apt install qemu-system-amd64 qemu-utils qemu-system-gui curl ca-certificates --no-install-recommends -y

    sudo usermod -a -G kvm $USER

IMPORTANT: Logout and reconnect from your desktop or ssh session or the added group will not be applied and you will not be able to use the programming device

Prepare the working space

    mkdir -p ~/winxp-k150
    cd ~/winxp-k150

Then get the images :

    # windows (one of them) for example from https://computernewb.com/wiki/QEMU/Guests/Windows_XP
    curl -LO https://archive.org/download/fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982_202012/fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982.iso
    # curl -LO https://computernewb.com/isos/windows/en_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73974.iso
    # curl -LO https://computernewb.com/isos/windows/en_win_xp_pro_x64_with_sp2_vl_x13-41611.iso

    # floppy
    curl -LO https://computernewb.com/isos/driver/xp_q35_x86.img
    # curl -LO https://computernewb.com/isos/driver/xp_q35_x64.img

    # virtio iso
    curl -LO https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

Create the disk image

    qemu-img create -f qcow2 k150.qcow2 5G

    qemu-img create -f qcow2 c.qcow2 5G

Launch once to start the install

    qemu-system-i386 \
        -enable-kvm \
        -m 512 \
        -hda c.qcow2 \
        -cdrom fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982.iso \
        -boot d

    qemu-system-i386 \
        -M q35,usb=on,acpi=on,hpet=off \
        -global q35-pcihost.x-pci-hole64-fix=false \
        -m 1G \
        -cpu host \
        -accel kvm  \
        -drive if=virtio,file=k150.qcow2 \
        -drive if=floppy,file=xp_q35_x86.img,format=raw \
        -device usb-tablet \
        -device VGA,vgamem_mb=64 \
        -nic user,model=virtio \
        -device qemu-xhci \
        -device usb-host,vendorid=0x067b,productid=0x2303 \
        \
        -cdrom fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982.iso \
        -boot d

To proceed with the install

- Get the focus, and press F6 before it disappears
- When speaking of additional peripherals
  - press S, select a device, press enter
  - press S, select the other device, press enter
  - verify that the two devices are listed
  - press enter to validate devices
- press enter to install
- press F8 to accept
- press enter to install
- select NTFS fast and enter
- after the first reboot
  - click yes to install unsigned driver (once for each driver)
  - enter the key MRX3F 47B9T 2487J KWKMF RPWBY
  - finish the install as usual
- after the second reboot
  - create a desktop shortcut to `cmd.exe`
  - create a desktop shortcur to `devmgmt.msc`
- launch device management shortcut
  - right click ethernet controler
  - update driver
  - no
  - specific location
  - list or location (advanced)
  - these locations (check removables)

Finally once booted, create some useful shortcuts on the desktop :

    devmgmt.msc
    cmd.exe


Poweroff the VM, and start it again with networking to load drivers

    qemu-system-i386 \
        -enable-kvm \
        -m 512 \
        -hda c.qcow2 \
        -nic user,model=virtio \
        -cdrom virtio-win.iso


    # NETWORKING
    # open device manager (win+r / cmd.exe / devmgmt.msc)
    # right click uninitialized ethernet controller, and install the driver
    # do not search on windows update, specific location, cd-rom
    # accept the version of the driver found in the **xp** folder

    # launch internet explorer
    # 
    # 
    # 
    # 
    # 
    # 


    # VGA
    # ignore the missing driver


Download and install 


Now setup USB

    qemu-system-i386 \
        -enable-kvm \
        -m 512 \
        -hda c.qcow2 \
        -nic user,model=virtio \
        -cdrom virtio-win.iso \
        -usb \
        -device usb-ehci,id=ehci


Then poweroff and reboot once again with the final USB device to share

    lsusb | grep PL2303
    Bus 002 Device 005: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port / Mobile Phone Data Cable

    qemu-system-i386 \
        -enable-kvm \
        -m 512 \
        -hda c.qcow2 \
        -nic user,model=virtio \
        -cdrom virtio-win.iso \
        -usb \
        -device usb-ehci,id=ehci \
        -device usb-host,bus=ehci.0,vendorid=0x067b,productid=0x2303

        -device usb-host,hostbus=2,hostport=5
        -device usb-host,bus=usb-bus.0,hostbus=2,hostport=5
        -device usb-host,bus=ehci.0,hostbus=2,hostport=5

        -device usb-host,vendorid=0x067b,productid=0x2303
        -device usb-host,bus=usb-bus.0,vendorid=0x067b,productid=0x2303


Press CTRL+ALT+2 (with SHIFT on french keyboard) to enter QEMU console

B
    qemu) info usbhost
B
      Bus 2, Addr 5, Port 1.1, Speed 12 Mb/s
        Class 00: USB device 067b:2303
      Bus 1, Addr 5, Port 1.4.2, Speed 12 Mb/s
        Class 00: USB device 045e:0823
      Bus 1, Addr 4, Port 1.4.1, Speed 1.5 Mb/s
        Class 00: USB device 046a:0113

Press CTRL+ALT+1 (with SHIFT on french keyboad) to return to the guest

pcap=mouse.pcap

        -snapshot \
        -device qemu-xhci \



        -device usb-host,vendorid=0x067b,productid=0x2303
        -device usb-host,hostbus=2,hostport=4





        -device qemu-xhci \
        -device usb-host,vendorid=0x067b,productid=0x2303

    # USB controller XHCI provides 1B36:000D


    # K150
    # get the old `3.3.2` usb driver for the Prolific USB-to-serial chip on the K150 either from
    # https://github.com/theAmberLion/Prolific/raw/refs/heads/main/PL2303_Prolific_v3.3.2.105.exe
    # http://www.totalcardiagnostics.com/files/PL2303_64bit_Installer.exe
    # https://oemdrivers.com/usb-prolific-usb-to-serial-driver-3-3-2-102
    # http://wp.brodzinski.net/2014/10/01/fake-pl2303-how-to-install/

    # get DIYpack25ep.zip
    # - either http://www.kitsrus.com/zip/DIYpack25ep.zip from https://www.kitsrus.com/pic.html (beware, my windows defender found a virus in it)
    # - or https://www.bristolwatch.com/k150/K150.rar from https://www.bristolwatch.com/k150/index.htm


    # Winows XP keygen
    # https://computernewb.com/software/windows_activation/%5BWINXP%5D%20XP%20Phone%20Keygen.exe

    # MyPal (firefox-based)
    # https://www.mypal-browser.org/release/mypal-68.14.4.en-US.win32.zip



Now restart and use it as throwaway (the `snapshot` option) sharing your USB device (`-device usb-...` option)

    lsusb
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 001 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
    Bus 001 Device 003: ID 064e:e330 Suyin Corp. HD WebCam
    Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 002 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
    Bus 002 Device 003: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port / Mobile Phone Data Cable

    qemu-system-x86_64 -hda c.qcow -m 512 -enable-kvm -vnc :0 -snapshot -device qemu-xhci -device usb-host,vendorid=0x067b,productid=0x2303





    qemu-img create -f qcow2 k150.qcow2 5G
