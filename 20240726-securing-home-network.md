# Securing home network

## Intro

It's been a long time since Raspberry Pi pioneered SBCs for masses.
and nowadays there's a lot of amazing boards on the market.
But there's still no easy way to build an inexpensive GPON/WiFi router.
The industry is trying to make it extremely difficult to build such a thing.
On Armbian's download page I see only a couple of Clearfrog boards
that have SFP. But they have no WiFi and they aren't budget.
Of course there are budget routers that can run OpenWRT
but the hardware is so shitty that OpenWRT is the only choice
for them.

That's why most people have to install a router provided by their ISP.
ISPs, after all those apples and microsofts, have a powerful urge
to break into our homes too.

Another side of the problem is tech support.
If 20 years ago you called ISP and said something like
"as I observe from my logs, there's no response to LCP packets from you"
they easily understood that and took immediate actions.
That's because you talked with admins.
Now you'll talk with someone as my ex-wife who's happily unaware
of the existence of IP protocol.
"So what's the problem?" you may ask.
Well, ISPs are affiliated with TV streaming platforms and nobody will
fix any problem on their side if there's a custom
router between ISP cable and your SmartTV.
Even if it works like a charm and has nothing to do with their problems.

But all that is little things.

Recently my wife called support to fix video artifacts on the TV
and they wondered "Wow, how many computers in your home!"
The router displays active devices probably by inspecting local traffic
from higher level down to the ARP protocol.

I felt myself like a suspect. Or a potential victim.
They would understand nothing if I told them that there are simply containers
on my three SBCs.
But they are pretty sure there's a lot of valuable hardware in the suite 13,
666 Mazafaka street Horsedick city.

## Requirements

So, ISP's router is a hostile device in my home but I have to keep it in.
But why can I be so careless? I need a safe home network.

Of course such shit as smart TVs, windows machines and smartphones
should be connected to the ISP's router directly.
In most cases they use WiFi so they are at the right side.

But all the rest should reside behind an isolating router.
Preferably with the same internal subnet as the outer one just because
I don't want to reconfigure my static IP if I connect my laptop
to either LAN.

Here's what I want:
```
ISP --- [Real IP address]
            Router
         192.168.0.1 --- untrusted LAN  -- TV, smartphones
                         192.168.0.0/24
                              |
                          192.168.0.2
                        isolating router
                          192.168.0.1
                              |
        my laptop          trusted LAN
      192.168.0.10  ---  192.168.0.0/24  --- other computers
```
See what I mean? Both trusted and untrusted LANs use the same address space
but they don't share it.
Because of this, devices from the trusted LAN cannot directly communicate with
untrusted devices.
This is just one of security measures.
But if I plug my laptop to either LAN I don't have to reconfigure its static
IP address and the default gateway.

But there's one problem. What if I want to connect to my trusted LAN via WiFi?
For example, I might want to control some of my IoT devices from smartphone.
Wireguard could be the simplest solution: I tap on home VPN and get in.
But I won't be able to connect to the trusted LAN directly and that's very good.
For WiFi, I woudld need a separate subnet, let's call it IoT subnet.

Also, I want  a couple of VPNs in the trusted LAN.
They should be exposed as another gateways, say 192.168.0.2 and 192.168.0.3.
I may want a Tor service, exposing its SOCKS5 port.
And I want a DNS resolver working via VPN.

And the last point: plausible deniability matters.
If shit ever happen and they seize my hardware, I don't want they know
I used VPN and Tor.
Of course I can't hide VPN traffic, which, however, may look like SSH or HTTP,
but I can hide the originator.
After power on, there should be no evidence of VPN and Tor on the router.

## Implementation

veth naming:
* first digit: LXC container
* second digit: interface
* third digit: disposition
  0. inside container
  1. outside container

For example, veth321 is an interface number 2 paired with veth320 in container 3

Router structure is quite simple:
```
                      Untrusted LAN
                      192.168.0.0/24
                default gateway: 192.168.0.1
                            |
                          eth0
                       192.168.0.2
                      gw 192.168.0.1
                            |
        +-------------------+-------------------+------------+
        |                   |                   |            |
       NAT                 NAT                 NAT           |
        |                   |                   |            |
      veth101             veth201             veth301        |
   192.168.240.1       192.168.242.1       192.168.244.1     |
        |                   |                   |            |
        |                   |                   |     +------+-netns iot-+
        |                   |                   |     |      |           |
        |                   |                   |     |     wg0          |
        |                   |                   |     |  192.168.1.1     |
        |                   |                   |     |                  |
        |               +-----------------------------+-veth111          |
        |               |   |                   |     | DNS DNAT         |
+-lxc1--+---------------+--------+              |     |                  |
|       |               |        |          +---------+-veth211          |
|     veth100         veth110    |          |   |     |                  |
| 192.168.240.2   192.168.241.2  |          |   |     |    IoT subnet    |
| gateway:        route to:      |          |   |     |  192.168.1.0/24  |
| 192.168.240.1   192.168.1.0/24 |          |   |     |                  |
|          \      /              |          |   |     |       veth311    |
|          NAT   NAT             |          |   |     |         |        |
|            \  /                |          |   |     +---------+--------+
|  DNS ---- veth120              |          |   |               |
|          192.168.0.1           |          |   |               |
|        default gateway         |          |   |               |
|              |                 |          |   |               |
+--------------+-----------------+          |   |               |
               |            |               |   |               |
               |    +-lxc2--+---------------+--------+          |
               |    |       |               |        |          |
               |    |     veth200         veth210    |          |
               |    | 192.168.242.2   192.168.243.2  |          |
               |    | gateway:        route to:      |          |
               |    | 192.168.242.1   192.168.1.0/24 |          |
               |    |          \       /             |          |
               |    |          VPN   NAT             |          |
               |    |            \   /               |          |
               |    |           veth220              |          |
               |    |         192.168.0.2            |          |
               |    |        VPN-1 gateway           |          |
               |    |              |                 |          |
               |    +--------------+-----------------+          |
               |                   |            |               |
               |                   |    +-lxc3--+---------------+--------+
               |                   |    |       |               |        |
               |                   |    |     veth300         veth310    |
               |                   |    | 192.168.244.2   192.168.245.2  |
               |                   |    | gateway:        route to:      |
               |                   |    | 192.168.244.1   192.168.1.0/24 |
               |                   |    |          \       /             |
               |                   |    |          VPN   NAT             |
               |                   |    |            \   /               |
               |                   |    |           veth320              |
               |                   |    |         192.168.0.3            |
               |                   |    |        VPN-2 gateway           |
               |                   |    |              |                 |
               |     veth_ssh      |    +--------------+-----------------+
               |   192.168.254.1   |                   |
               |         |         |                   |
   +-----------+---------+---------+-------------------+------------+
   |           |         |         |                   |            |
   |           |    veth_ssh_int   |                   |            |
   |           |   192.168.254.2   |                   |            |
   |           |         |         |                   |            |
   |           |      SSH DNAT     |                   |            |
   |           |         |         |                   |            |
   |           |    veth_ssh_ext   |                   |            |
   |           |    192.168.0.2    |                   |            |
   |           |         |         |                   |            |
   |    +-br0--+---------+---------+-------------------+-------+    |
   |    |      |         |         |                   |       |    |
   |    |    veth121 veth_ssh_br veth221             veth321   |    |
   |    |      |         |         |                   |       |    |
   |    |      +---------+---------+-------------------+       |    |
   |    |                          |                           |    |
   |    |                         eth1                         |    |
   |    |                          |                           |    |
   |    +--------------------------+---------------------------+    |
   |                               |                                |
   +-netns trusted_lan-------------+--------------------------------+
                                   |
                               Trusted LAN
                              192.168.0.0/24
```

Normally SSH should not be exposed to the untrusted LAN.
But it should be accessible from the trusted one at the same
address 192.168.0.2.
This is what `veth_ssh*` chain is for, with SSH DNAT in the middle.
SSH traffic is forwarded to the internal IP address 192.168.254.1
in the root namespace

The `trusted_lan` namespace isolates SSH DNAT which includes same IP
address as in the root namespace.

Without plausible deniability in mind, all the rest could also be
implemented with network namespaces only, without LXC.
But LXC containers are fucking good for PD.
In addition, all they are unprivileged and all services running
inside are perfectly isolated.

Initially `lxc2` and `lxc3` are hidden and not running after boot.
DNS resolver in `lxc1` does not use VPN.
But when the system is bootstrapped, VPN containers get up and `lxc1`
is replaced with a different version.
How to do that is outside of the scope of this post.

IoT subnet dictates the need for vethX2X pairs in each container.
That's for ease of client configuration where only default route is needed,
no matter which one.

A separate namespace `iot` isolates that subnet.
For installing updates on isolated devices, I'll provide access to my
apt-cacher from the trusted LAN by adding DNAT rules.

Let's prepare an [Armvuan](https://github.com/amateur80lvl/armvuan)-powered
pie and install necessary packages.

Utilities and essentials:
```bash
apt install tcpdump unzip wireguard-tools ca-certificates
```

[wgman](https://github.com/amateur80lvl/wgman) dependencies:
```bash
apt install python3-yaml python3-qrcode
```

LXC:
```bash
apt install cgroupfs-mount debootstrap distro-info lxc lxcfs \
    lxc-templates libpam-cgfs libvirt0 uidmap
```

Make changes to `/etc/sysctl.conf` (I don't need IPv6):
```
net.ipv4.ip_forward=1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

Given that the router is a headless system, let ifupdown manages
the untrusted LAN interface `eth0`.
So we avoid touching `/etc/network/interfaces` to keep
the basic networking unbroken and always up.
But for all the rest let's create an init script:
`/etc/init.d/advanced-networking`:
```bash
#!/bin/bash
### BEGIN INIT INFO
# Provides:          advanced-networking
# Required-Start:    $local_fs
# Required-Stop:     $local_fs
# Should-Start:      $network
# Default-Start:     S
# Default-Stop:      0 6
# Short-Description: Advanced networking configuration.
### END INIT INFO

# subnet for both trusted and untrusted LAN
LAN_SUBNET=192.168.0.0/24

UNTRUSTED_INTERFACE=eth0
UNTRUSTED_IP_ADDRESS=192.168.0.2/24

TRUSTED_INTERFACE=eth1

# SSH public address is the same as for untrusted LAN interface
SSH_PUBLIC_ADDRESS=$UNTRUSTED_IP_ADDRESS
SSH_PRIVATE_ADDRESS=192.168.254.1/24
SSH_DNAT_INTERNAL_ADDRESS=192.168.254.2/24

IOT_WG_NET=192.168.1.0/24
IOT_WG_ADDRESS=192.168.1.1/32


. /lib/lsb/init-functions

create_interfaces()
{
    # create and configure namespaces
    for ns in trusted_lan iot ; do
        ip netns add $ns
        # bring loopback up to make ping working
        ip -n $ns link set lo up
        # set networking params same as in the root namespace
        ip netns exec $ns sysctl --pattern '^net' --system >/dev/null
    done

    # create bridge
    ip -n trusted_lan link add name br0 type bridge

    # create veth interfaces for SSH DNAT
    ip -n trusted_lan link add dev veth_ssh_ext type veth peer name veth_ssh_br
    ip link add dev veth_ssh type veth peer name veth_ssh_int netns trusted_lan

    # create IoT wireguard interface
    ip link add wg0 type wireguard
    ip link set wg0 netns iot
    ip -n iot address add $IOT_WG_ADDRESS dev wg0
    ip netns exec iot wg setconf wg0 <(wg-quick strip wg0)
    ip -n iot link set wg0 up
    ip -n iot route add $IOT_WG_NET dev wg0
}

delete_interfaces()
{
    set +e

    ip -n iot link del wg0

    # for veth, it's sufficient to delete one to delete entire pair
    ip link del dev veth_ssh
    ip -n trusted_lan link del dev veth_ssh_br
    ip -n trusted_lan link del dev br0

    # delete namespaces
    ip netns del iot
    ip netns del trusted_lan
}

advanced_networking_is_up()
{
    if [ -e /run/netns/trusted_lan ] ; then
        return 0
    else
        return 1
    fi
}

configure_networking()
{
    # make up the bridge
    ip link set $TRUSTED_INTERFACE netns trusted_lan
    ip -n trusted_lan link set $TRUSTED_INTERFACE master br0
    ip -n trusted_lan link set $TRUSTED_INTERFACE up

    # setup SSH public veth pair
    ip -n trusted_lan address add $SSH_PUBLIC_ADDRESS dev veth_ssh_ext
    ip -n trusted_lan link set veth_ssh_ext up
    ip -n trusted_lan link set veth_ssh_br master br0
    ip -n trusted_lan link set veth_ssh_br up

    # setup SSH private veth pair
    ip -n trusted_lan address add $SSH_DNAT_INTERNAL_ADDRESS dev veth_ssh_int
    ip -n trusted_lan link set veth_ssh_int up
    ip address add $SSH_PRIVATE_ADDRESS dev veth_ssh
    ip link set veth_ssh up

    # bring the bridge up
    ip -n trusted_lan link set br0 up
}

create_container_interfaces()
{
    VETH=$1
    ADDR=$2
    IOT_ADDR=$3

    # create veth pairs for LXC container
    ip link add dev "${VETH}00" type veth peer name "${VETH}01"
    ip link add dev "${VETH}10" type veth peer name "${VETH}11" netns iot
    ip link add dev "${VETH}20" type veth peer name "${VETH}21" netns trusted_lan

    # connect one end to the bridge
    ip -n trusted_lan link set "${VETH}21" master br0
    ip -n trusted_lan link set "${VETH}21" up

    # configure untrusted lan end
    ip address add $ADDR dev "${VETH}01"
    ip link set "${VETH}01" up

    # configure iot end
    ip -n iot address add $IOT_ADDR dev "${VETH}11"
    ip -n iot link set "${VETH}11" up
}

add_container_routes()
{
    # This procedure is executed in container's namespace.
    # things would be simpler if LXC container config had a route
    # parameter but they don't want to implement that
    # https://github.com/lxc/lxc/issues/496
    INTERFACE=$1

    # a route to IoT subnet
    ip route add $IOT_WG_NET dev $INTERFACE
}

delete_container_interfaces()
{
    VETH=$1

    ip link del dev "${VETH}00"
    ip link del dev "${VETH}10"
    ip link del dev "${VETH}20"
}

do_start()
{
    if advanced_networking_is_up ; then
        return 0
    fi

    log_action_begin_msg "Initializing advanced networking"
    set -e
    trap do_start_failed EXIT

    create_interfaces
    configure_networking

    # reload nftables
    /etc/nftables.conf
    ip netns exec trusted_lan /etc/nftables-trusted.conf
    ip netns exec iot /etc/nftables-iot.conf

    trap '' EXIT
    log_action_end_msg 0
}

do_start_failed()
{
    delete_interfaces
    log_action_end_msg 1
}

do_stop()
{
    log_action_begin_msg "Stopping advanced networking"
    delete_interfaces
    log_action_end_msg 0
}

ACTION=$1
shift
case $ACTION in
(start)
    do_start
    ;;
(stop)
    do_stop
    ;;
(restart|reload|force-reload)
    do_stop
    do_start
    ;;
(status)
    if advanced_networking_is_up ; then
        echo up
    else
        echo down
    fi
    ;;
(create-container-interfaces)
    set -e
    trap 'delete_container_interfaces "$@"' EXIT
    create_container_interfaces "$@"
    trap '' EXIT
    ;;
(add-container-routes)
    add_container_routes "$@"
    ;;
(delete-container-interfaces)
    delete_container_interfaces "$@"
    ;;
(*)
    echo >&2 "Usage: $0 {start|stop|restart|reload|force-reload|status}"
    exit 3
    ;;
esac
```

This script can also act as a pre-start and post-stop hook for LXC
containers to create veth interfaces on demand.

Let's install it:
```
update-rc.d advanced-networking defaults
```

Let's create the first LXC container:
```
mkdir -p /var/lib/lxc/lxc1/rootfs
debootstrap --variant=minbase --include dialog,libc-l10n,locales,nano \
    --exclude bootlogd,dmidecode,sysvinit-core,vim-common,vim-tiny \
    daedalus /var/lib/lxc/lxc1/rootfs http://deb.devuan.org/merged

chroot /var/lib/lxc/lxc1/rootfs

dpkg-reconfigure locales
apt update
apt install apt-utils bind9 iputils-ping iputils-tracepath iproute2 \
    less netbase nftables procps psutils psmisc runit runit-init tcpdump
```

setup runit in native boot mode:
```
touch /etc/runit/native.boot.run
touch /etc/runit/no.emulate.sysv
mkdir /etc/runit/boot-run
mkdir /etc/runit/shutdown-run
for f in /etc/service/getty* ; do unlink $f ; done
```

and take `/etc/runit/boot-run` scripts from
[lxcex](https://github.com/amateur80lvl/lxcex/tree/3f3a6b0e3473cd167be0222006b5866ce161bac6/containers/base/rootfs/etc/runit/boot-run)

finally, make changes to `/etc/nftables.conf`:
```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # accept any localhost traffic
        iif lo accept

        # accept ICMP traffic
        ip protocol icmp accept

        # accept DNS traffic
        tcp dport 53 accept
        udp dport 53 accept

        # accept traffic originated from us
        ct state established,related accept
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;

        oif { untrusted, iot } masquerade random,persistent
    }
}
```

Back to the host system, create /etc/subuid, /etc/subgid with the same content:
```
root:100000:65536
root:200000:65536
root:300000:65536
root:400000:65536
root:500000:65536
root:600000:65536
root:700000:65536
root:800000:65536
root:900000:65536
```

Create container configuration `/var/lib/lxc/lxc1/config`:
```
lxc.apparmor.profile = unconfined

# untrusted LAN
lxc.net.0.type = phys
lxc.net.0.name = untrusted
lxc.net.0.link = veth100
lxc.net.0.flags = up
lxc.net.0.ipv4.address = 192.168.240.2/24
lxc.net.0.ipv4.gateway = 192.168.240.1

# IoT
lxc.net.1.type = phys
lxc.net.1.name = iot
lxc.net.1.link = veth110
lxc.net.1.flags = up
lxc.net.1.ipv4.address = 192.168.241.2/24

# trusted LAN
lxc.net.2.type = phys
lxc.net.2.name = trusted
lxc.net.2.link = veth120
lxc.net.2.flags = up
lxc.net.2.ipv4.address = 192.168.0.1/24

lxc.hook.pre-start = /etc/init.d/advanced-networking create-container-interfaces veth1 192.168.240.1/24 192.168.241.1/24
lxc.hook.mount = /etc/init.d/advanced-networking add-container-routes iot
lxc.hook.post-stop = /etc/init.d/advanced-networking delete-container-interfaces veth1

# Common configuration
lxc.include = /usr/share/lxc/config/devuan.common.conf

# Container specific configuration
lxc.rootfs.path = dir:/var/lib/lxc/lxc1/rootfs
lxc.rootfs.options = idmap=container
lxc.uts.name = lxc1

lxc.include = /usr/share/lxc/config/devuan.userns.conf
lxc.idmap = u 0 100000 65536
lxc.idmap = g 0 100000 65536

lxc.start.auto = 1
```

Make changes to `/etc/nftables.conf` on the host system
(above we made changes to the container's one):
```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # accept any localhost traffic
        iif lo accept

        # accept ICMP traffic
        ip protocol icmp accept

        # accept SSH for initial configuration
        # normally it should be blocked
        iif eth0 tcp dport ssh accept

        # accept SSH on dedicated veth
        iif veth_ssh tcp dport ssh accept

        # accept IoT wireguard
        iif eth0 udp dport 10999 accept

        # accept traffic originated from us
        ct state established,related accept
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;

        oif eth0 masquerade random,persistent
    }
}
```

And create `/etc/nftables-trusted.conf`:
```
#!/usr/sbin/nft -f

flush ruleset

table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100; policy accept;

        # SSH DNAT
        iif veth_ssh_ext tcp dport 22 dnat to 192.168.254.1:22
    }
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;

        oif { veth_ssh_ext, veth_ssh_int }  masquerade random,persistent
    }
}
```

and  `/etc/nftables-iot.conf`:
```
#!/usr/sbin/nft -f

flush ruleset

table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100; policy accept;

        # DNS DNAT
        iifname veth111 udp dport 53 dnat to 192.168.241.2:53
        iifname veth111 tcp dport 53 dnat to 192.168.241.2:53
    }
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;

        masquerade random,persistent
    }
}
```

Let's setup wireguard interface.
I use my [wgman](https://github.com/amateur80lvl/wgman) suite.

Get [wgman-main.zip](https://github.com/amateur80lvl/wgman/archive/refs/heads/main.zip)
and unpack it to `/etc/wireguard/wgman`.

Then
```
cd /etc/wireguard/wgman
mkdir this-server
```

Create `/etc/wireguard/wgman/this-server/config.yaml`:
```
subnet: 192.168.1.0
subnet_bits: 24
server_private_address: 192.168.1.1
client_address_start: 2

server_public_address: 192.168.0.2
server_port: 10999

use_preshared_key: true

default_route: false
dns: 192.168.241.2
```

and run
```
./create-server this-server
chmod 600 this-server/wg0.conf
pushd ..
ln -s wgman/this-server/wg0.conf .
popd

```

Create configuration for a couple clients:
```
./create-client this-server test
./create-client this-server myiphone
```

If everything is done correctly it's okay to reboot.

Let's test. The following should work from the trusted network:
* ping 1.1.1.1
* ping 192.168.1.1
* ssh root@192.168.0.2

> [!NOTE]
> Either network stack or sshd need some kick to make SSH accessible
> from the trusted LAN. Immediately after boot it says Connection Refused,
> but approximately in 10 seconds after the first attempt SSH becomes
> available.
> * proxy ARP? -- tried to enable, no luck
> * any ideas?

Now, let's install DNS resolver.
```
lxc-attach lxc1
apt install bind9 dnsutils
```

Create `/etc/sv/named/run`
```bash
#!/bin/sh

exec 1>&2

if [ -e /etc/runit/verbose ]; then
    echo "invoke-run: starting ${PWD##*/}"
fi

[ -f /etc/default/named ] && . /etc/default/named
exec /usr/sbin/named -f $OPTIONS
```

Make changes to `/etc/bind/named.conf.options`
```
// disable IPv6 in addition to -4 option in /etc/defaults/named
server ::/0 {
    bogus yes;
};

options {
    directory "/var/cache/bind";

    dnssec-validation auto;

    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { none; };

    recursion yes;
    allow-recursion { any; };
    allow-query { any; };
    allow-query-cache { any; };

    max-cache-size 16M;

    response-policy {
        zone "rpz-home";
        zone "rpz-filter";
    };
};
```

Define RPZ zones in `/etc/bind/named.conf.local`:
```
zone "rpz-home" {
    type master;
    file "/etc/bind/rpz-home.db";
};

zone "rpz-filter" {
    type master;
    file "/etc/bind/rpz-filter.db";
};
```

`rpz-home` is responsible for resolving simple names in the home network.
RPZ is just a little bit more complicated than `hosts` file,
but the advantage is easy synchronization across
other DNS servers in the home network, if any.

Create `/etc/bind/rpz-home.db`
```
$TTL 1w
@ IN SOA localhost. root.localhost. (
         1   ; serial
         2w  ; refresh
         2w  ; retry
         2w  ; expiry
         2w) ; minimum
     IN NS localhost.

localhost     A   127.0.0.1
router        A   192.168.0.2
```

`rpz-filter` is generated from https://github.com/StevenBlack/hosts
by a script based on circuitshelter's one[^6].
To avoid installing yet another python in the container,
let's generate it on the host side:
```python
#!/usr/bin/env python3

from urllib import request

rpz_filename   = '/var/lib/lxc/lxc1/rootfs/etc/bind/rpz-filter.db'
hosts_file_url = 'https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts'

zone_header = """$TTL 2w
@ IN SOA localhost. root.localhost. (
       2   ; serial
       2w  ; refresh
       2w  ; retry
       2w  ; expiry
       2w) ; minimum
    IN NS localhost.

"""

def generate_rpz_file():
    num_hosts = 0
    with open(rpz_filename, 'w') as rpz_file:
        rpz_file.write(zone_header)

        with request.urlopen(hosts_file_url) as hosts_file:
            for line in hosts_file:
                line = line.decode('utf-8').strip()
                if line.startswith('0.0.0.0 '):
                    domain = line.split(' ')[1]
                    rpz_file.write(f'{domain} CNAME .\n')
                    rpz_file.write(f'*.{domain} CNAME .\n')
                num_hosts += 1

    print(f'Total hosts in filter: {num_hosts}')

if __name__ == '__main__':
    generate_rpz_file()
```

Okay to start name server now:
```
ln -s /etc/sv/named /etc/service/
```

As for VPN containers, every kid knows how to setup VPN.
Plausible deniability clues can be found
[here](https://github.com/amateur80lvl/lxcex/tree/main/book/ch7-plausible-deniability.md).

Bye for now.

## References

[^1]: [Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking)
[^2]: [Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/)
[^3]: [Routing & Network Namespaces](https://www.wireguard.com/netns/)
[^4]: [Netfilter hooks](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)
[^5]: https://github.com/StevenBlack/hosts
[^6]: [Using BIND 9 RPZ as DNS firewall](https://www.circuitshelter.com/posts/bind9-rpz-firewall/)
[^7]: [Overriding DNS for fun and profit](https://www.redpill-linpro.com/techblog/2015/12/08/dns-rpz.html)
