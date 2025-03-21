# QSW-M2116P-2T2S

The QNAP [QSW-M2116P-2T2S](https://www.qnap.com/en-us/product/qsw-m2116p-2t2s) is a PoE switch with 16 2.5GbE ports, 2 10GbE ports (also support 1/2.5/5G) and 2 SFP+ ports.

It is advertised as a layer 2 web-managed switch, but as shown in this document, it can also do layer 3 routing, be managed with a Cisco IOS-like CLI and it is trivial to obtain a root Linux shell.

This document is based on the latest [firmware](https://www.qnap.com/en/download?model=qsw-m2116p-2t2s&category=firmware) as of March 2025: 2.0.1.32808.

## Overview

The switch uses a [Microchip VSC7448-02 SparX-IV-80](https://www.microchip.com/en-us/product/vsc7448) switch chip ([datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/SparX-IV_L2_L3_Enterprise_Gigabit_Ethernet_Switches_Datasheet_00004426A.pdf)).

The chip contains a 500MHz MIPS CPU on which a proprietary Linux distribution made by Microchip is running: [WebStaX](https://www.microchip.com/en-us/product/vsc6819). This system can be customized by the manufacturer. In thise case QNAP seems to stay relatively close to the base system, mainly adding a custom web server written in Go (`qnssweb`) providing the QNAP [QSS web interface](https://www.qnap.com/en-us/solution/qnap-switch-system).

Microchip has a [documentation](https://ww1.microchip.com/downloads/aemDocuments/documents/UNG/ProductDocuments/UserGuides/UG1068-SW-Introduction-to-WebStaX-on-Linux.pdf
) on building and customizing WebStaX [features](https://ww1.microchip.com/downloads/en/Appnotes/Ethernet_Switch_Software_Features_30010224.pdf), but unfortunately this requires access to proprietary sources which [costs](https://www.microchipdirect.com/dev-tools/VSC6817SDK) $100k.

However the system stays pretty open:
- Obtaining a root shell is trivial
- The firmware image is not encrypted
- The MIPS toolchain seems to be publicly [available](https://github.com/microchip-ung/mesa/releases) (not tested yet)

In addition to these OS-level features, the chip itself offers more features than QNAP advertises for the switch. In particular it supports hardware IP routing with up to 4k IPv4 routes and 1k IPv6 routes, which is very helpful for inter-VLAN routing.

### `show version` dump

```
QSW-M2116P# sh version

MAC Address      : xx-xx-xx-xx-xx-xx
Previous Restart : Cold

System Contact   :
System Name      : QSW-M2116P
System Location  :
System Time      : 2025-03-15T14:08:26+01:00
System Uptime    : 18:41:09


Bootloader
----------
Image            : RedBoot
Version          : version 1_6_build202011271809-c7d6d80
Date             : 18:10:08, Nov 27 2020

Primary Image
-------------
Image            : linux (Active)
Version          : 2.0.1.32808
Date             : 2024-05-31T16:00:51+08:00


------------------
SID : 1
------------------
Chipset ID       : VSC7448 Rev. D
Board Type       : SparX-IV_80_24
Flash Type       : NOR-only
Port Count       : 21
Product          : Microchip IStaX Switch
Software Version : 2.0.1.32808
Build Date       : 2024-05-31T16:00:51+08:00
Code Revision    : 79525bf+
PoE Version      : HW Ver.:0, Prod:24, sw ver:352, param:0, build:30, internal sw ver:1000, Asic Patch Num:0
---
```

```
QSW-M2116P# debug show firmware mfi-info
ImageId      SectionId    Attr.Id      Name                                     Value
-----------  -----------  -----------  ---------------------------------------  ------------------
0            0            0            Image type name                          Bootloader
0            0            1            Image Name                               RedBoot
0            0            2            Image Version                            version 1_6_build202011271809-c7d6d80
0            0            3            Image Built date                         18:10:08, Nov 27 2020
0            0            4            Image Code revision
1            0            0            Image type name                          Active Firmware
1            0            1            Image Name                               linux
1            0            2            Image Version                            2.0.1.32808
1            0            3            Image Built date                         2024-05-31T16:00:51+08:00
1            0            4            Image Code revision                      79525bf+
1            1            0            Section Name                             rootfs
1            1            1            Section Version                          2022.03
1            1            2            Section SQUASHFS contents length         5521408
1            1            3            Section File-name                        wd-istax_sparxIV_80_24-jaguar2_pcb110.squashfs
1            2            0            Section Name                             vtss
1            2            1            Section Version                          79525bf+
1            2            2            Section SQUASHFS contents length         12165120
1            2            3            Section File-name                        istax_sparxIV_80_24.app-rootfs
1            3            0            Section Name                             vtss-web-ui
1            3            1            Section Version                          79525bf+
1            3            2            Section SQUASHFS contents length         589824
1            3            3            Section File-name                        vtss-www-rootfs.squashfs
```


## CLI management

To obtain console access, connect a Cisco console cable ([example](https://www.fs.com/products/170189.html)) to the RJ45 port on the back of the switch. The serial port configuration is 115200 8N1, which is also the default [`tio`](https://github.com/tio/tio) configuration:
```bash
$ tio /dev/ttyUSB0
[17:36:59.220] tio v2.5
[17:36:59.220] Press ctrl-t q to quit
[17:36:59.220] Connected
Username: admin
Password:
QSW-M2116P#
```

The password is the same as the one for the web interface.

This gives us a Cisco IOS-like CLI, provided by a binary called `icli` (also set as the default SSH shell).

### Enable SSH

To enable SSH:
```
QSW-M2116P# conf t
QSW-M2116P(config)# ip ssh
QSW-M2116P(config)# end
```

Then you can connect with the same credentials as the web/console interface:
```
❯ ssh admin@x.x.x.x
admin@x.x.x.x's password:
QSW-M2116P#
```

To save the configuration:
```
QSW-M2116P# copy run start
```

To view the configuration:
```
QSW-M2116P# show run
```

### IPv6 management address

The web interface only shows the IPv4 management IP, but the switches also acquires an IPv6 address through SLAAC. To view it:
```
QSW-M2116P# sh ipv6 interface
Interface Address                                     Status
--------- ------------------------------------------- ------
VLAN 1    fe80::xxxx:xxxx:xxxx:xxxx/64                UP
VLAN 15   2a01:cb08:9155:xxxx:xxxx:xxxx:xxxx:xxxx/64  UP
VLAN 15   fde9:32e2:15b3:xxxx:xxxx:xxxx:xxxx:xxxx/64  UP
VLAN 15   fe80::xxxx:xxxx:xxxx:xxxx/64                UP
```

It's also possible to manually set an IPv6 address:
```
QSW-M2116P(config)# interface vlan 15
QSW-M2116P(config-if-vlan)# ipv6 address dead:beef::/64
```

### Routing

I haven't tested it yet, but according to the datasheet the switches supports IPv4/v6 routing. To enable it:
```
QSW-M2116P(config)# ip routing
QSW-M2116P(config)# ip route ... # Add static routes
```

## Shell access

To obtain a root shell, from the console port (`platform` command doesn't seem to be available over SSH):
```
QSW-M2116P# platform debug allow

WARNING: The use of 'debug' commands may negatively impact system behavior.
Do not enable unless instructed to. (Use 'platform debug deny' to disable
debug commands.)

NOTE: 'debug' command syntax, semantics and behavior are subject to change
without notice.

QSW-M2116P# debug system shell
/ #
```

### System overview

```
/ # whoami
root

/ # uname -a
Linux (none) 5.15.25 #27 Thu Sep 15 06:55:52 GMT 2022 mips GNU/Linux

/ # free -m
              total        used        free      shared  buff/cache   available
Mem:            499          63         383           0          53         427
Swap:             0           0           0

/ # df -h
Filesystem                Size      Used Available Use% Mounted on
devtmpfs                249.2M         0    249.2M   0% /dev
ubi0:switch              19.9M     72.0K     18.8M   0% /switch
overlay                 640.0K    640.0K         0 100% /

/ # cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00040000 00010000 "RedBoot"
mtd1: 00010000 00010000 "conf"
mtd2: 01400000 00010000 "linux"    # current firwmare image
mtd3: 01400000 00010000 "linux.xx" # previous firmware image (?)
mtd4: 01780000 00010000 "rootfs_data"
mtd5: 00010000 00010000 "FIS_directory"
mtd6: 00010000 00010000 "RedBoot_config"
mtd7: 00010000 00010000 "Redundant_FIS"
mtd8: 00a00000 00010000 "split_linux_lo"
mtd9: 00a00000 00010000 "split_linux_hi"

/ # cat /proc/cpuinfo
system type             : Jaguar2 Cu8-Sfp16 PCB110 Reference Board
machine                 : Jaguar2 Cu8-Sfp16 PCB110 Reference Board
processor               : 0
cpu model               : MIPS 24KEc V5.4
BogoMIPS                : 332.54
wait instruction        : yes
microsecond timers      : yes
tlb_entries             : 16
extra interrupt vector  : yes
hardware watchpoint     : yes, count: 4, address/irw mask: [0x0ffc, 0x0ffc, 0x0ffb, 0x0ffb]
isa                     : mips1 mips2 mips32r1 mips32r2
ASEs implemented        : mips16 dsp
shadow register sets    : 1
kscratch registers      : 0
package                 : 0
core                    : 0
VCED exceptions         : not available
VCEI exceptions         : not available

/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: vtss.ifh: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 10400 qdisc pfifo_fast qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
3: sit0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
4: vtss.vlan.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link
       valid_lft forever preferred_lft forever
5: vtss.vlan.15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.15.242/24 brd 192.168.15.255 scope global vtss.vlan.15
       valid_lft forever preferred_lft forever
    inet6 fde9:32e2:15b3:xxxx:xxxx:xxxx:xxxx:xxxx/64 scope global dynamic flags 100
       valid_lft 2591996sec preferred_lft 604796sec
    inet6 2a01:cb08:9155:xxxx:xxxx:xxxx:xxxx:xxxx/64 scope global dynamic flags 100
       valid_lft 2591996sec preferred_lft 604796sec
    inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link
       valid_lft forever preferred_lft forever

/ # ps aux
PID   USER     COMMAND
    1 root     /usr/bin/stage2-loader
    2 root     [kthreadd]
    3 root     [kworker/0:0-eve]
    4 root     [kworker/0:0H-kb]
    5 root     [kworker/u2:0-ev]
    6 root     [mm_percpu_wq]
    7 root     [ksoftirqd/0]
    8 root     [kdevtmpfs]
    9 root     [inet_frag_wq]
   10 root     [oom_reaper]
   11 root     [writeback]
   12 root     [kcompactd0]
   13 root     [kblockd]
   14 root     [kswapd0]
   15 root     [kworker/0:1-eve]
   17 root     [spi0]
   18 root     [mld]
   19 root     [ipv6_addrconf]
   21 root     [ubi_bgt0d]
   22 root     [ubifs_bgt0_0]
   23 root     [kworker/0:1H-kb]
   43 root     /usr/bin/switch_app
  113 root     /usr/sbin/zebra -f /etc/quagga/zebra.conf -i /tmp/zebra.pid -P 0
  114 root     /usr/sbin/staticd -f /tmp/staticd.conf -i /tmp/staticd.pid -P 0
  119 root     /usr/sbin/dropbear -r /switch//dropbear_rsa_host_key -p 22 -j -k
  120 root     /usr/sbin/ntpd -g -n -L -c /tmp/ntp.conf -l /tmp/ntp.log
  125 root     {qnssweb.sh} /bin/sh /etc/init.d/qnssweb.sh
  127 root     /usr/bin/qnss-web
  133 nobody   hiawatha -d -c /tmp/hiawatha
  355 root     [kworker/u2:1-ev]
  357 root     /bin/sh
  361 root     ps aux

/ # ls -lh /switch
total 596K
drwx------    2 root     root         160 Jan  1  1970 cert_indexes
-rw-------    1 root     root         805 Jan  1  1970 dropbear_rsa_host_key
-rw-r--r--    1 root     root        1.7K Mar  4 16:06 efcfg.json_bk
drwxr-xr-x    2 root     root         592 Mar 16 10:36 icfg
drwxr-xr-x    3 root     root         304 Mar  4 16:11 qnss_web
---------T    1 root     root         512 Mar 16 10:36 random-seed
-rw-r--r--    1 root     root      512.0K Mar 16 10:36 stackconf
-rw-r--r--    1 root     root           0 Jan  1  1970 startup-config-created
-rw-r--r--    1 root     root       64.0K Mar 16 10:25 syslog
-rwxr-xr-x    1 root     root          82 Nov  4  2020 tget.sh
-rw-------    1 root     root         675 Mar 16 10:37 vtss_snmpd.conf

/ # ls -lh /switch/icfg
total 68K
-rw-r--r--    1 root     root       47.2K Mar 16 10:25 crashfile
lrwxrwxrwx    1 root     root          31 Mar 16 10:36 default-config -> /switch/icfg/new-default-config
-rw-r--r--    1 root     root          12 May 26  2021 macaddr
-rw-r--r--    1 root     root        3.5K Mar 16 10:36 new-default-config
-rw-r--r--    1 root     root        6.8K Mar 15 17:15 startup-config
-rw-r--r--    1 root     root          61 Mar  6 20:27 tzlocaion
```

## Extracting the firmware

The firmware can be extracted with [binwalk](https://github.com/ReFirmLabs/binwalk):
```bash
docker run --rm -it -v $(pwd):/data binwalk --extract --matryoshka --directory /data/extractions /data/QSW-M2116P-2.0.1.32808.img
```

The list of the extracted files is provided in [`QSW-M2116P-2.0.1.32808.img.extracted.txt`](/QSW-M2116P-2.0.1.32808.img.extracted.txt).

## Todo

- IP routing testing
- Allow direct /bin/sh access via SSH instead of icli (shell hardcoded in dropbear binary?)
- sflow

### Dynamic routing

It would be really cool if the switch supported a routing protocol like OSPF. According the [IStaX specification](https://ww1.microchip.com/downloads/aemDocuments/documents/UNG/ProductDocuments/UserGuides/IStax_software_product_specification_30010225.pdf) OSPF is supported by the OS for this switch chip, but the feature doesn't seem to be enabled when building the OS.

In absence of the sources, we can still try to cross-compile Quagga/ospfd and run it on the switch. The switch chip appears to provide a [switchdev](https://docs.kernel.org/networking/switchdev.html) driver for Linux, but it's unclear if just adding routes to the kernel will automatically configure L3 offload (as opposed to adding routed via `icli`).

### Fan control

Can use QNAP `hal_app`, similar to QNAP NASes, from https://www.reddit.com/r/qnap/comments/oq2r8n/qnap_qm2_loud_fan_noise_solved/:
```
# RPM
/ # hal_app --se_sys_get_fan enc_sys_id=root,obj_index=0
enc_sys_get_fan(2631):fan index = 0,ret = 0,fan = 5212 rpm,fan_fail = 0
/ # hal_app --se_sys_get_fan enc_sys_id=root,obj_index=1
enc_sys_get_fan(2631):fan index = 1,ret = 0,fan = 5335 rpm,fan_fail = 0

# PWM
/ # hal_app --se_sys_get_fan_pwm enc_sys_id=root,obj_index=0
enc_sys_get_fan_pwm(2689):fan index = 0,ret = 0,fan_pwm = 159
/ # hal_app --se_sys_get_fan_pwm enc_sys_id=root,obj_index=1
enc_sys_get_fan_pwm(2689):fan index = 1,ret = 0,fan_pwm = 159
```

Can we do the same via the Microchip APIs?

### QEMU userland emulation

```bash
apt install binfmt-support qemu-user-static
docker run --rm -it -v $(pwd):/data binwalk -x dtb --carve --extract --directory /data/extractions /data/QSW-M2116P-2.0.1.32808.img
mkdir -p /data/mnt/squashfs1 /data/mnt/squashfs2 /data/mnt/squashfs3 /data/mnt/overlay/upper /data/mnt/overlay/work /data/mnt/overlay/merged
unsquashfs -d /data/mnt/squashfs1 extractions/QSW-M2116P-2.0.1.32808.img_20270812_squashfs.raw
unsquashfs -d /data/mnt/squashfs2 extractions/QSW-M2116P-2.0.1.32808.img_2275820_squashfs.raw
unsquashfs -d /data/mnt/squashfs3 extractions/QSW-M2116P-2.0.1.32808.img_2584041_squashfs.raw
unsquashfs -d /data/mnt/squashfs4 extractions/QSW-M2116P-2.0.1.32808.img_8105575_squashfs.raw
cp $(which qemu-mipsel-static) /data/mnt/squashfs2/$(which qemu-mipsel-static)
mount -t overlay overlay -o lowerdir=/data/mnt/squashfs1:/data/mnt/squashfs2:/data/mnt/squashfs3:/data/mnt/squashfs4,upperdir=/data/mnt/overlay/upper,workdir=/data/mnt/overlay/work /data/mnt/overlay/merged
chroot /data/mnt/overlay/merged/ qemu-mipsel-static /usr/bin/switch_app
```

```
12:36:43 Starting application...
Unable to open I2C bus 0: No such file or directory
Using existing mount point for /switch/
URANDOM: Failed to open /dev/urandom: No such file or directory
W conf 12:36:43 15400/conf_board_start#490: Warning: MAC address not set, using random: 02:00:C1:33:44:55
W conf 12:36:43 15400/conf_sec_read#894: Warning: illegal cookie[0] 0x00000000
W conf 12:36:43 15400/conf_sec_read#894: Warning: illegal cookie[1] 0x00000000
Starting hal application...
/sys/class/uio: No such file or directory
corrupted size vs. prev_size while consolidating
```

### MFI (Modular Firmware Image)

See https://www.microchip.com/content/dam/mchp/documents/UNG/ApplicationNotes/ApplicationNotes/AN1163-WebstaX-Customization.pdf.

```bash
hexdump QSW-M2116P-2.0.1.32808.img | grep 'd5de edd4'
# 0001000 d5de edd4 4c4d 987b 0002 0000 005c 0000 # 0001000 => 4096
dd if=QSW-M2116P-2.0.1.32808.img ibs=4096 skip=1 of=mfi.img
./mesa/mesa/demo/mfi.rb --dump --input mfi.img
```

```
Stage1
  Version:2
  Magic1:0xedd4d5de, Magic2:0x987b4c4d, HdrLen:92, ImgLen:2486100
  Machine:jaguar2c, SocName:jaguar2, SocNo:7, SigType:1
  Tlv Type:Kernel(0), Data Length:2271572
  Tlv Signature(1),   Data Length:16 (validated)
  Tlv Initrd(2),      Data Length:204800
  Tlv KernelCmd(3),   Data Length:48
  Tlv Metadata(4),    Data Length:82
  Tlv License(5),     Data Length:9396
Stage2 - Index:0
  Tlv FsElement(2), Data Length:5615287
  MD5(1), Length:16 Data: fb0d3791eeaa110f58c29c1a839369e2 (validated)
    Name:                           rootfs
    Version:                        2022.03
    License:                        Found
    Content file name:              wd-istax_sparxIV_80_24-jaguar2_pcb110.squashfs
    Content (squashfs) file length: 5521408
Stage2 - Index:1
  Tlv FsElement(2), Data Length:12165194
  MD5(1), Length:16 Data: a7ed34129a32ace697db183517193841 (validated)
    Name:                           vtss
    Version:                        79525bf+
    Content file name:              istax_sparxIV_80_24.app-rootfs
    Content (squashfs) file length: 12165120
Stage2 - Index:2
  Tlv FsElement(2), Data Length:589899
  MD5(1), Length:16 Data: 3976f44303cb0e43fffd66c109c232f0 (validated)
    Name:                           vtss-web-ui
    Version:                        79525bf+
    Content file name:              vtss-www-rootfs.squashfs
    Content (squashfs) file length: 589824
```

### Flash layout

- https://github.com/vtss/flash_builder
- https://ww1.microchip.com/downloads/aemDocuments/documents/ENT/ApplicationNotes/ApplicationNotes/ENT-AN1144-4.00-VPPD-04306.pdf
- https://ww1.microchip.com/downloads/secure/aemDocuments/documents/UNG/ApplicationNotes/ApplicationNotes/AN1289-SW_Configuration_Guide_Flash.pdf

## Related works

It's inspired by the following posts, but they target different QNAP switches which run on OpenWRT, whereas the M2116P runs on another distribution ([Microchip IStaX](https://www.microchip.com/en-us/product/vsc6817)):
- https://github.com/marcan/qsw-tools
- https://stevetech.me/posts/qnap-switch-serial-console
- https://www.reddit.com/r/homelab/comments/p4czkr/exploring_hidden_features_of_qnap_qswm21082s/

