# Configuring 6RD for CenturyLink on Ubiquiti

CenturyLink still does not provide native IPv6, but you can configure
a 6RD tunnel. This configurations have been tested on their gigabit fiber
offering, but will probably work for any subscribers using a Ubiquiti Edge
Router.

This will work best if you have a static IP address because your 6RD prefix
is a function of your IPv4 address. I have scripted managing this change, but
have never quite been satisfied with its stability over the long run.

## Get a default firewall ready

We'll want to have this ready to attach to an interface in a moment, so
let's prep it now. This is a simple firewall that will permit ICMP and any
return traffic from outbound connections.

    set firewall ipv6-name internet6-in enable-default-log
    set firewall ipv6-name internet6-in rule 10 action accept
    set firewall ipv6-name internet6-in rule 10 description 'Allow established connections'
    set firewall ipv6-name internet6-in rule 10 log disable
    set firewall ipv6-name internet6-in rule 10 state established enable
    set firewall ipv6-name internet6-in rule 10 state related enable
    set firewall ipv6-name internet6-in rule 20 action drop
    set firewall ipv6-name internet6-in rule 20 log enable
    set firewall ipv6-name internet6-in rule 20 state invalid enable
    set firewall ipv6-name internet6-in rule 30 action accept
    set firewall ipv6-name internet6-in rule 30 log disable
    set firewall ipv6-name internet6-in rule 30 protocol icmpv6

## Permit proto 41 in your IPv4 firewall

Look for the firewall configuration attached to your WAN interface. Make a
rule that permits inbound proto 41 (ipv6) from CenturyLink:

    set firewall name internet-in rule 100 source address 205.171.2.64
    set firewall name internet-in rule 100 protocol 41
    set firewall name internet-in rule 100 action accept

I use the same ruleset for both `in` and `local`, but if you don't, ensure
these rules are applied to the `local` chain.

## Creating the tunnel interface

Now that the prerequisites are out of the way, we will begin by creating a
tunnel; we'll use `tun0`. Tunnels have transport configurations (the outer
header) and their own addresses (the inner header).  

    set interfaces tunnel tun0 description 'CenturyLink 6rd Tunnel'
    set interfaces tunnel tun0 encapsulation sit
    set interfaces tunnel tun0 firewall in ipv6-name internet6-in
    set interfaces tunnel tun0 firewall local ipv6-name internet6-in
    set interfaces tunnel tun0 local-ip 0.0.0.0
    set interfaces tunnel tun0 mtu 1472
    set interfaces tunnel tun0 multicast enable
    set interfaces tunnel tun0 remote-ip 205.171.2.64
    set interfaces tunnel tun0 ttl 255

We use `sit` to indicate this is an ipip tunnel. The firewall configurations
reference the newly created ruleset from above. 

We can set `local-ip` to the anonymous address (`0.0.0.0`), which allows the
kernel to automatically select the source address of outbound packets. If you
have a static ip, you can place it here instead. If you don't, leaving this
automatic is one less configuration parameter to script updates on.

The `mtu` setting accounts for the 20-byte IPv4 header and the 8-byte PPPoE
header. The `remote-ip` field is the CenturyLink tunnel endpoint.

## Determining your prefix

6RD uses your IPv4 address to fill in part of an IPv6 prefix. On CenturyLink,
2602::/24 is the supernet for 6RD. These 24 bits are followed by the 32 bits
of your IPv4 address, resulting in a 56-bit prefix usable on your home 
network. The only trick here is turning your base-10 IPv4 octets into
IPv6 hex nibbles. You can use this shell command to turn your IPv4 address
into your 6RD prefix:

    $ IP='198.51.100.78'
    $ printf "2602:%02x:%02x%02x:%02x00::/56\n" $(echo $IP | tr . ' ')
    2602:c6:3364:4e00::/56

In this example, the `00` represent 8 bits, `0x00 - 0xFF` that we can
use to number 256 unique /64 prefixes for our home network.

## Assign an address to the tunnel

We need to select one of our 2^72 addresses for our tunnel interface
(technically speaking, the tunnel can remain unnumbered, but it will be
easier to troubleshoot if we give it an address). I recommend assigning
from either the `2602:c6:3364:4e00::/64` or `2602:c6:3364:4eFF::/64` for
"infrastructure".

    set interfaces tunnel tun0 address 2602:c6:3364:4eFF::/128

This configuration uses the `FF` subnet number and the all zeroes host
number.

Many of the other guides available on this topic suggest using `/24` or
some other prefix length on the tunnel interface. These configurations don't
really make sense -- the `/24` is not directly attached to the tunnel
interface, and we don't want to send packets to unknown hosts from our own
prefix out to the CenturyLink tunnel server. Use a /128 on this interface.

## Define your routes

Because there is no "on net" IPv6 address on the other end of the tunnel,
we use an interface route to send traffic for other IPv6 networks to the
tunnel endpoint.

We will also pin up a blackhole route for our /56 prefix to ensure that
local traffic for unknown IPs belonging to us is discarded by our router 
instead of being sent out to CenturyLink.

    set protocols static route6 2602:c6:3364:4e00::/56 blackhole
    set protocols static interface-route6 ::/0 next-hop-interface tun0

## Commit and Save 
You have to save and commit the configuration, after that you can test
    commit;save

At this point, you should be able to use `ping6` on your Ubiquiti router.
Try `www.google.com`.

## Configure your LAN

Assuming you have a single local network on `switch0`, you can pick a prefix
(for example, `00`), and assign it to switch0:

    set interfaces switch switch0 address 2602:c6:3364:4e00::1/64

Then you can enable router advertisements:

    set interfaces switch switch0 ipv6 dup-addr-detect-transmits 1
    set interfaces switch switch0 ipv6 router-advert cur-hop-limit 64
    set interfaces switch switch0 ipv6 router-advert managed-flag false
    set interfaces switch switch0 ipv6 router-advert max-interval 30
    set interfaces switch switch0 ipv6 router-advert other-config-flag false
    set interfaces switch switch0 ipv6 router-advert prefix '::/64' autonomous-flag true
    set interfaces switch switch0 ipv6 router-advert prefix '::/64' on-link-flag true
    set interfaces switch switch0 ipv6 router-advert prefix '::/64' valid-lifetime 600
    set interfaces switch switch0 ipv6 router-advert reachable-time 0
    set interfaces switch switch0 ipv6 router-advert retrans-timer 0
    set interfaces switch switch0 ipv6 router-advert send-advert true

## Commit and Save 
You have to save and commit the configuration, after that you can test
    commit;save

These timers are somewhat aggressive to account for possible changes in
the underlying IPv4 address.

At this point, hosts on your LAN should begin performing SLAAC.

## Follow up

Look at the centurylink-6rd script in this project for inspiration on
managing prefix changes automatically. This script is intended to be
dropped into `/config/scripts/ppp/ip-up.d`, which will cause it to be run
automatically on any IP changes.

You may also want to set up IPv6-based DNS resolution. I run unbound
directly on my Edge Router and hand out its IPs for DNS configurations.
Your approach may look quite different.
