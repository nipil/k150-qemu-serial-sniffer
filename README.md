# USB serial traffic sniffer for PIC K150 programmer and microbrn.exe

Prerequisite : Install Debian 13 on a **non virtual** host

Sources used for this project

- https://github.com/openrazer/openrazer/wiki/Reverse-Engineering-USB-Protocol
- https://computernewb.com/wiki/QEMU/Guests/Windows_XP
- https://linux-kvm.org/page/Change_cdrom

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

## Using virt-manager

Install required packages (for wireshark allow non-root users to capture if member of group wireshark)

    sudo apt install --no-install-recommends -y \
        qemu-system-amd64 qemu-utils qemu-system-gui \
        curl ca-certificates \
        virt-manager libvirt-daemon-system \
        wireshark python3

    sudo usermod -a -G kvm $USER

IMPORTANT: Logout and reconnect from your desktop or ssh session or the added group will not be applied and you will not be able to use the programming device

Prepare the working space

    mkdir -p ~/winxp-k150
    cd ~/winxp-k150

Then get the images :

    curl -LO https://archive.org/download/fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982_202012/fr_windows_xp_professional_with_service_pack_3_x86_cd_vl_x14-73982.iso
    curl -LO https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

Create a VM with `virt-manager`

- from the windows cd rom
- add hardware / storage / cdrom choose `virtio-win.iso`
- add hardware / host usb / PL2303
- change network card device model to `virtio`
- start the vm

To proceed with the install

- press enter to install
- press F8 to accept
- press enter to install
- select NTFS fast and enter
- after the first reboot
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
  - choose the driver with xp in its path
- ignore the missing driver for VGA

On your host, download

- PL 2303 old "fake compatible" drivers
  - https://github.com/theAmberLion/Prolific/raw/refs/heads/main/PL2303_Prolific_v3.3.2.105.exe
  - http://www.totalcardiagnostics.com/files/PL2303_64bit_Installer.exe
  - https://oemdrivers.com/usb-prolific-usb-to-serial-driver-3-3-2-102
  - http://wp.brodzinski.net/2014/10/01/fake-pl2303-how-to-install/
- MicroBrn DIYpack25ep.zip
  - either http://www.kitsrus.com/zip/DIYpack25ep.zip from https://www.kitsrus.com/pic.html (beware, my windows defender found a virus in it)
  - or https://www.bristolwatch.com/k150/K150.rar from https://www.bristolwatch.com/k150/index.htm

And get/compile any HEX file compatible with your target chip

    # put it in the download folder too !

In the download folder of your host

    python3 -m http.server

From the guest, open in internet explorer

    http://YOUR_HOST_REAL_IP:8000

From the guest

- get all the downloaded files
- unpack and install driver
- unpack programmer software
- in device manager, update driver for PL 2303
- note the number of the final COM port
- launch microbrn.exe
- select programmer K150
- select port COM3 (for example)

On your host

    # ctrl-C of python http.server

Then load the usb monitor driver

    sudo modprobe usbmon

USB monitor devices number mean the following

- number `0` monitors all USB busses
- number `x` monitors USB bus X

USB bus is displayed for each device

    lsusb
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 001 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
    Bus 001 Device 003: ID 064e:e330 Suyin Corp. HD WebCam
    Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 002 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
    Bus 002 Device 003: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port / Mobile Phone Data Cable

From host, now _we can finally capture USB dialog between `microbrn.exe` and k150 programmer_ !

- `sudo wireshark` (unsafe)
- Start capture from `usbmon2` (example bus 2 from above)
- from guest `microbrn.exe`
  - try "Check erase"
  - try "Read"
  - try "Program"
- power off cleanly the VM

Finally from the host

- stop wireshark capture
- save capture as `.pcapng` file

And analyze it !

If you want to do a text capture, you can instead

    sudo tcpdump -qAXni usbmon2 -s 0
