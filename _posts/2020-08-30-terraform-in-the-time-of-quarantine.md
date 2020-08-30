---
title: Terraform in the time of quarantine
tags: terraform linux systemd network-namespaces wireguard
---

For _reasons_, my current ISP has this funny hard limit of 2000 connections per device, and when you hit the limit you
get locked out of the network and need to "re-authenticate" to get back on, which typically includes logging into their
"portal" from a different device and removing the offending device and reconnecting.

This has generally not been an issue for me, I first hit the limit one day when I was messing around and running
`terraform plan` in multiple (fairly large) environments, and didn't bother too much about it.

However, since started working from home come March the fine year of 2020 (pandemic anyone?), I'm doing a lot of
terraform'ing daily and have some environments that are quite large and initially forced me over this limit multiple
times due to the amount of connections made to AWS. It's not that each individual `terraform plan/apply` creates 2000
connections, so I assume the ISP is measuring the number of connections created during a time period, or something like
that. Contacting their support did nothing, they closed the case after two weeks without any reply whatsoever.

As I have a subscription with the fine folks over at [ovpn](https://ovpn.com), and they (at the time) had started their
beta for [wireguard](https://wireguard.com), I figured I could just connect to their London endpoint and be done with
it. This worked perfectly fine initially, but it was just a bit _too easy_, and sometimes I don't want to be connected
to a VPN.

Linux network namespaces are pretty cool though, so why not create a namespace, wire it up with wireguard and `exec`
into it whenever I need to run Terraform?

So off we go with two `systemd` service units, inspired by [cloudnull.io]'s blog post, using veth pairs and a
configuration file used in said units:

[/etc/systemd/system/systemd-netns@.service]:

```
[Unit]
Description=Named network namespace %i
JoinsNamespaceOf=systemd-netns@%i.service
BindsTo=systemd-netns-access@%i.service
After=syslog.target network.target

[Service]
Type=oneshot
RemainAfterExit=true
PrivateNetwork=true
ExecStart=/usr/bin/ip netns add %I
ExecStart=/usr/bin/ip netns exec %I ip link set lo up
ExecStop=-/usr/bin/ip netns delete %I 2> /dev/null

[Install]
WantedBy=multi-user.target
WantedBy=network-online.target
```

[/etc/systemd/system/systemd-netns-access@.service]:

```
[Unit]
Description=Named network namespace %I
After=syslog.target network.target systemd-netns@%i.service
BindsTo=systemd-netns@%i.service

[Service]
Type=oneshot
RemainAfterExit=true
EnvironmentFile=/etc/netns.d/%I/config
ExecStartPre=/usr/bin/ip link add veth-${GATEWAY_DEVICE} type veth peer name veth-%I
ExecStartPre=/usr/bin/ip addr add ${GATEWAY_IP}/${NETWORK_NETMASK} dev veth-${GATEWAY_DEVICE}
ExecStartPre=/usr/bin/ip link set veth-%I netns %I
ExecStartPre=/usr/bin/ip netns exec %I ip addr add ${NETWORK_IP}/${NETWORK_NETMASK} dev veth-%I
ExecStartPre=/usr/bin/ip link set veth-${GATEWAY_DEVICE} up
ExecStartPre=/usr/bin/ip netns exec %I ip link set veth-%I up
ExecStartPre=/usr/bin/ip netns exec %I ip route add default via ${GATEWAY_IP}
ExecStartPre=/usr/bin/iptables -A FORWARD -o ${GATEWAY_DEVICE} -i veth-${GATEWAY_DEVICE} -j ACCEPT
ExecStartPre=/usr/bin/iptables -A FORWARD -i ${GATEWAY_DEVICE} -o veth-${GATEWAY_DEVICE} -j ACCEPT
ExecStartPre=/usr/bin/iptables -t nat -A POSTROUTING -s ${NETWORK_IP}/${NETWORK_NETMASK} -o ${GATEWAY_DEVICE} -j MASQUERADE
ExecStartPre=/usr/bin/sysctl -w net.ipv4.ip_forward=1
ExecStart=/usr/bin/ip netns exec %I wg-quick up /etc/wireguard/wg0-uk.conf
ExecStop=/usr/bin/ip netns exec %I wg-quick down /etc/wireguard/wg0-uk.conf
ExecStopPost=-/usr/bin/ip netns del %I
ExecStopPost=-/usr/bin/iptables -D FORWARD -o ${GATEWAY_DEVICE} -i veth-%I -j ACCEPT
ExecStopPost=-/usr/bin/iptables -D FORWARD -i ${GATEWAY_DEVICE} -o veth-%I -j ACCEPT
ExecStopPost=-/usr/bin/iptables -D POSTROUTING -t nat -s ${NETWORK_IP}/${NETWORK_NETMASK} -o ${GATEWAY_DEVICE} -j MASQUERADE

[Install]
WantedBy=multi-user.target
WantedBy=network-online.target
```

[/etc/netns.d/wireguard/config]:

```
GATEWAY_DEVICE=wlan0
GATEWAY_IP=192.168.0.1
NETWORK_IP=192.168.0.2
NETWORK_NETMASK=24
```

After that I was able to run this monstrosity:

```shell
$ sudo systemctl enable systemd-netns@wireguard.service --now
$ curl https://icanhazip.com
1.2.3.4
$ sudo -E ip netns exec wireguard sudo -E -u dist curl https://icanhazip.com
5.6.7.8
```

That's all well and good, but I don't really fancy running sudo (twice!) to run Terraform. Instead, enter [firejail],
which is SUID program that can (among other things) use different network namespaces. Perfect.

So rather than the `sudo` madness, I created a [wrapper script for Terraform]:

```shell
#!/usr/bin/env bash

if ip netns | grep --quiet wireguard; then
  exec firejail \
    --noprofile \
    --netns=wireguard \
    --writable-var \
    --writable-run-user \
    --quiet \
    /usr/bin/terraform $@
else
  echo "\`wireguard\` network namespace doesn't exist, running in \`init\` namespace" >&2
  exec /usr/bin/terraform $@
fi
```

Et voilà, runs like a dream. However, as systemd has native support for wireguard I might look into changing it up to
use that instead.

[cloudnull.io]: https://cloudnull.io/2019/04/running-services-in-network-name-spaces-with-systemd/
[/etc/systemd/system/systemd-netns@.service]: https://github.com/mthssdrbrg/dotfiles/blob/7d9e31c7cd6936ec87077481435db23987b8046a/etc/systemd/system/systemd-netns%40.service
[/etc/systemd/system/systemd-netns-access@.service]: https://github.com/mthssdrbrg/dotfiles/blob/7d9e31c7cd6936ec87077481435db23987b8046a/etc/systemd/system/systemd-netns-access%40.service
[/etc/netns.d/wireguard/config]: https://github.com/mthssdrbrg/dotfiles/blob/7d9e31c7cd6936ec87077481435db23987b8046a/etc/netns.d/wireguard/config
[firejail]: https://firejail.wordpress.com/
[wrapper script for Terraform]: https://github.com/mthssdrbrg/dotfiles/blob/7d9e31c7cd6936ec87077481435db23987b8046a/local/bin/terraform
