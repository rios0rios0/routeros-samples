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

Workflows in `.github/workflows/` all call reusable definitions in `rios0rios0/pipelines`. `release.yaml` creates releases automatically on push to `main`. `claude-review.yaml` runs an automated Claude code review on every pull request, and `claude-mention.yaml` responds to `@claude` mentions in issues and PRs. No automated linting or testing of the `.rsc` scripts themselves.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for prerequisites, workflow, and commit conventions.

<!-- chlog:start -->
## Changelog (chlog) — MANDATORY

If the repository you are working in uses chlog (a `.chlog.yaml` or `.chlog.yml`
config file, or a `.changes/` directory, exists at the project root), the
following is binding and ALWAYS applies: whenever you make ANY change, you MUST
create a changelog fragment as part of the same change — automatically, without
being asked, before committing.

- Do NOT edit CHANGELOG.md directly; it is generated from fragments.
- Create the fragment with:
  `chlog new --kind <Kind> --body "<imperative description>"`
- Valid kinds: Added, Changed, Deprecated, Removed, Fixed, Security
- Choose the kind that best matches the change (e.g., new feature → Added,
  bug fix → Fixed, behavior change → Changed, removal → Removed, security fix → Security).
- If the change is backward-INCOMPATIBLE with the public API (a breaking
  change), you MUST add the `--breaking` flag:
  `chlog new --kind <Kind> --breaking --body "<description>"`.
  This is the ONLY thing that triggers a major version bump — the kind alone
  never does (per SemVer, major = incompatible change). When unsure whether a
  change breaks compatibility, ask the user instead of guessing.
- Fragments are YAML files in `.changes/unreleased/`; stage them with your commit.
- `chlog check` fails the build when a fragment is missing — never skip it.
<!-- chlog:end -->
