# Pi 4 Bootloader Configuration

You can display the currently-active configuration using
```
vcgencmd bootloader_config
```

To change these bootloader configuration items, you need to extract the configuration segment, make changes, re-insert it, then reprogram the EEPROM with the new bootloader. The Raspberry Pi will need to be rebooted for changes to take effect.

```
# Copy the EEPROM image of interest from /lib/firmware/raspberrypi/bootloader/ to pieeprom.bin
rpi-eeprom-config pieeprom.bin > bootconf.txt

# Edit the configuration using a text editor e.g. nano bootconf.txt

# Example change. If you have a UART cable then setting BOOT_UART=1 will help debug boot issues
BOOT_UART=1

# Save the new configuration and exit editor

# Apply the configuration change to the EEPROM image file
rpi-eeprom-config --out pieeprom-new.bin --config bootconf.txt pieeprom.bin
```

To update the bootloader EEPROM with the edited bootloader:

```
# Flash the bootloader EEPROM
# Run 'rpi-eeprom-update -h' for more information
sudo rpi-eeprom-update -d -f ./pieeprom-new.bin
sudo reboot
```

### Subsequent Bootloader Updates

If you update your bootloader via apt, then any configuration changes made using the process described here will be migrated to the updated bootloader.


## Configuration Properties
This section describes all the configuration items available in the bootloader. The syntax is the same as [config.txt](../../configuration/config-txt/) but the properties are specific to the bootloader. [Conditional filters](../../configuration/config-txt/conditional.md) are also supported except for EDID.

### BOOT_UART

If 1 then enable UART debug output on GPIO 14 and 15. Configure the receiving debug terminal at 115200bps, 8 bits, no parity bits, 1 stop bit.

Default: 0  
Version: All  

### WAKE_ON_GPIO

If 1 then 'sudo halt' will run in a lower power mode until either GPIO3 or GLOBAL_EN are shorted to ground.

Default: 1 (0 in original version of bootloader 2019-05-10)  
Version: All  

### POWER_OFF_ON_HALT

If 1 and WAKE_ON_GPIO=0 then switch off all PMIC outputs in halt. This is lowest possible power state for halt but may cause problems with some HATs because 5V will still be on. GLOBAL_EN must be shorted to ground to boot.

Default: 0  
Version: 2019-07-15  

### BOOT_ORDER
The BOOT_ORDER setting allows flexible configuration for the priority of different bootmodes. It is represented as 32bit unsigned integer where each nibble represents a bootmode. The bootmodes are attempted in lowest significant nibble to highest significant nibble order.

E.g. 0x21 means try SD first followed by network boot then stop. Whereas 0x2 would mean try network boot and then stop without trying to boot from the SD card.

The retry counters are reset when switching to the next boot mode.

BOOT_ORDER fields  
The BOOT_ORDER property defines the sequence for the different boot modes. It is read right to left and up to 8 digits may be defined.

* 0x0 - NONE (stop with error pattern)
* 0x1 - SD CARD
* 0x2 - NETWORK
* 0x3 - USB device boot - Reserved - Compute Module only.
* 0x4 - USB mass storage boot (since 2020-09-03)
* 0xf - RESTART (loop) - start again with the first boot order field. (since 2020-09-03)

Default: 0xf41 (0x1 in versions prior to 2020-09-03)  
Version: 2020-04-16  

* Boot mode `0x0` will retry the SD boot if the SD card detect pin indicates that the card has been inserted or replaced.
* The default boot order is `0xf41` which means continuously try SD then USB mass storage.

### MAX_RESTARTS
If the RESTART (`0xf`) boot mode is encountered more than MAX_RESTARTS times then a watchdog reset is triggered. This isn't recommended for general use but may be useful for test or remote systems where a full reset is needed to resolve issues with hardware or network interfaces.

Default: -1 (infinite)  
Version: 2020-09-03  

### SD_BOOT_MAX_RETRIES
The number of times that SD boot will be retried after failure before moving to the next boot mode defined by `BOOT_ORDER`.  
-1 means infinite retries.

Default: 0  
Version: 2020-04-16  

### NET_BOOT_MAX_RETRIES
The number of times that network boot will be retried after failure before moving to the next boot mode defined by `BOOT_ORDER`.  
-1 means infinite retries.

Default: 0  
Version: 2020-04-16  

### DHCP_TIMEOUT
The timeout in milliseconds for the entire DHCP sequence before failing the current iteration.

Minimum: 5000  
Default: 45000  
Version: 2020-04-16  

### DHCP_REQ_TIMEOUT
The timeout in milliseconds before retrying DHCP DISCOVER or DHCP REQ.

Minimum: 500  
Default: 4000  
Version: 2020-04-16  

### TFTP_FILE_TIMEOUT
The timeout in milliseconds for an individual file download via TFTP.

Minimum: 5000  
Default: 30000  
Version: 2020-04-16  

### TFTP_IP
Optional dotted decimal ip address (e.g. "192.168.1.99") for the TFTP server which overrides the server-ip from the DHCP request.  
This may be useful on home networks because tftpd-hpa can be used instead of dnsmasq where broadband router is the DHCP server.

Default: ""  
Version: 2020-04-16  

### TFTP_PREFIX
In order to support unique TFTP boot directories for each Pi the bootloader prefixes the filenames with a device specific directory. If neither start4.elf nor start.elf are found in the prefixed directory then the prefix is cleared.
On earlier models the serial number is used as the prefix, however, on Pi 4 the MAC address is no longer generated from the serial number making it difficult to automatically create tftpboot directories on the server by inspecting DHCPDISCOVER packets. To support this the TFTP_PREFIX may be customized to either be the MAC address, a fixed value or the serial number (default).

* 0 - Use the serial number e.g. "9ffefdef/"
* 1 - Use the string specified by TFTP_PREFIX_STR
* 2 - Use the MAC address e.g. "dc-a6-32-01-36-c2/"

Default: 0  
Version: 2020-04-16  

### TFTP_PREFIX_STR
Specify the custom directory prefix string used when `TFTP_PREFIX` is set to 1. For example:- `TFTP_PREFIX_STR=tftp_test/`

Default: ""  
Version: 2020-04-16  

### PXE_OPTION43
Overrides the PXE Option43 match string with a different string. It's normally better to apply customisations to the DHCP server than change the client behaviour but this option is provided in case that's not possible.

Default: "Raspberry Pi Boot"  
Version: 2020-04-16  

### DHCP_OPTION97
In earlier releases the client GUID (Option97) was just the serial number repeated 4 times. By default, the new GUID format is
the concatenation of the fourcc for RPi4 (0x34695052 - little endian), the board revision (e.g. 0x00c03111) (4-bytes), the least significant 4 bytes of the mac address and the 4-byte serial number.
This is intended to be unique but also provide structured information to the DHCP server, allowing Raspberry Pi4 computers to be identified without relying upon the Ethernet MAC OUID.

Specify DHCP_OPTION97=0 to revert the the old behaviour or a non-zero hex-value to specify a custom 4-byte prefix.

Default: 0x34695052  
Version: 2020-04-16  

### Static IP address configuration
If TFTP_IP and the following options are set then DHCP is skipped and the static IP configuration is applied. If the TFTP server is on the same subnet as the client then GATEWAY may be omitted.

#### CLIENT_IP
The IP address of the client e.g. "192.168.0.32"

Default: ""  
Version: 2020-04-16  

#### SUBNET
The subnet address mask e.g. "255.255.255.0"

Default: ""  
Version: 2020-04-16  

#### GATEWAY
The gateway address to use if the TFTP server is on a differenet subnet e.g. "192.168.0.1"

Default: ""  
Version: 2020-04-16  

#### MAC_ADDRESS
Overrides the Ethernet MAC address with the given value. e.g. dc:a6:32:01:36:c2

Default: ""  
Version: 2020-04-16  

### DISABLE_HDMI
The [HDMI boot diagnostics](./boot_diagnostics.md) display is disabled if DISABLE_HDMI=1. Other non-zero values are reserved for future use.

Default: 0  
Version: 2020-04-16  

From version 2020-09-03 the `disable_splash` property in `config.txt` is no longer checked because the HDMI diagnostics screen is started before `config.txt` is read.

### ENABLE_SELF_UPDATE
Enables the bootloader to update itself from a TFTP or USB mass storage device (MSD) boot filesystem.

If self update is enabled then the bootloader will look for the update files (.sig/.upd) in the boot file system. If the update image differs from the current image then the update is applied and system is reset. Otherwise, if the EEPROM images are byte-for-byte identical then boot continues as normal.

Notes:-
* Self-update is not enabled in SD boot; the ROM can already load recovery.bin from the SD card.
* Before self-update can be used the bootloader must have already been updated to a version which supports self update. The recommended approach would be to use the Raspberry Pi Imager and a spare SD card to update to pieeprom-2020-09-03 then use self-update for subsequent updates.
* For network boot make sure that the TFTP `boot` directory can be mounted via NFS and that `rpi-eeprom-update` can write to it.

Default: 1 (0 in versions prior to 2020-09-03)  
Version: 2020-04-16  

### FREEZE_VERSION
Previously this property was only checked by the `rpi-eeprom-update` script. However, now that self-update is enabled the bootloader will also check this property. If set to 1, this overrides `ENABLE_SELF_UPDATE` to stop automatic updates. To disable `FREEZE_VERSION` you will have to use an SD card boot with recovery.bin.

**Custom EEPROM update scripts must also check this flag.**

Default: 0  
Version: All  

### NETCONSOLE - advanced logging
`NETCONSOLE` duplicates debug messages to the network interface. The IP addresses and ports are defined by the `NETCONSOLE` string.

N.B. NETCONSOLE blocks until the ethernet link is established or a timeout occurs. The timeout value is `DHCP_TIMEOUT` although DHCP is not attempted unless network boot is requested.

#### Format
See https://wiki.archlinux.org/index.php/Netconsole
```
src_port@src_ip/dev_name,dst_port@tgt_ip/tgt_mac
E.g. 6665@169.254.1.1/eth0,6666@/
```
In order to simplify parsing, the bootloader requires every field separator to be present. In the example the target IP address (255.255.255.255) and target mac address (00:00:00:00:00) are assigned default values.

One way to view the data is to connect the test Pi 4 to another Pi running WireShark and select “udp.srcport == 6665” as a filter and select `Analyze -> Follow -> UDP stream` to view as an ASCII log.

`NETCONSOLE` should not be enabled by default because it may cause network problems. It can be enabled on demand via a GPIO filter e.g.
```
# Enable debug if GPIO 7 is pulled low
[gpio7=0]
BOOT_UART=1
NETCONSOLE=6665@169.254.1.1/eth0,6666@/
```

Default: ""  
Version: 2020-09-03  

### USB_MSD_EXCLUDE_VID_PID
A list of up to 4 VID/PID pairs specifying devices which the bootloader should ignore. If this matches a HUB then the HUB won’t be enumerated, causing all downstream devices to be excluded.
This is intended to allow problematic (e.g. very slow to enumerate) devices to be ignored during boot enumeration. This is specific to the bootloader and is not passed to the OS.

The format is a comma-separated list of hexadecimal values with the VID as most significant nibble. Spaces are not allowed.
E.g. `034700a0,a4231234`

Default: “”  
Version: 2020-09-03  

### USB_MSD_DISCOVER_TIMEOUT
If no USB mass storage devices are found within this timeout then USB-MSD is stopped and the next boot mode is selected

Default: 20000 (20 seconds)  
Version: 2020-09-03  

### USB_MSD_LUN_TIMEOUT
How long to wait in milliseconds before advancing to the next LUN e.g. a multi-slot SD-CARD reader. This is still being tweaked but may help speed up boot if old/slow devices are connected as well as a fast USB-MSD device containing the OS.

Default: 2000 (2 seconds)  
Version: 2020-09-03  

### USB_MSD_PWR_OFF_TIME
During USB mass storage boot, power to the USB ports is switched off for a short time to ensure the correct operation of USB mass storage devices. Most devices work correctly using the default setting: change this only if you have problems booting from a particular device. Setting `USB_MSD_PWR_OFF_TIME=0` will prevent power to the USB ports being switched off during USB mass storage boot.

Minimum: 250  
Maximum: 5000  
Default: 1000 (1 second)  
Version: 2020-09-03  

### XHCI_DEBUG
This property is a bit field which controls the verbosity of USB trace messages for mass storage boot mode. Enabling all of these messages generates a huge amount of log data which will slow down booting and may even cause boot to fail. For verbose logs it's best to use `NETCONSOLE`

* Bit 0 - USB descriptors
* Bit 1 - Mass storage mode state machine
* Bit 2 - Mass storage mode state machine - verbose
* Bit 3 - All USB requests
* Bit 4 - Log device and hub state machines
* Bit 5 - Log all xHCI TRBs (VERY VERBOSE)
* Bit 6 - Log all xHCI events (VERY VERBOSE)

By default, no extra debug messages are enabled.

```
# Example: Enable mass storage and USB descriptor logging
XHCI_DEBUG=0x3
```

Default: 0x0  
Version: 2020-09-03  

## config.txt - configuration properties
### boot_load_flags
Experimental property for custom firmware (bare metal).

Bit 0 (0x1) indicates that the .elf file is custom firmware. This disables any compatiblity checks (e.g. is USB MSD boot supported) and resets PCIe before starting the executable.

Default: 0x0  
Version: 2020-09-03  

### uart_2ndstage
If set to 0x1 then enable debug logging to the UART. In newer firmware versions (Raspberry Pi OS 2020-08-20 and later) UART logging is also automatically enabled in start.elf. This is also described on the [Boot options](../../configuration/config-txt/boot.md) page.

The `BOOT_UART` property also enables bootloader UART logging but does not enable UART logging in `start.elf` unless `uart_2ndstage=1` is also set.

Default: 0x0  
Version: 2020-09-03  

### eeprom_write_protect
Controls whether the bootloader and VLI EEPROMs are marked as write protected.

**This has no effect unless the EEPROM `/WP` pin is pulled low (TP5). Similarly, `/WP` low only write protects the EEPROM status register so if EEPROM write protect was not previously defined then the EEPROM would not actually be write protected**

See: [Winbond W25x40cl datasheet](https://www.winbond.com/resource-files/w25x40cl_f%2020140325.pdf)

* 1 - Configures the write protect regions to cover the entire EEPROM.
* 0 - Clears write protect regions
* -1 - Do nothing.

Default: -1  
Version: 2020-09-03  

### bootloader_update
This option may be set to 0 to block self-update without requiring the EEPROM configuration to be updated. This is sometimes useful when updating multiple Pis via network boot because this option can be controlled per Raspberry Pi (e.g. via a serial number filter in config.txt).

Default: 1 (0 in versions prior to 2020-09-03)  
Version: 2020-04-16  

## Advanced boot modes - Network / USB mass storage boot.
For network or USB mass storage boot we recommend updating to bootloader version 2020-09-03 and Raspberry Pi OS 2020-08-20 or newer.

### Updating the bootloader
#### Update using the Raspberry Pi Imager
The easiest method to update the bootloader with a factory default configuration supporting USB boot is to use the Raspberry Pi Imager.
1. Download the [Raspberry Pi Imager](https://www.raspberrypi.org/downloads/)
1. Download the latest [EEPROM recovery image](https://github.com/raspberrypi/rpi-eeprom/blob/master/releases.md)
1. Select `Use Custom` to reformat and flash a **blank** SD card with the EEPROM update enabling USB MSD boot support.

#### Manual bootloader update
* From a Raspberry Pi OS 2020-08-20 or newer SD card:
```
sudo apt update
sudo apt full-upgrade
```
* Run `vcgencmd bootloader_config` to check your current configuration and decide whether to use the factory default config or migrate your existing boot settings.
* Update the bootloader
```
# Install with the factory default configuration (-d).
sudo rpi-eeprom-update -a -d
```

#### Changing the boot mode
To change the default boot mode use the `Boot Options`/`Boot Order` option in [raspi-config](../../configuration/raspi-config.md). Alternatively, edit the EEPROM configuration file manually and set the `BOOT_ORDER` according to the desired boot mode then use `rpi-eeprom-update -d -f ` to update the bootloader.

### USB mass storage boot
This is a new feature and we recommend you check the Raspberry Pi [general discussion forum](https://www.raspberrypi.org/forums/viewforum.php?f=63&sid=c5b91609d530566a752920ca7996eb21) for queries or interoperability questions.

N.B. For other operating systems please check the maintainer's website for USB boot support.

#### Check that the USB mass storage device works under Linux
Before attempting to boot from a USB mass storage device it is advisible to verify that the device works correctly under Linux. Boot using an SD card and plug in the USB mass storage device. This should appears as a removable drive.

*Spinning hard-disk drives nearly always require a powered USB hub. Even if it appears to work you are likely to encounter intermittent failures without a powered USB HUB*

This is especially important with USB SATA adapters which may be supported by the bootloader in mass storage mode but fail if Linux selects [USB Attached SCSI - UAS](https://en.wikipedia.org/wiki/USB_Attached_SCSI) mode.

See this [forum thread](https://www.raspberrypi.org/forums/viewtopic.php?t=245931) about UAS and how to add [usb-storage.quirks](https://www.kernel.org/doc/html/v5.0/admin-guide/kernel-parameters.html) to workaround this issue.

#### Multiple bootable drives
When searching for a bootable partition the bootloader scans all USB mass storage devices in parallel and will select the first to respond. If the boot partition does not contain a suitable start.elf file the next available device is selected.

As with earlier Raspberry Pi models there is no method for specifying the boot device according to the USB topology because this would slow down boot and adds unecessary and hard to support configuration complexity.

N.B. config.txt [conditional filters](../configuration/config-txt/conditional.md) can be used to select alternate firmware in complex device configurations.

### Network boot server configuration
Network boot requires a TFTP and NFS server to be configured. See [Network boot server tutorial](bootmodes/net_tutorial.md)

Additional notes:-
* The MAC address on the Pi 4 is programmed at manufacture and is not derived from the serial number.
```
# mac address (ip addr) - it should start with DC:A6:32
ip addr | grep ether | head -n1 | awk '{print $2}' | tr [a-z] [A-Z]
# serial number
vcgencmd otp_dump | grep 28: | sed s/.*://g
```
