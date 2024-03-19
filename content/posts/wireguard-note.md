+++
title = "WireGuard Note"
date = "2024-03-19"
description = """wireguard deployment notes for myself case, recording best practice and problems"""

[taxonomies]
tags = [ "network", "proxy" ]

+++

It can be very hard to get stable network connection between Japan and China Mainland, even the two countries are not too far geographically. (Thanks to the Mama of GFW.) The same as Singapore. 

To solve the problem, we can set up a server node in HongKong, which route is stable and fast from China mainland. By the intermediate node, we reroute all traffic from mainland to Tokyo. At last, we have a stable international network route. Of course, HongKong server is stable enough and why not just use it? Maybe some applications are forbidden or would be blocked in the future.

So wireguard helps link two nodes as a virtual local subnetwork. Based on it, we can configure simple routes and rules to transfer all network traffic. 

## Wireguard Setup

It is not hard to set up a wireguard server and client peer. Just follow the tutorial on official quickstart document. And the best deployment tutorial from digitalocean.

Here is my personal configuration for server and client wireguard interface.

### Wireguard Server Config 

```shell
# cat /etc/wireguard/wg0.conf 
[Interface]
Address = 10.18.0.1/24
Address = fd3b:bd46:db84::1/64
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = ip6tables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
PostUp = ip6tables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = ip6tables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51888
PrivateKey = xxxxxxxx

[Peer]
PublicKey = xxxxxxxx
AllowedIPs = 10.18.0.2/32, fd3b:bd46:db84::2/128
```

1. `eth0` is the actual interface name for your server node
2. Address `10.18.0.1/24` and `fd3b:bd46:db84::1/64` is from the digitalocean tutorial
3. Update of `Forward` chain if it is `DROP` policy. `iptables -A FORWARD -i wg0 -j ACCEPT`
4. it is important to NAT the forwarding traffic packages with `MASQUERADE`, which hides your actual IPs
5. Set a fixed listening port, and allow it in firewall. `ListenPort = 51888`

### Wireguard Client Config

```shell
# cat /etc/wireguard/wg0.conf 
[Interface]
PrivateKey = xxxxxxxx
Address = 10.18.0.2/24
Address = fd3b:bd46:db84::2/64
ListenPort = 51888
DNS = 108.61.10.10 2001:19f0:300:1704::6
Table = off

PostUp = ip -4 route add default dev wg0 table 618
PostUp = ip -4 rule add fwmark 108 table 618
PostUp = ip -4 rule add from 10.18.0.2 table 618

PostUp = ip -6 route add default dev wg0 table 618
PostUp = ip -6 rule add fwmark 108 table 618
PostUp = ip -6 rule add from fd3b:bd46:db84::2 table 618

PreDown = ip -4 rule delete fwmark 108 table 618
PreDown = ip -4 rule delete from 10.18.0.2 table 618
PreDown = ip -4 route delete default dev wg0 table 618

PreDown = ip -6 rule delete fwmark 108 table 618
PreDown = ip -6 rule delete from fd3b:bd46:db84::2 table 618
PreDown = ip -6 route delete default dev wg0 table 618

[Peer]
PublicKey = xxxxxxxx
AllowedIPs = 0.0.0.0/0, ::/0 
Endpoint = {SERVER_PUBLIC_IP}:51888
```

1. Address must match the server net mask range.
2. `DNS` is optional, our intermediate server node could resolve domains, actually.
3. `Table = off` is very important, when starting the `wg-quick`, it disables the autoconfiguration for routing all traffics. So we can set up rules based on `fwmark` and `interface`.
4. It creates a route in table `route add default dev wg0 table 618`. Then add the two rules for rerouting to the table `618`
   1. `rule add fwmark 108 table 618`, set the `fwmark` value as you like
   2. `rule add from 10.18.0.2 table 618`, match the interface ip `10.18.0.2`
5. `AllowedIPs` is very important. It defines the target IP ranges for this peer, that it can reach by the wireguard server.

For our rerouting proxy service, we need the wireguard client to transfer all traffics. So the `AllowedIPs` must be `0.0.0.0/0, ::/0`. Or we could get error `No route to host`. However, once it is set, the wireguard tool `wg-quick` will set the route and rules automatically, which results ssh connection lost. For more details, check the [DigitalOcean Tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04#step-8-adding-the-peer-s-public-key-to-the-wireguard-server).

To resolve the problem, We should set `Table = Off`, it will disable the autoconfiguration.

### Useful Tips

#### Check the routes and rules work correctly

```shell
# ip route get 1.1.1.1 from 10.18.0.2
1.1.1.1 from 10.18.0.2 dev wg0 table 618 mark 0x6c uid 0 
    cache 

# ip route get 1.1.1.1 mark 108
1.1.1.1 dev wg0 table 618 src 10.18.0.2 mark 0x6c uid 0 
    cache 
```

#### Check the wireguard client work correctly

[Wireguard Kernel Debug Info](https://www.wireguard.com/quickstart/#debug-info)

```shell
modprobe wireguard && echo module wireguard +p > /sys/kernel/debug/dynamic_debug/control
```

```shell
# get the kernel debug info
dmesg -W
```

## Shadowsocks-Rust Config 

To make it work with the shadowsocks server together, we need to config the `outbound_fwmark` for servers. 

1. `"outbound_fwmark": 108` in config json file.
2. Or add with the start command line `--outbound-fwmark 108`

Check your IP and location on `ifconfig.co`

## Conclusion

With the help of wireguard, now we can set up our own routes to the world. If it has a wall, it always has a gap.

## Reference

1. [Wireguard Official Doc](https://www.wireguard.com/quickstart/)
2. [DigitalOcean Tutorial: How to set up wireguard on ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04)
3. [Vultr Tutorial: Set up wireguard vpn on ubuntu 20.04](https://docs.vultr.com/set-up-wireguard-vpn-on-ubuntu-20-04)
4. [Manual: wg-quick](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8)
5. [Manual: wg](https://man7.org/linux/man-pages/man8/wg.8.html)
6. [Manual: ip-rule](https://man7.org/linux/man-pages/man8/ip-rule.8.html)
6. [Manual: ip-route](https://man7.org/linux/man-pages/man8/ip-route.8.html)
