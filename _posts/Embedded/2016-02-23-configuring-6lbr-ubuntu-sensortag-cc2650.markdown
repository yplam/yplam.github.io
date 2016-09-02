---
title:  "Configuring 6lbr for Ubuntu and SensorTag CC2650"
categories: Embedded
---

This post show you how to configure Ubuntu as 6LoWPAN router and bridge 6LoWPAN devices to the IPv4/IPv6 Internet.

**Slip-radio**

Because I do not have a CC2531 Dongle around, I use a SensorTag CC2650 with DevPack as Slip-radio.

First, change dir to contiki/examples/ipv6/slip-radio, run command:

```
make TARGET=srf06-cc26xx BOARD=sensortag/cc2650
```

Then flash slip-radio.bin to SensorTag using uniflash.

You need to validate that slip radio is running:

```
sudo minicom -s
``` 

Config minicom to use /dev/ttyACM0 with 115200 8N1. Your will see message like this after resetting SensorTag:

```
Starting Contiki-3.x-2102-g9f1376d
With DriverLib v0.44336                                                   
TI CC2650 SensorTag                                                       
 Net: slipnet                                                                
 MAC: nullmac                                                                
 RDC: ContikiMAC, Channel Check Interval: 16 ticks                           
 RF: Channel 25                                                              
Slip Radio started...
```
 
**1) Edge Router**

Follow the instructions on the following link to compile and install the 6lbr router: [https://github.com/cetic/6lbr/wiki/Other-Linux-Software-Configuration](https://github.com/cetic/6lbr/wiki/Other-Linux-Software-Configuration)

Edit /etc/6lbr/6lbr.conf as follow:

```
MODE=ROUTER
RAW_ETH=0
DEV_TAP=tap0

BRIDGE=1
CREATE_BRIDGE=6LBR
DEV_BRIDGE=eth0
ETH_JOIN_BRIDGE=0

DEV_RADIO=/dev/ttyACM0
BAUDRATE=115200

```

More about configure options: [https://github.com/cetic/6lbr/wiki/6LBR-Configuration-file](https://github.com/cetic/6lbr/wiki/6LBR-Configuration-file) 

Start 6lbr:

```
sudo service 6lbr start
```

Config 6LBR to access WSN network:

```
sudo route -A inet6 add aaaa::/64 gw bbbb::100
```

Now open [http://[bbbb::100]](http://[bbbb::100]/) in your browser and you can access the embedded web server.



**2) Use Smart Bridge to access localhost MQTT Server**

Edit /etc/6lbr/6lbr.conf as follow:

```
MODE=ROUTER
RAW_ETH=0
DEV_TAP=tap0

BRIDGE=1
CREATE_BRIDGE=6LBR
DEV_BRIDGE=eth0
ETH_JOIN_BRIDGE=0

DEV_RADIO=/dev/ttyACM0
BAUDRATE=115200

```

Add /etc/6lbr/ifup.d/60dev

```
#!/bin/bash
. $CETIC_6LBR_CONF
. $1/6lbr-functions
config_default
MODE_6LBR=$2
DEV=$3
OS=`uname`

ip -6 address add 2001:db8:2::2/64 dev tap0
sysctl -w net.ipv4.conf.all.forwarding=1
sysctl -w net.ipv6.conf.all.forwarding=1

```

Add /etc/6lbr/ifdown.d/60dev

```
#!/bin/bash
. $CETIC_6LBR_CONF
. $1/6lbr-functions
config_default
MODE_6LBR=$2
DEV=$3
OS=`uname`
sysctl -w net.ipv4.conf.all.forwarding=0
sysctl -w net.ipv6.conf.all.forwarding=0
```

You alse need to run:

```
chmod +x /etc/6lbr/ifup.d/60dev
chmod +x /etc/6lbr/ifdown.d/60dev
```

Install and configure radvd:

```
interface tap0
{
   AdvSendAdvert on;
   AdvManagedFlag off;     #stateless autoconfiguration
   AdvOtherConfigFlag on;  #clients get extra parameters via DHCPv6
   MaxRtrAdvInterval 10;   #resend RA @ random times, max 10sec delay
   prefix 2001:db8:2::/64  #announce prefix to clients
   {
       AdvOnLink on;
       AdvAutonomous on;
       AdvRouterAddr on;
   };
   RDNSS 2001:db8:2::2
   {
   };
};
```

Start 6lbr and radvd, you can access router machine var 2001:db8:2::2 from a WSN node. And your WSN node will get an ipv6 address like 2001:db8:2:****:****:****:****:**** from radvd.

**3) MQTT over IPV4 WAN**

I only have IPV4 WAN access, and have a mosquitto server running on my Linode server. So I need a NAT64 tool to bridge ipv6 to ipv4 network.

After trying jool with no luck, I decided to use 6tunnel to forward ipv6 traffic.

```
sudo apt-get install 6tunnel
6tunnel -6 1883 your.mqtt.server.ip 1883
```

That's all!

See also:

[http://processors.wiki.ti.com/index.php/Contiki-6LOWPAN-BBB](http://processors.wiki.ti.com/index.php/Contiki-6LOWPAN-BBB)


