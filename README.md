# QSW-M2116P-2T2S

The QNAP [QSW-M2116P-2T2S](https://www.qnap.com/en-us/product/qsw-m2116p-2t2s) is a PoE switch with 16 2.5GbE ports, 2 10GbE ports (also support 1/2.5/5G) and 2 SFP+ ports.

It is advertised as a layer 2 web-managed switch, but as shown in this document, it can also do layer 3 routing, be managed with a Cisco IOS-like CLI and it is trivial to obtain a root Linux shell.

This document is based on the latest [firmware](https://www.qnap.com/en/download?model=qsw-m2116p-2t2s&category=firmware) as of March 2025: 2.0.1.32808.

It's inspired by the following posts, but they target different QNAP switches which run on OpenWRT, whereas the M2116P runs on another distribution ([Microchip IStaX](https://www.microchip.com/en-us/product/vsc6817)):
- https://github.com/marcan/qsw-tools
- https://stevetech.me/posts/qnap-switch-serial-console
- https://www.reddit.com/r/homelab/comments/p4czkr/exploring_hidden_features_of_qnap_qswm21082s/

## Datasheet

The switch uses a [Microchip VSC7448-02 SparX-IV-80](https://www.microchip.com/en-us/product/vsc7448) switch chip. In addition to standard switching features, the [datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/SparX-IV_L2_L3_Enterprise_Gigabit_Ethernet_Switches_Datasheet_00004426A.pdf) tells us that the chip supports IP routing with up to 4k IPv4 routes and 1k IPv6 routes.

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

The configuration interface seems to only support static routes, but it should be possible to run an OSPF daemon (e.g. Quagga/ospfd) on the switch.

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
```

## Extracting the firmware

The firmware can be extracted with [binwalk](https://github.com/ReFirmLabs/binwalk):
```bash
docker run --rm -it -v $(pwd):/data binwalk --extract --matryoshka --directory /data/extractions /data/QSW-M2116P-2.0.1.32808.img
```

The list of the extracted files is provided in [`QSW-M2116P-2.0.1.32808.img.extracted.txt`](/QSW-M2116P-2.0.1.32808.img.extracted.txt).

## Todo

- IP routing testing
- Cross-compile Quagga/ospfd for the switch and try to run OSPF
- Customize the firmware (without access to the source?)
  - https://ww1.microchip.com/downloads/aemDocuments/documents/UNG/ProductDocuments/UserGuides/UG1068-SW-Introduction-to-WebStaX-on-Linux.pdf
- Allow direct /bin/sh access via SSH instead of icli (shell hardcoded in dropbear binary?)
- sflow

