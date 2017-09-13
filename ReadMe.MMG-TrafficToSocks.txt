https://unix.stackexchange.com/questions/144562/redirect-all-packets-from-eth1-eth2-through-a-socks-proxy

Redirect ALL packets from eth1 & eth2 through a SOCKS proxy

First, you need tun2socks (often a part of the 'badvpn' package). tun2socks sets up a virtual interface which you can route traffic through, and that traffic will get sent through the target socks proxy.

Setting it up gets a little tricky as you only want to route certain traffic through the tunnel.

This script should do what you want:

-----------------------------------------------------------------
SHELL SCRIPT
-----------------------------------------------------------------
#!/bin/bash
socks_server=127.0.0.1:8080

id="$RANDOM"
tun="$(printf 'tun%04x' "$id")"
ip tuntap add dev $tun mode tun
ip link set $tun up
ip addr add 169.254.1.1/30 dev $tun
sysctl -w net.ipv4.conf.$tun.forwarding=1
ip rule add fwmark $id lookup $id
ip route add default via 169.254.1.2 table $id
iptables -t mangle -I PREROUTING -i eth1 -p tcp -j MARK --set-mark $id
iptables -t mangle -I PREROUTING -i eth2 -p tcp -j MARK --set-mark $id
badvpn-tun2socks --tundev $tun --netif-ipaddr 169.254.1.2 --netif-netmask 255.255.255.252 --socks-server-addr $socks_server

iptables -t mangle -D PREROUTING -i eth2 -p tcp -j MARK --set-mark $id
iptables -t mangle -D PREROUTING -i eth1 -p tcp -j MARK --set-mark $id
ip route del default via 169.254.1.2 table $id
ip rule del from fwmark $id lookup $id
ip tuntap del dev $tun mode tun
-----------------------------------------------------------------

Explanation:

socks_server=127.0.0.1:8080
This is the socks server we will use.
 

id="$RANDOM"
tun="$(printf 'tun%04x' "$id")"
These generate a random ID to use for the tunnel. Since you may have other tunnels on the system, we can't just use tun0 or tun1. 99% of the time this will work fine. Adjust accordingly though.
 

ip tuntap add dev $tun mode tun
ip link set $tun up
ip addr add 169.254.1.1/30 dev $tun
sysctl -w net.ipv4.conf.$tun.forwarding=1
These set up the tunnel interface tun2socks will use.
 

ip rule add fwmark $id lookup $id
ip route add default via 169.254.1.2 table $id
These create a routing table with a single rule which sends any traffic with firewall mark $id (covered next) through the tunnel.
 

iptables -t mangle -I PREROUTING -i eth1 -p tcp -j MARK --set-mark $id
iptables -t mangle -I PREROUTING -i eth2 -p tcp -j MARK --set-mark $id
These set firewall mark $id on any TCP packets coming in eth1 or eth2. We only want to match TCP. Socks can't handle UDP or ICMP (tun2socks does have a way to forward UDP, but it's more complicated, and so I'm leaving it out).

badvpn-tun2socks --tundev $tun --netif-ipaddr 169.254.1.2 --netif-netmask 255.255.255.252 --socks-server-addr $socks_server

This starts tun2socks up. It'll sit in the foreground until terminated.
 

iptables -t mangle -D PREROUTING -i eth2 -p tcp -j MARK --set-mark $id
iptables -t mangle -D PREROUTING -i eth1 -p tcp -j MARK --set-mark $id
ip route del default via 169.254.1.2 table $id
ip rule del from fwmark $id lookup $id
ip tuntap del dev $tun mode tun
These tear down everything we created during the setup process. They will only run once badvpn-tun2socks exits.
