# keba_wallbox_firmware_update

Documentation of a procedure to update firmware on a Keba P30 b-series wallbox. Use at your own risk!

This allows to perform a firmware update with a Linux system connected to the wallbox via ethernet.

The following description assumes that the wallbox is connected to the Linux system and that an IP
address has been configured.

## discover wallbox

TCP and UDP functionality is provided via lwIP. The "Device Locator" functionality of lwIP can be used
to discover a connected wallbox. See https://github.com/ragunath3252/lwip-port/blob/master/ports/locator.c 
for an example implementation.

To run this discovery on the Linux command line, use this procedure:

```code
echo "ff0402fb" | xxd -r -p | nc -u ip.addr.of.wb 23
```

You will see a response that includes the currently running firmware.

## firmware update procedure

To initiate the firmware update procedure via Ethernet, a "magic packet" has to be sent. The documentation
for this packet can be found here: https://www.ti.com/lit/ug/spmu063/spmu063.pdf on page 354, chapter
"27.2.2.38 SoftwareUpdateInit".

Once the magic packet is sent, the wallbox goes into firmware update mode. Details are described in the same
document on page 368, chapter "29.1.3 Ethernet Update".

### find out MAC address of wallbox

Find out the MAC address of the wallbox. This example scans device "eth0".

```code
arp-scan -I eth0 -l
```

The result could look like this:

```code
ip.addr.of.wb	00:60:b5:XX:YY:ZZ	KEBA GmbH
```

### setup required tooling

Port numbers are shifted, probably due to privileges, by 55100, e.g.:

BOOTP server: 55167
BOOTP client: 55168
TFTP server:  55169

Tooling must be set up appropriately to handle this. In this example, 
the apporach for Debian and related distributions is documented:

* Install the packages "bootp" and "tftpd-hpa".

* Change the values in "/etc/services" for bootps, bootpc and tftp.

Copy and rename the firmware appropriately. Example:

```code
mkdir -p /tftpboot
sudo cp kec_pdc-v.3.AA.BB.bin /tftpboot/firmware.bin
```

* add the IP address of your system to /etc/hosts with server name "stellaris". Example:

```code
ip.addr.host.system	stellaris
```

* add the following line to /etc/bootptab (fill IP and MAC of Wallbox)

```code
.default:ip=ip.addr.of.wb:bf=firmware.bin:ht=ethernet:ha=0060B5XXYYZZ
```

* Run the tools with these parameters: 

```code
sudo bootpd -s -d 127 -h stellaris -c /tftpboot/
```

(bootpd needs sudo for altering ARP table)

```code
in.tftpd -L --verbosity=127 --blocksize=516 -s --address ip.addr.host.system:55169 /tftpboot/
```

### send magic packet

Send 6 times "aa" and then repeat the MAC address of the wallbox 4 times:

```code
echo "aaaaaaaaaaaa0060b5XXYYZZ0060b5XXYYZZ0060b5XXYYZZ0060b5XXYYZZ" | xxd -r -p | nc -u ip.addr.of.wb 9
```

This starts the firmware update process.

Check with the "discovery" command that the firmware has been updated successfully.
