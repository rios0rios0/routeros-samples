# RouterOS Samples Repository

This repository is a collection of production-ready MikroTik RouterOS configuration scripts (`.rsc`) covering Layer 7 content filtering, multi-WAN load balancing, firewall rules, NAT, DHCP, web proxy, and VLAN-style network segmentation. Scripts were extracted from real RouterBOARD 750 series deployments (RB750Gr2 and RB750Gr3) running RouterOS 6.x.

**ALWAYS** reference these instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

## Working Effectively

### Bootstrap and Test the Repository

There is no build system or package manager. All scripts are standalone `.rsc` files interpreted by the RouterOS CLI. To validate a script you need a RouterOS device or a [CHR](https://mikrotik.com/download) virtual machine:

```sh
# Upload the .rsc file to the device via SCP
scp my-script.rsc admin@<router-ip>:/

# SSH into the router and import the script (takes <5 seconds)
ssh admin@<router-ip>
/import file-name=my-script.rsc
```

### Running the Auto-Balance Script

```routeros
# Add the script content via System > Scripts in WinBox, then run:
/system script run <SCRIPT_NAME>
```

### Verifying Applied Configuration

```routeros
/ip firewall filter print
/ip firewall mangle print
/ip route print
/ip firewall nat print
```

## Validation

### Manual Testing Steps

- **ALWAYS** test `.rsc` scripts on a real RouterOS device or MikroTik CHR VM before submitting a PR
- Import the full config to verify no syntax errors: `/import file-name=<filename>.rsc`
- Verify individual subsystems after import (firewall, mangle, routes, NAT)
- For `AUTO-BALANCE.rsc`: run the script and confirm mangle/route/NAT rules were created for each WAN link
- **No automated linting or CI/CD exists** in this repository -- all validation is manual

### Expected Timing

- **SCP file transfer**: <1 second for small `.rsc` files
- **`/import` execution**: 1-10 seconds depending on config size
- **Auto-balance script**: <5 seconds on live hardware

## Repository Structure

```
routeros-samples/
├── Layer7 Filter/
│   ├── RB750Gr2v1.rsc       # RB750Gr2 config (RouterOS 6.43.8) - 172.31.x.x subnets, proxy + sniffer
│   ├── RB750Gr2v2.rsc       # RB750Gr2 config (RouterOS 6.43.8) - adds UniFi rules and extra JW domains
│   └── RB750Gr3.rsc         # RB750Gr3 config (RouterOS 6.36.1) - 192.168.x.x subnets, proxy to jw.org
├── LoadBalancer/
│   ├── AUTO-BALANCE.rsc     # Standalone script: auto-discovers PPPoE + DHCP WANs, builds PCC rules
│   └── RB750Gr3.rsc         # RB750Gr3 config (RouterOS 6.47.10) - dual-WAN PCC load balancing
├── .github/
│   └── copilot-instructions.md
├── CONTRIBUTING.md
├── LICENSE                  # GNU General Public License v3.0
└── README.md
```

## Technology Stack

| Component | Details |
|-----------|---------|
| **Platform** | MikroTik RouterOS 6.36.1 -- 6.47.10 |
| **Hardware** | RouterBOARD 750 r2 (RB750Gr2), RouterBOARD 750 r3 (RB750Gr3) |
| **Language** | RouterOS Scripting Language (`.rsc`) -- proprietary CLI-based scripting |
| **Protocols** | Layer 7 (HTTP regex), TCP/IP, DHCP, PPPoE, NAT, Web Proxy |
| **Dependencies** | None -- standalone configuration files with no external dependencies |

## Architecture and Design Patterns

### Layer 7 Filter Configurations

All three Layer 7 configurations share a common architecture:

1. **Interface layout**: `ether1` as WAN input, `ether2` as open output, `ether3-5` bridged as protected outputs via a bridge named `Ponte de Segurança`
2. **DHCP pools**: a single-address "open" pool for an unrestricted host, and a range pool for the protected segment
3. **Firewall chain order**: accept whitelisted destinations → reject HTTPS (443) → reject push notifications (2195) → reject Layer 7 Google/YouTube matches → reject all remaining traffic (all with ICMP network-unreachable)
4. **Web proxy**: HTTP traffic on port 80 redirected to on-device proxy (port 8080) via DSTNAT

`RB750Gr2v1` and `RB750Gr2v2` use `172.31.0.0/24` and `172.31.1.0/24` subnets. `RB750Gr2v2` adds UniFi controller address lists and additional JW domains. `RB750Gr3` uses `192.168.0.0/24` and `192.168.1.0/24` subnets.

### Load Balancer Configurations

- **AUTO-BALANCE.rsc**: iterates over all PPPoE-client and DHCP-client interfaces, removes existing balance config (NAT, mangle, routes), then recreates PCC-based load balancing using `both-addresses` as the classifier, assigning each link a sequential slot
- **RB750Gr3.rsc**: complete config with two WAN interfaces on `ether1`/`ether2`, three LAN ports bridged (`ether3-5`), PCC rules `both-addresses:2/0` and `both-addresses:2/1`, DNS set to `8.8.8.8`/`8.8.4.4`, and AUTO-BALANCE embedded as a system script

### Network Topology

**Layer 7 Filter (single-WAN + segmented LAN):**

```
ISP ──► ether1 (WAN) ──► MikroTik RB750 ──► ether2 (Open LAN) ──► Unrestricted Host
                                  │
                                  └──► Bridge (ether3-5, Ponte de Segurança) ──► Protected LAN
```

**Load Balancer (dual-WAN):**

```
ISP 1 ──► ether1 (WAN1) ──┐
                            ├──► MikroTik RB750 ──► Bridge (ether3-5) ──► LAN
ISP 2 ──► ether2 (WAN2) ──┘
```

## Coding Conventions

### RouterOS Scripting Style

```routeros
# Variable declarations (local scope)
:local cont 0
:local qtdTotalLink 0

# Global variables
:global nomeEther

# Loops and conditions
:foreach p in=[/interface pppoe-client find] do={
  :local nomePppoe [/interface pppoe-client get $p value-name=name]
}

# Delay operations
:delay 2s

# String interpolation and path-based commands
/ip firewall nat add comment="Balance$nomePppoe" out-interface=$nomePppoe action=masquerade
```

### Key Patterns

- Use `:local` for function-scoped variables and `:global` for shared state across script blocks
- Iterate interfaces with `:foreach p in=[/interface ... find]`
- Use RouterOS path-based CLI commands (e.g., `/ip firewall nat add ...`)
- Comments use `#` prefix
- Firewall rules are ordered: whitelist first, then specific blocks, then reject-all last
- NAT masquerade for outbound traffic on each WAN interface
- PCC (`per-connection-classifier`) for distributing load across WAN links

### File Organization

- Place Layer 7 filtering configs (complete device configs) in `Layer7 Filter/`
- Place load balancing configs and scripts in `LoadBalancer/`
- Name device-specific config files as `<DeviceModel>.rsc` or `<DeviceModel>v<N>.rsc` for revisions
- Name automation scripts descriptively in ALL-CAPS (e.g., `AUTO-BALANCE.rsc`)

## CI/CD Pipeline

There is **no CI/CD automation** in this repository. All testing is manual on RouterOS hardware or a [MikroTik CHR](https://mikrotik.com/download) VM.

## Development Workflow

1. Fork and clone the repository
2. Create a branch: `git checkout -b feat/my-change`
3. Edit or create `.rsc` files in the appropriate directory (`Layer7 Filter/` or `LoadBalancer/`)
4. Validate by importing to a RouterOS test device or CHR VM
5. Verify configuration was applied correctly with print commands
6. Commit following the [commit conventions](https://github.com/rios0rios0/guide/wiki/Life-Cycle/Git-Flow)
7. Open a pull request against `main`

## Common Tasks

### Adding a New Device Configuration

1. Create `<DeviceModel>.rsc` (or `<DeviceModel>v<N>.rsc` for revisions) in the correct directory
2. Include a comment header identifying the hardware model, RouterOS version, and subnet range
3. Follow the firewall chain ordering convention (whitelist → specific blocks → reject-all)
4. Test the import on a matching or compatible RouterOS version
5. Update `README.md` project structure if adding a new file

### Modifying Layer 7 Rules

1. Edit the address list entries or L7 pattern definitions in the `.rsc` file
2. Validate regex patterns against RouterOS L7 protocol matcher syntax
3. Verify rule ordering in the firewall filter chain after import

### Modifying Load Balancer Rules

1. For `AUTO-BALANCE.rsc`: ensure the script cleans up existing rules before re-creating them
2. Test with both PPPoE-only, DHCP-only, and mixed WAN setups if possible
3. Verify `check-gateway=ping` failover behaves correctly after changes

### Troubleshooting

- **Import fails with syntax error**: verify RouterOS version compatibility; some commands differ between 6.36 and 6.47
- **Firewall rules not matching**: check rule ordering; RouterOS evaluates rules top-to-bottom and stops at first match
- **PCC load balancing not distributing traffic**: verify both WAN interfaces are up and routes are correctly tagged with routing marks
- **Layer 7 patterns not matching**: RouterOS L7 inspection only works on the first packets of a connection; ensure the pattern regex is valid Perl-compatible regex

## Prerequisites

- [MikroTik RouterOS](https://mikrotik.com/software) v6.36+ device or [CHR](https://mikrotik.com/download) virtual machine for testing
- [WinBox](https://mikrotik.com/download) v3.x+ or SSH client for device access
- A text editor with RouterOS scripting language support (e.g., [VS Code](https://code.visualstudio.com/) with a RouterOS syntax extension)
