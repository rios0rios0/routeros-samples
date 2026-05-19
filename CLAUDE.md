# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Collection of production-ready MikroTik RouterOS configuration scripts (`.rsc`) for Layer 7 content filtering and multi-WAN load balancing. No build system or package manager -- all scripts are standalone files interpreted by the RouterOS CLI.

## Validation

No automated linting or tests. Validate scripts manually on a RouterOS device or [CHR](https://mikrotik.com/download) VM:

```sh
scp my-script.rsc admin@<router-ip>:/
ssh admin@<router-ip>
/import file-name=my-script.rsc
```

Verify with: `/ip firewall filter print`, `/ip firewall mangle print`, `/ip route print`, `/ip firewall nat print`.

## Architecture

**Layer 7 Filter configs** (`Layer7 Filter/`): single-WAN with segmented LAN. `ether1` = WAN, `ether2` = open (unrestricted) output, `ether3-5` bridged as protected output. Firewall chain order matters: accept whitelisted destinations, reject HTTPS (443), reject push notifications (2195), reject Layer 7 matches, reject all remaining -- all with ICMP network-unreachable.

**Load Balancer configs** (`LoadBalancer/`): dual-WAN PCC-based load balancing. `AUTO-BALANCE.rsc` auto-discovers PPPoE and DHCP WAN interfaces, cleans up previous rules, and rebuilds mangle/route/NAT entries using `both-addresses` as the PCC classifier. `RB750Gr3.rsc` is a complete device config with the auto-balance script embedded.

## Conventions

- `:local` for function-scoped variables, `:global` for shared state across script blocks.
- Iterate interfaces with `:foreach p in=[/interface ... find]`.
- Firewall rules are order-dependent: whitelist first, specific blocks next, reject-all last.
- Device-specific configs: `<DeviceModel>.rsc` or `<DeviceModel>v<N>.rsc` for revisions.
- Automation scripts: ALL-CAPS naming (e.g., `AUTO-BALANCE.rsc`).
- Layer 7 configs go in `Layer7 Filter/`, load balancing configs in `LoadBalancer/`.

## CI

A `release.yaml` workflow creates releases automatically on push to `main`. No automated linting or testing.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for prerequisites, workflow, and commit conventions.
