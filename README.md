# Whole-home Mullvad VPN + AdGuard Home on UGREEN/Linux NAS

A practical, tested setup for routing most home devices through **Mullvad WireGuard** while using **AdGuard Home** for network-wide DNS filtering, with a **client kill switch**, **DHCP on AdGuard**, and a simple **Netflix/smart-TV bypass** that keeps AdGuard filtering.

Tested topology:

- **ISP:** AT&T Fiber
- **ISP gateway:** BGW-series gateway in IP Passthrough mode
- **Router:** TP-Link Archer AX11000
- **Gateway/DNS/DHCP host:** UGREEN NAS running Linux
- **VPN:** Mullvad WireGuard
- **DNS filtering:** AdGuard Home in Docker

> **Important:** The IP addresses below are examples from one working network. Change them to match your own setup. Never publish your WireGuard private key, passwords, account information, or other secrets.

---

## What this setup does

```text
Internet
   |
AT&T / ISP gateway
   |
TP-Link AX11000
Router: 192.168.0.1
DHCP: OFF
   |
   +--------------------------+
   |                          |
UGREEN/Linux NAS              Smart TV
192.168.0.38                  192.168.0.50
AdGuard Home                  Gateway: 192.168.0.1
DHCP server                   DNS: 192.168.0.38
WireGuard/Mullvad             |
VPN gateway                   +--> ISP directly --> Netflix
   |
   +--> Most home devices
        Gateway: 192.168.0.38
        DNS: 192.168.0.38
        |
        +--> Mullvad VPN --> Internet
```

### Example values used in this guide

| Setting | Example |
|---|---|
| Main router | `192.168.0.1` |
| NAS / AdGuard / VPN gateway | `192.168.0.38` |
| LAN subnet | `192.168.0.0/24` |
| AdGuard DHCP pool | `192.168.0.100 - 192.168.0.249` |
| NAS LAN interface | `eth0` |
| WireGuard interface | `us-den-wg-101` |
| Smart TV static IP | `192.168.0.50` |
| Mullvad WireGuard DNS | `10.64.0.1` |

---

# 1. Optional: put the ISP gateway into passthrough mode

If your own router already receives the public WAN address, skip this section.

For an AT&T BGW-series gateway:

1. Open the AT&T gateway interface, commonly `192.168.1.254`.
2. Go to **Firewall > IP Passthrough**.
3. Set:
   - Allocation Mode: **Passthrough**
   - Passthrough Mode: **DHCPS-fixed**
   - Fixed MAC: your main router's WAN MAC
4. Save/restart as required.
5. If you only use your own router for Wi-Fi, disable AT&T Wi-Fi.
6. Confirm your main router receives the public IPv4 address on its WAN side.

AT&T documentation:

https://www.att.com/support/smallbusiness/article-modal/smb-internet/KM1188700/

---

# 2. Give the NAS a static LAN address

The machine running AdGuard and acting as the VPN gateway must not change IP.

Example:

```text
NAS IP:      192.168.0.38
Subnet:      255.255.255.0
Router:      192.168.0.1
Interface:   eth0
```

Check it on Linux:

```bash
ip -4 addr show dev eth0
```

Do **not** make the NAS use itself as its normal physical-interface gateway. Its base route still needs the real router (`192.168.0.1`) so WireGuard can reach the Mullvad server.

---

# 3. Run AdGuard Home in Docker

If AdGuard Home is already working, skip ahead.

For DHCP to work correctly from Docker, AdGuard Home recommends **host networking**.

Example:

```bash
docker run -d \
  --name adguard-new \
  --restart unless-stopped \
  --network host \
  -v /volume1/docker/adguard/work:/opt/adguardhome/work \
  -v /volume1/docker/adguard/conf:/opt/adguardhome/conf \
  adguard/adguardhome
```

Before doing this, check that nothing else is using DNS port 53 on the LAN interface:

```bash
sudo ss -lntup | grep ':53'
```

Do not disable an existing NAS DNS service unless you know what it is doing.

Official AdGuard Docker docs:

https://github.com/AdguardTeam/AdGuardHome/wiki/Docker

Example web interface used in this setup:

```text
http://192.168.0.38:3001
```

---

# 4. Set up Mullvad WireGuard on the NAS

Generate a Linux WireGuard configuration from Mullvad.

Official Mullvad guide:

https://mullvad.net/en/help/easy-wireguard-mullvad-setup-linux

Put the file under `/etc/wireguard/`:

```bash
sudo cp us-den-wg-101.conf /etc/wireguard/
sudo chmod 600 /etc/wireguard/us-den-wg-101.conf
```

Never post the contents publicly because it contains your private key.

Check that WireGuard tools exist:

```bash
which wg
which wg-quick
```

Bring the tunnel up:

```bash
sudo wg-quick up us-den-wg-101
```

Check it:

```bash
sudo wg
```

You want to see a recent handshake and:

```text
allowed ips: 0.0.0.0/0, ::/0
```

Test Mullvad:

```bash
curl https://am.i.mullvad.net/connected
```

It should report that you are connected to Mullvad.

## If `wg-quick` fails because `resolvconf` is missing

Some NAS distributions include WireGuard but not `resolvconf`.

If AdGuard Home will handle DNS, remove the `DNS = ...` line from the Mullvad WireGuard config:

```bash
sudo sed -i '/^[[:space:]]*DNS[[:space:]]*=/d' /etc/wireguard/us-den-wg-101.conf
```

Then try again:

```bash
sudo wg-quick up us-den-wg-101
```

This does **not** remove VPN encryption. It only stops `wg-quick` from trying to change the NAS resolver.

---

# 5. Make WireGuard start automatically

```bash
sudo systemctl enable wg-quick@us-den-wg-101
sudo systemctl start wg-quick@us-den-wg-101
```

Check:

```bash
sudo systemctl status wg-quick@us-den-wg-101
```

---

# 6. Enable IPv4 forwarding

Check:

```bash
sysctl net.ipv4.ip_forward
```

If it is `0`:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-vpn-gateway.conf
```

---

# 7. Add LAN-to-VPN firewall/NAT rules and a client kill switch

Create:

```text
/usr/local/sbin/mullvad-gateway-firewall.sh
```

Contents:

```sh
#!/bin/sh

LAN_IF="eth0"
WG_IF="us-den-wg-101"
LAN_NET="192.168.0.0/24"

iptables -C FORWARD -s "$LAN_NET" -i "$LAN_IF" -o "$WG_IF" -j ACCEPT 2>/dev/null || \
iptables -I FORWARD 1 -s "$LAN_NET" -i "$LAN_IF" -o "$WG_IF" -j ACCEPT

iptables -C FORWARD -d "$LAN_NET" -i "$WG_IF" -o "$LAN_IF" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
iptables -I FORWARD 2 -d "$LAN_NET" -i "$WG_IF" -o "$LAN_IF" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

iptables -C FORWARD -s "$LAN_NET" ! -d "$LAN_NET" -i "$LAN_IF" -o "$LAN_IF" -j REJECT --reject-with icmp-port-unreachable 2>/dev/null || \
iptables -I FORWARD 3 -s "$LAN_NET" ! -d "$LAN_NET" -i "$LAN_IF" -o "$LAN_IF" -j REJECT --reject-with icmp-port-unreachable

iptables -t nat -C POSTROUTING -s "$LAN_NET" -o "$WG_IF" -j MASQUERADE 2>/dev/null || \
iptables -t nat -I POSTROUTING 1 -s "$LAN_NET" -o "$WG_IF" -j MASQUERADE
```

Make it executable:

```bash
sudo chmod 700 /usr/local/sbin/mullvad-gateway-firewall.sh
sudo /usr/local/sbin/mullvad-gateway-firewall.sh
```

### What these rules do

- Allow LAN clients to leave through WireGuard.
- Allow established replies back from WireGuard.
- NAT LAN clients behind the WireGuard interface.
- Reject forwarded LAN traffic that tries to fall back out the normal `eth0` path toward the ISP.

> This kill switch protects **forwarded LAN clients**. It does not automatically create a kill switch for applications running locally on the NAS itself. NAS-originated traffic uses the `OUTPUT` chain and is a separate hardening task.

---

# 8. Make the firewall persistent with systemd

Create:

```text
/etc/systemd/system/mullvad-gateway-firewall.service
```

Contents:

```ini
[Unit]
Description=Mullvad LAN gateway firewall
Requires=wg-quick@us-den-wg-101.service
After=wg-quick@us-den-wg-101.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/mullvad-gateway-firewall.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mullvad-gateway-firewall.service
sudo systemctl start mullvad-gateway-firewall.service
```

Check:

```bash
sudo systemctl status mullvad-gateway-firewall.service
```

---

# 9. Configure AdGuard Home DNS

Set AdGuard's upstream DNS server to:

```text
10.64.0.1
```

That is Mullvad's unfiltered DNS inside the WireGuard tunnel.

Mullvad documentation:

https://mullvad.net/en/help/pfsense-with-wireguard

Result:

```text
Home device
   -> AdGuard Home
      -> Mullvad DNS through WireGuard
         -> Internet
```

AdGuard filters first; Mullvad handles upstream DNS resolution.

---

# 10. Move DHCP from the router to AdGuard Home

This was necessary on the tested AX11000 because configuring the router DHCP to advertise only AdGuard still resulted in clients receiving both:

```text
192.168.0.38
192.168.0.1
```

That allowed the router's own DNS proxy to bypass AdGuard.

AdGuard Home DHCP documentation:

https://github.com/AdguardTeam/AdGuardHome/wiki/DHCP

Configure AdGuard DHCP:

```text
Interface:       eth0
Gateway:         192.168.0.38
Subnet mask:     255.255.255.0
DHCP start:      192.168.0.100
DHCP end:        192.168.0.249
DNS:             192.168.0.38
```

Then disable DHCP on the main router.

> **Do not disable the router DHCP until AdGuard DHCP is ready.**

---

# 11. AdGuard DHCP YAML fallback

Normally use the AdGuard web interface.

If the DHCP UI fails, or clients receive DNS but no default gateway, stop AdGuard before editing its YAML.

Backup:

```bash
sudo cp /volume1/docker/adguard/conf/AdGuardHome.yaml \
        /volume1/docker/adguard/conf/AdGuardHome.yaml.bak
```

Stop AdGuard:

```bash
sudo docker stop adguard-new
```

Relevant configuration:

```yaml
dhcp:
  enabled: true
  interface_name: eth0
  local_domain_name: lan
  dhcpv4:
    gateway_ip: 192.168.0.38
    subnet_mask: 255.255.255.0
    range_start: 192.168.0.100
    range_end: 192.168.0.249
    lease_duration: 7200
    icmp_timeout_msec: 1000
    options:
      - 3 ip 192.168.0.38
      - 6 ip 192.168.0.38
  dhcpv6:
    range_start: ""
    lease_duration: 86400
    ra_slaac_only: false
    ra_allow_slaac: false
```

- DHCP option **3** = router/default gateway
- DHCP option **6** = DNS server

Restart:

```bash
sudo docker start adguard-new
sudo docker logs adguard-new --tail 100
```

Look for:

```text
dhcpv4: listening
Server listening on 0.0.0.0:67
Ready to handle requests
```

---

# 12. Renew a Windows client

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```

Expected:

```text
IPv4 Address:    192.168.0.x
Default Gateway: 192.168.0.38
DHCP Server:     192.168.0.38
DNS Server:      192.168.0.38
```

That means:

```text
PC -> NAS -> Mullvad
DNS -> AdGuard
```

---

# 13. End-to-end tests

## DNS

```powershell
nslookup google.com
```

The DNS server should be:

```text
192.168.0.38
```

## Ad blocking

If your list blocks it:

```powershell
nslookup doubleclick.net
```

AdGuard often returns:

```text
0.0.0.0
::
```

The best proof is the **AdGuard Query Log**: confirm the client's IP is generating requests and that some domains are marked blocked.

## VPN

```powershell
curl.exe https://am.i.mullvad.net/connected
```

Expected:

```text
You are connected to Mullvad
```

## Route

```powershell
tracert -d -h 2 1.1.1
```

The first hop should be the NAS:

```text
192.168.0.38
```

---

# 14. Recommended AdGuard blocklists

A good balance for a home network:

## HaGeZi Multi PRO

```text
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt
```

## HaGeZi TIF Medium

```text
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/tif.medium.txt
```

Project:

https://github.com/hagezi/dns-blocklists

Do not stack every general-purpose blocklist you can find. Large lists overlap heavily and make troubleshooting harder.

### YouTube limitation

DNS blocking cannot reliably remove YouTube's in-stream video ads because video and advertising can use overlapping first-party infrastructure. DNS filtering is still very useful for trackers, telemetry, malicious domains, third-party advertising, and smart-TV background traffic.

---

# 15. Bypass Mullvad for a smart TV while keeping AdGuard

Some streaming services reject known VPN exit addresses.

The simplest fix is to give only the TV a static network configuration.

Example:

```text
IP address:    192.168.0.50
Subnet mask:   255.255.255.0
Gateway:       192.168.0.1
DNS:           192.168.0.38
Secondary DNS: blank
```

If the TV forces you to enter two DNS servers:

```text
Primary DNS:   192.168.0.38
Secondary DNS: 192.168.0.38
```

Choose a static IP **outside the DHCP pool**.

Result:

```text
TV Internet -> main router -> ISP -> Netflix
TV DNS      -> AdGuard Home
```

Netflix now sees the real residential ISP connection, while AdGuard still provides DNS filtering.

The TV is **not protected by Mullvad** in this mode.

Verify AdGuard by opening **AdGuard Home > Query Log** and confirming requests from the TV's IP.

---

# 16. Fast reboot / power-loss health check

After a reboot or accidental shutdown, wait for the NAS and Docker to finish starting, then run:

```bash
sudo systemctl is-active wg-quick@us-den-wg-101 mullvad-gateway-firewall.service; \
sudo docker ps --filter name=adguard-new --format 'AdGuard: {{.Status}}'; \
sudo wg | grep -E 'interface:|latest handshake'
```

Healthy output should include:

```text
active
active
AdGuard: Up ...
interface: us-den-wg-101
latest handshake: ... ago
```

Then on Windows:

```powershell
nslookup google.com
curl.exe https://am.i.mullvad.net/connected
```

If DNS fails immediately after a cold boot but all services are active, wait 30-90 seconds and retry before changing configuration.

---

# 17. Non-disruptive kill-switch checks

These do not intentionally take the VPN down:

```bash
sudo iptables -S FORWARD
sudo iptables -t nat -S POSTROUTING
sudo wg
ip rule
ip route show table 51820
```

You want equivalents of:

```text
LAN -> WireGuard ACCEPT
WireGuard -> LAN established/related ACCEPT
LAN eth0 -> eth0 non-LAN REJECT
LAN -> WireGuard MASQUERADE
```

The WireGuard interface should have a recent handshake.

A **true** kill-switch test requires deliberately taking the tunnel down and briefly interrupts VPN-routed clients. Do that only when you are prepared for a temporary outage.

---

# Troubleshooting

## `curl` says "Could not resolve host"

Test AdGuard directly:

```powershell
nslookup google.com 192.168.0.38
```

### If that works

AdGuard is healthy. The client may not currently be using it.

Check:

```powershell
nslookup google.com
ipconfig /all
```

DNS should be `192.168.0.38`.

Renew DHCP if needed:

```powershell
ipconfig /release
ipconfig /renew
```

A cold-boot race can also cause a temporary DNS failure while Docker, WireGuard, and AdGuard are still starting.

### If direct AdGuard lookup returns SERVFAIL

AdGuard is reachable, but the upstream resolver may not be.

Check:

```bash
sudo wg
curl https://am.i.mullvad.net/connected
```

Confirm the WireGuard handshake is recent.

### If direct AdGuard lookup times out

Check the container:

```bash
sudo docker ps --filter name=adguard-new
sudo docker logs adguard-new --tail 100
```

Check DNS listeners:

```bash
sudo ss -lntup | grep ':53'
```

---

## WireGuard is up but clients have no Internet

Check IP forwarding:

```bash
sysctl net.ipv4.ip_forward
```

It should be:

```text
net.ipv4.ip_forward = 1
```

Check forwarding/NAT:

```bash
sudo iptables -S FORWARD
sudo iptables -t nat -S POSTROUTING
```

Check policy routing:

```bash
ip rule
ip route show table 51820
```

The WireGuard routing table should have a default route via the WireGuard interface.

---

## `wg-quick` complains about `resolvconf`

If AdGuard is handling DNS, remove the `DNS = ...` line from the WireGuard config and bring the tunnel up again.

Do **not** remove `PrivateKey`, `Address`, `AllowedIPs`, `Endpoint`, or other VPN settings.

---

## Windows receives AdGuard DNS but no default gateway

Verify AdGuard DHCP.

It should advertise:

```text
Option 3 = 192.168.0.38
Option 6 = 192.168.0.38
```

If needed, use the YAML fallback above.

Then:

```powershell
ipconfig /release
ipconfig /renew
```

---

## TP-Link clients receive both AdGuard and the router as DNS

On the tested Archer AX11000 firmware, leaving secondary DNS blank still resulted in clients receiving both AdGuard and the router's own DNS proxy.

The clean fix was:

1. Run DHCP from AdGuard Home.
2. Disable DHCP on the AX11000.
3. Let AdGuard hand out both gateway and DNS.

### AX11000-specific warning

Do **not** assume setting the AX11000 WAN DNS to the LAN AdGuard IP is a safe workaround.

On the tested AX11000 + AT&T topology, setting WAN DNS to the NAS LAN IP caused the router to detect an apparent upstream IP conflict and change its LAN subnet.

That change was reverted. Moving DHCP to AdGuard was the reliable fix.

---

## Netflix says a VPN is being used

Give the TV:

```text
Gateway: 192.168.0.1
DNS:     192.168.0.38
```

Netflix traffic now exits through the ISP while DNS still goes through AdGuard.

---

## Netflix works, but how do I know AdGuard still works on the TV?

Open **AdGuard Home > Query Log** and use the TV for a minute.

You should see DNS requests from the TV's static IP, including both allowed and blocked domains.

---

## AdGuard DHCP UI does not save

Back up the config, stop the container, edit `AdGuardHome.yaml`, then restart the container.

Do not edit the YAML while AdGuard is running because it can overwrite the file.

---

# Security and privacy limitations

## 1. This client kill switch is IPv4-focused

The tested LAN clients did not have globally routed IPv6; they only had link-local IPv6.

If your ISP/router gives clients global IPv6, do **not** assume the IPv4 firewall rules above prevent an IPv6 bypass. Configure equivalent IPv6 routing/firewall policy or intentionally disable IPv6 only after understanding the consequences.

## 2. The NAS itself is different from forwarded clients

The firewall above protects devices forwarding **through** the NAS.

It does not automatically stop locally generated NAS traffic from using the physical ISP route if WireGuard is down. A host-level `OUTPUT` kill switch is a separate step.

## 3. DNS filtering is not the same as a firewall

A device or application can use hard-coded DNS, DNS-over-HTTPS, or another resolver instead of AdGuard.

This guide makes normal DHCP clients use AdGuard. It does not forcibly intercept every possible encrypted DNS implementation.

## 4. Never publish VPN secrets

Before sharing logs or screenshots, redact:

```text
WireGuard PrivateKey
router passwords
ISP account information
public IP address if you prefer not to disclose it
device MAC addresses
serial numbers
API keys/tokens
```

WireGuard public keys are not secret, but there is usually no reason to publish them either.

---

# Quick reference

```text
AT&T / ISP
    |
Main router
192.168.0.1
DHCP OFF
    |
    +-------------------------------+
    |                               |
NAS 192.168.0.38                    TV 192.168.0.50
AdGuard Home                        Gateway .1
DHCP                                DNS .38
WireGuard                           |
iptables/NAT                        +--> ISP direct
    |                                    Netflix works
    |
    +--> Normal DHCP clients
         Gateway .38
         DNS .38
              |
              +--> AdGuard filtering
              +--> Mullvad WireGuard
```

## Minimal "is everything alive?" check

### NAS

```bash
sudo systemctl is-active wg-quick@us-den-wg-101 mullvad-gateway-firewall.service; \
sudo docker ps --filter name=adguard-new --format 'AdGuard: {{.Status}}'; \
sudo wg | grep -E 'interface:|latest handshake'
```

### Windows

```powershell
nslookup google.com
curl.exe https://am.i.mullvad.net/connected
```

### TV

```text
Netflix plays
AdGuard Query Log shows requests from the TV
```

If all three pass, the practical setup is working.

---

# Sources

- Mullvad — WireGuard on Linux:
  https://mullvad.net/en/help/easy-wireguard-mullvad-setup-linux
- Mullvad — WireGuard DNS:
  https://mullvad.net/en/help/pfsense-with-wireguard
- AdGuard Home — DHCP:
  https://github.com/AdguardTeam/AdGuardHome/wiki/DHCP
- AdGuard Home — Docker:
  https://github.com/AdguardTeam/AdGuardHome/wiki/Docker
- AdGuard Home — Configuration:
  https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration
- HaGeZi DNS blocklists:
  https://github.com/hagezi/dns-blocklists
- AT&T IP Passthrough:
  https://www.att.com/support/smallbusiness/article-modal/smb-internet/KM1188700/
- TP-Link Archer AX11000 user guide:
  https://www.tp-link.com/us/user-guides/archer-ax11000_v1/

---

## Tested result

With the setup above:

- Normal clients automatically use AdGuard DNS and Mullvad.
- The LAN client kill switch prevents normal IPv4 fallback through the physical router if the VPN route disappears.
- The TV can bypass Mullvad while still using AdGuard.
- Netflix works on the bypassed TV.
- AdGuard continues filtering TV DNS requests.
- WireGuard, the firewall service, and AdGuard can all be checked quickly after a reboot.

Contributions, corrections, and hardware-specific notes are welcome.


---

# License

This project's original documentation is licensed under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**. You may share and adapt it, including commercially, as long as appropriate attribution is provided and changes are indicated.

See [LICENSE.md](LICENSE.md) for details.
