<h1 align="center">RouterOS Samples</h1>
<p align="center">
    <a href="https://github.com/rios0rios0/routeros-samples/releases/latest">
        <img src="https://img.shields.io/github/release/rios0rios0/routeros-samples.svg?style=for-the-badge&logo=github" alt="Latest Release"/></a>
    <a href="https://github.com/rios0rios0/routeros-samples/blob/main/LICENSE">
        <img src="https://img.shields.io/github/license/rios0rios0/routeros-samples.svg?style=for-the-badge&logo=github" alt="License"/></a>
</p>

A collection of production-ready MikroTik RouterOS configuration scripts (`.rsc`) covering Layer 7 content filtering, multi-WAN load balancing, firewall rules, NAT, DHCP, web proxy, and VLAN-style network segmentation. These scripts were extracted from real RouterBOARD 750 series deployments (RB750Gr2 and RB750Gr3) running RouterOS 6.x.

## Features

- **Layer 7 Protocol Filtering** -- regex-based deep packet inspection rules that match and block traffic to Google and YouTube at the application layer
- **Address List Whitelisting** -- firewall address lists that permit traffic only to a curated set of allowed domains (JW.org ecosystem), blocking all other destinations
- **HTTPS and Push Notification Blocking** -- targeted rejection of TCP port 443 (HTTPS) and port 2195 (Apple Push Notifications) for non-exempt hosts
- **Dual-Subnet Network Segmentation** -- bridge-based architecture separating "open" (unrestricted) and "protected" (filtered) network segments with independent DHCP pools
- **Web Proxy with Transparent Redirect** -- HTTP traffic on port 80 is redirected to an on-device web proxy (port 8080) using DSTNAT rules
- **NAT Masquerading and Address Translation** -- source NAT (masquerade) for internet access plus bidirectional DSTNAT/SRCNAT for internal host mapping between subnets
- **Multi-WAN Load Balancing** -- per-connection-classifier (PCC) based load balancing across two WAN interfaces (DHCP and/or PPPoE uplinks) with `check-gateway=ping` failover
- **Auto-Balance Script** -- a RouterOS scripting language automation that dynamically discovers all PPPoE-client and DHCP-client interfaces, cleans up previous balancing configuration, and creates mangle rules, routes, and NAT entries for each link automatically
- **UniFi Controller Access Rules** -- dedicated address list and firewall rules for UniFi network management controller connectivity

## Technologies

| Component | Details |
|-----------|---------|
| **Platform** | MikroTik RouterOS 6.36.1 -- 6.47.10 |
| **Hardware** | RouterBOARD 750 r2 (RB750Gr2), RouterBOARD 750 r3 (RB750Gr3) |
| **Language** | RouterOS Scripting Language (`.rsc`) |
| **Protocols** | Layer 7 (HTTP regex), TCP/IP, DHCP, PPPoE, NAT, Web Proxy |

## Project Structure

```
routeros-samples/
├── Layer7 Filter/
│   ├── RB750Gr2v1.rsc       # RB750Gr2 config (RouterOS 6.43.8) - 172.31.x.x subnets, proxy + sniffer
│   ├── RB750Gr2v2.rsc       # RB750Gr2 config (RouterOS 6.43.8) - updated with UniFi rules, additional JW domains
│   └── RB750Gr3.rsc         # RB750Gr3 config (RouterOS 6.36.1) - 192.168.x.x subnets, proxy redirect to jw.org
├── LoadBalancer/
│   ├── AUTO-BALANCE.rsc     # Automated multi-WAN balancing script (PPPoE + DHCP auto-discovery)
│   └── RB750Gr3.rsc         # RB750Gr3 config (RouterOS 6.47.10) - dual-WAN PCC load balancing with Google DNS
├── LICENSE                  # GNU General Public License v3.0
└── README.md
```

### Layer 7 Filter Configurations

All three Layer 7 configurations share a common architecture:

1. **Interface Layout**: `ether1` as WAN input ("Entrada do Provedor"), `ether2` as open output ("Saida 01 - Aberta"), `ether3-5` bridged together as protected outputs ("Saida 02-04 - Protegida") via a bridge named "Ponte de Seguranca"
2. **DHCP Pools**: a single-address "open" pool for an unrestricted host, and a range pool for the protected segment
3. **Firewall Chain**: accept whitelisted destinations, reject HTTPS (443), reject push notifications (2195), reject Layer 7 Google/YouTube matches, reject all remaining traffic -- all with ICMP network-unreachable responses
4. **Proxy**: HTTP proxy enabled with domain-specific access rules

**RB750Gr2v1** and **RB750Gr2v2** use `172.31.0.0/24` and `172.31.1.0/24` subnets. **RB750Gr2v2** adds UniFi controller address lists (`177.71.206.0/24`, `192.168.1.254`) and additional JW domains (`my.jw.org`, `login.jwpub.org`, `ba.jw.org`). **RB750Gr3** uses `192.168.0.0/24` and `192.168.1.0/24` subnets and includes an active proxy deny rule that redirects non-exempt hosts to `jw.org`.

### Load Balancer Configurations

- **AUTO-BALANCE.rsc**: a standalone script that iterates over all PPPoE-client and DHCP-client interfaces, removes any existing balance configuration (NAT, mangle, routes), and recreates PCC-based load balancing rules using `both-addresses` as the classifier. It calculates the total link count and assigns each link a sequential classifier slot.
- **RB750Gr3.rsc**: a complete router configuration with two WAN interfaces (1-WAN1 on ether1, 2-WAN2 on ether2), three LAN ports bridged together (ether3-5), PCC load balancing with `both-addresses:2/0` and `both-addresses:2/1`, DNS set to Google (8.8.8.8 / 8.8.4.4), and the AUTO-BALANCE script embedded as a system script.

## How to Use

### Importing a Full Configuration

1. Connect to the MikroTik device via WinBox or SSH
2. Navigate to **Files** and upload the `.rsc` file
3. Open **New Terminal** and run: `/import file-name=<filename>.rsc`

### Running the Auto-Balance Script

1. Navigate to **System > Scripts**
2. Click **(+)** to add a new script
3. Paste the contents of `AUTO-BALANCE.rsc` into the script body
4. Open **New Terminal**
5. Run: `/system script run <SCRIPT_NAME>`

The script will automatically detect all PPPoE and DHCP WAN links and configure PCC load balancing across them.

## Network Architecture

```
ISP 1 ──► ether1 (WAN1) ──┐
                           ├──► MikroTik RB750 ──► Bridge (ether3-5) ──► Protected LAN
ISP 2 ──► ether2 (WAN2) ──┘         │
                                     └──► ether2 (Open LAN) ──► Unrestricted Host
```

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).
