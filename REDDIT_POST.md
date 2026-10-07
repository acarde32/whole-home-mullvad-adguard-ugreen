# Reddit post draft

## Recommended title

**Whole-home Mullvad + AdGuard Home using a UGREEN/Linux NAS — DHCP, kill switch, AX11000, and Netflix bypass**

## Post

I finally got a whole-home setup working where almost every device automatically uses **Mullvad WireGuard** and **AdGuard Home** without installing VPN software on each client.

My topology is:

```text
AT&T Fiber
   |
AT&T gateway (IP Passthrough)
   |
TP-Link Archer AX11000
   |
UGREEN/Linux NAS
   |-- AdGuard Home (DNS + DHCP)
   |-- Mullvad WireGuard
   |-- IPv4 forwarding/NAT
   |-- forwarded-client kill switch
   |
Most home devices -> Mullvad
```

The NAS is the normal clients' **default gateway and DNS server**, so:

```text
Client -> AdGuard -> Mullvad -> Internet
```

I also wanted Netflix to work on the TV without giving up AdGuard. The simple solution was to give the TV a static configuration with:

```text
Gateway: main router
DNS:     AdGuard Home
```

So the TV's Internet traffic goes directly through the residential ISP, while its DNS requests still go through AdGuard.

That ends up looking like:

```text
Most devices:
Gateway -> NAS -> Mullvad
DNS     -> AdGuard

TV:
Gateway -> Router -> ISP
DNS     -> AdGuard
```

I documented the complete setup, including:

- Mullvad WireGuard on Linux/UGREEN
- AdGuard Home in Docker
- moving DHCP from the TP-Link router to AdGuard
- persistent iptables forwarding/NAT rules
- an IPv4 client kill switch
- systemd persistence after reboot
- HaGeZi Multi PRO + TIF Medium blocklists
- Netflix/smart-TV VPN bypass
- quick post-reboot health checks
- troubleshooting the problems I hit along the way
- IPv6 / host-level kill-switch limitations

Full guide:

https://github.com/acarde32/whole-home-mullvad-adguard-ugreen

One important caveat: the documented kill switch protects **forwarded IPv4 LAN clients**. It is not automatically a host-level kill switch for applications running on the NAS itself, and globally routed IPv6 needs its own routing/firewall treatment.

I would be interested in hearing from anyone running a similar NAS-as-router/VPN-gateway setup, especially on other UGREEN models or TP-Link firmware versions. Corrections and hardware-specific notes are welcome.
