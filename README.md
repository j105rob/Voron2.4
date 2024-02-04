# Voron V2.4
### Voron 2.4 Notes &amp; Stuff

#### Update Firmware on the MCUs & U2C

When getting the MCU Protocol Error message in the UI (e.g. Mainsail), you need to update firmware on the MCUs listed in the error message. These instructions assume that you have already installed Klipper on the MCUs and this is just a process on how to update the firmware. 

NOTE: Always do the main MCU last! My set up has one toolhead mcu (BTT EBB2040) and one main mcu (BTT Octopus v1.1).

##### Menuconfig for MCUs:
- Mainboard BTT Octopus is STM32F446 family, 16 MHz clock, comms on PA11/PA12 USB, 8KiB bootloader offset, always double check via datasheet.
- Toolhead BTT EBB2040 is STM32G0B1 family, 8 MHz clock, comms on PB0/PB1 canbus, 8KiB bootloader offset, UUID 0f6028239d4f.

##### Source Update to Latest
- Review the [Configuration Changes](https://github.com/Klipper3d/klipper/blob/master/docs/Config_Changes.md) documentation for the release to make sure you do not have a conflict.
- Pull latest version of `klipper` source.
  ```
  cd ~/klipper
  git pull
  ~/klipper/scripts/install-octopi.sh
  ```
##### Compile firmware for Toolhead
- Recompile the firmware for the toolhead using the `menuconfig` settings. VERIFY THIS!
  ```
  make menuconfig
  make clean
  make
  ```
##### Flash the Toolhead
- Double click the `RESET` button on the Toolhead to put in into `katapult` mode or run:
  ```
  python3 ~/katapult/scripts/flashtool.py -i can0 -u 0f6028239d4f -r
  ```
  Output:
  ```
    voron@voron:~/klipper $ python3 ~/katapult/scripts/flashtool.py -i can0 -u 0f6028239d4f -r
    Sending bootloader jump command...
    Bootloader request command sent
    Flash Success
  ```
  Nothing has been flashed at this point, that utility just puts the toolhead into `katapult` mode.
  
- Verify toolhead is in `katapult` mode. Query result will show the application for the UUID is `katapult`.
  ```
  python3 ~/katapult/scripts/flashtool.py -q
  ```
  Output:
  ```
  voron@voron:~/klipper $ python3 ~/katapult/scripts/flashtool.py -q
  Resetting all bootloader node IDs...
  Checking for Katapult nodes...
  Detected UUID: 0f6028239d4f, Application: Katapult
  Query Complete
  ```
- Flash the `klipper` firmware on the toolhead using `katapult`.
  ```
  python3 ~/katapult/scripts/flashtool.py -i can0 -u 0f6028239d4f -f ~/klipper/out/klipper.bin
  ```
  Output:
  ```
  voron@voron:~/klipper $ python3 ~/katapult/scripts/flashtool.py -i can0 -u 0f6028239d4f -f ~/klipper/out/klipper.bin
  Sending bootloader jump command...
  Resetting all bootloader node IDs...
  Attempting to connect to bootloader
  Katapult Connected
  Protocol Version: 1.0.0
  Block Size: 64 bytes
  Application Start: 0x8002000
  MCU type: stm32g0b1xx
  Verifying canbus connection
  Flashing '/home/voron/klipper/out/klipper.bin'...
  
  [##################################################]
  
  Write complete: 14 pages
  Verifying (block count = 427)...
  
  [##################################################]
  
  Verification Complete: SHA = 7623A988E50A14B273CE68E5F308EC4F4C9D5669
  Flash Success
  ```
- Verify `klipper` has been flashed. The same UUID but now the application says `klipper`. 
  ```
  voron@voron:~/klipper $ python3 ~/katapult/scripts/flashtool.py -i can0 -q
  Resetting all bootloader node IDs...
  Checking for Katapult nodes...
  Detected UUID: 0f6028239d4f, Application: Klipper
  Query Complete
  ```
- Reset toolhead; restart firmware via `mainsail`. Verify firmware version on the `machine` tab.

##### Compile firmware for main MCU
- Recompile the firmware for the main mcu using the `menuconfig` settings. VERIFY THIS!
  ```
  make menuconfig
  make clean
  make
  ```
##### Flash Main MCU
- Get the device address of the device in DFU mode
  ```
  voron@voron:~/klipper $ lsusb
  Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
  Bus 001 Device 009: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
  Bus 001 Device 003: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter
  Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
  Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
  ```
- Flash it, this might fail.
  ```
  voron@voron:~/klipper $ make flash FLASH_DEVICE=/dev/ttyACM0
  Building hid-flash
  gcc -c -Wall -std=gnu99 -I . `pkg-config libusb-1.0 --cflags` main.c -o main.o
  gcc -c -Wall -std=gnu99 -I . `pkg-config libusb-1.0 --cflags` hid-libusb.c -o hid-libusb.o
  gcc -c -Wall -std=gnu99 -I . `pkg-config libusb-1.0 --cflags` rs232.c -o rs232.o
  gcc -no-pie main.o hid-libusb.o rs232.o `pkg-config libusb-1.0 --libs` -lrt -lpthread -o hid-flash
    Flashing out/klipper.bin to /dev/ttyACM0
  Entering bootloader on /dev/ttyACM0
  Device reconnect on /sys/devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.2/1-1.2:1.0
  sudo dfu-util -p 1-1.2 -R -a 0 -s 0x8008000:leave -D out/klipper.bin
  
  dfu-util 0.9
  
  Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
  Copyright 2010-2016 Tormod Volden and Stefan Schmidt
  This program is Free Software and has ABSOLUTELY NO WARRANTY
  Please report bugs to http://sourceforge.net/p/dfu-util/tickets/
  
  dfu-util: Invalid DFU suffix signature
  dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
  dfu-util: No DFU capable USB device available
  
  Failed to flash to /dev/ttyACM0: Error running dfu-util
  
  If the device is already in bootloader mode it can be flashed with the
  following command:
    make flash FLASH_DEVICE=0483:df11
    OR
    make flash FLASH_DEVICE=1209:beba
  
  If attempting to flash via 3.3V serial, then use:
    make serialflash FLASH_DEVICE=/dev/ttyACM0
  
  make: *** [src/stm32/Makefile:111: flash] Error 255
  ```
- Might have to do this:
  ```
    voron@voron:~/klipper $ make flash FLASH_DEVICE=0483:df11
    Flashing out/klipper.bin to 0483:df11
    sudo dfu-util -d ,0483:df11 -R -a 0 -s 0x8008000:leave -D out/klipper.bin
    
    dfu-util 0.9
    
    Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
    Copyright 2010-2016 Tormod Volden and Stefan Schmidt
    This program is Free Software and has ABSOLUTELY NO WARRANTY
    Please report bugs to http://sourceforge.net/p/dfu-util/tickets/
    
    dfu-util: Invalid DFU suffix signature
    dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
    Opening DFU capable USB device...
    ID 0483:df11
    Run-time device DFU version 011a
    Claiming USB DFU Interface...
    Setting Alternate Setting #0 ...
    Determining device status: state = dfuERROR, status = 10
    dfuERROR, clearing status
    Determining device status: state = dfuIDLE, status = 0
    dfuIDLE, continuing
    DFU mode device DFU version 011a
    Device returned transfer size 2048
    DfuSe interface name: "Internal Flash  "
    Downloading to address = 0x08008000, size = 29084
    Download	[=========================] 100%        29084 bytes
    Download done.
    File downloaded successfully
    Transitioning to dfuMANIFEST state
    dfu-util: can't detach
    Resetting USB to switch back to runtime mode
  ```
- Might get stuck in DFU mode, so power cycle the printer.
- Verify firmware version in the machine tab.

##### Flash the BTT U2C

- Unplug the CAN cable from the BTT EBB2040 MCU. It can't be powered while updating the U2C firmware.
- Unplug the USB from the Pi for the U2C, hold the `BOOT` button, plug in the U2C USB cable to the Pi, then release the `BOOT` button. You are now in DFU mode.
- Check to see if the U2C is in DFU mode and verify the Address
  ```
  sudo dfu-util -l
  ```
  Output:
  ```
  voron@voron:~ $ sudo dfu-util -l
  [sudo] password for voron: 
  dfu-util 0.9
  
  Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
  Copyright 2010-2016 Tormod Volden and Stefan Schmidt
  This program is Free Software and has ABSOLUTELY NO WARRANTY
  Please report bugs to http://sourceforge.net/p/dfu-util/tickets/
  
  Found DFU: [0483:df11] ver=0200, devnum=5, cfg=1, intf=0, path="1-1.1", alt=2, name="@Internal Flash   /0x08000000/64*02Kg", serial="206D39855542"
  Found DFU: [0483:df11] ver=0200, devnum=5, cfg=1, intf=0, path="1-1.1", alt=1, name="@Internal Flash   /0x08000000/64*02Kg", serial="206D39855542"
  Found DFU: [0483:df11] ver=0200, devnum=5, cfg=1, intf=0, path="1-1.1", alt=0, name="@Internal Flash   /0x08000000/64*02Kg", serial="206D39855542"
  ```
- Flash the firmware, you can ignore the error in the output.
  ```
  sudo dfu-util -D ~/U2C_V2_STM32G0B1.bin -a 0 -s 0x08000000:leave
  ```
  Output:
  ```
  voron@voron:~ $ sudo dfu-util -D ~/U2C_V2_STM32G0B1.bin -a 0 -s 0x08000000:leave
  dfu-util 0.9
  
  Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
  Copyright 2010-2016 Tormod Volden and Stefan Schmidt
  This program is Free Software and has ABSOLUTELY NO WARRANTY
  Please report bugs to http://sourceforge.net/p/dfu-util/tickets/
  
  dfu-util: Invalid DFU suffix signature
  dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
  Opening DFU capable USB device...
  ID 0483:df11
  Run-time device DFU version 011a
  Claiming USB DFU Interface...
  Setting Alternate Setting #0 ...
  Determining device status: state = dfuIDLE, status = 0
  dfuIDLE, continuing
  DFU mode device DFU version 011a
  Device returned transfer size 1024
  DfuSe interface name: "Internal Flash   "
  Downloading to address = 0x08000000, size = 4804
  Download	[=========================] 100%         4804 bytes
  Download done.
  File downloaded successfully
  dfu-util: Error during download get_status
  ```
- Unplug and plug in the U2C from the Pi to reset and load the new firmware. You might need to reboot the Pi.



#### Resources
- [Klipper FAQs - How do I upgrade to the latest software?](https://github.com/Klipper3d/klipper/blob/master/docs/FAQ.md#how-do-i-upgrade-to-the-latest-software)
- [BTT U2C Flashing Instructions](https://github.com/Esoterical/voron_canbus/tree/main/can_adapter/BigTreeTech%20U2C%20v2.1)
- [BTT EBB2040 Klipper/Katapult Settings](https://github.com/Esoterical/voron_canbus/tree/main/toolhead_flashing/common_hardware/BigTreeTech%20SB2209%20and%20SB2240)
- [BTT EBB2040 CANBus toolhead Klipper update](https://github.com/Esoterical/voron_canbus/blob/main/Updating.md#updating-toolhead-klipper)


  
  
