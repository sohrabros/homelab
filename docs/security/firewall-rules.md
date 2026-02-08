# Firewall Rules Design

## Philosophy

OPNsense firewall rules follow a **default deny** model. Every VLAN interface starts with an implicit deny-all rule. Only explicitly permitted traffic passes.

Rules are evaluated **top to bottom, first match wins.** This means deny rules must come before allow rules for the same scope.

## Key Rule Patterns

### IoT VLAN — The Most Important Rules

The IoT VLAN demonstrates the core security pattern:

| # | Action | Source | Destination | Port | Purpose |
|---|--------|--------|-------------|------|---------|
| 1 | Allow | IoT Net | Gateway Address | 53 | DNS to local resolver |
| 2 | **Block** | IoT Net | RFC1918 ranges | Any | **Block all internal access** |
| 3 | Allow | IoT Net | Any | 443 | HTTPS to internet |
| 4 | Block | IoT Net | Any | Any | Default deny (implicit) |

Rule #2 is the most important rule in the entire firewall. By blocking IoT devices from reaching ANY private IP range (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), a compromised IoT device cannot scan or attack the management network, trusted devices, or anything else on the LAN.

The DNS allow rule (Rule #1) must come before the RFC1918 block because the gateway IP is a private address. Without this exception, IoT devices couldn't resolve DNS.

### Guest VLAN

Similar to IoT but even more restrictive:

| # | Action | Source | Destination | Port | Purpose |
|---|--------|--------|-------------|------|---------|
| 1 | Allow | Guest Net | Gateway Address | 53 | DNS only |
| 2 | Block | Guest Net | RFC1918 | Any | No internal access |
| 3 | Allow | Guest Net | Any | 80, 443 | Web browsing only |
| 4 | Block | Guest Net | Any | Any | Default deny |

### Conduit VLAN

The most locked-down VLAN:

| # | Action | Source | Destination | Port | Purpose |
|---|--------|--------|-------------|------|---------|
| 1 | Block | Conduit Net | RFC1918 | Any | No internal access whatsoever |
| 2 | Allow | Conduit Net | Any | Any | Internet access for proxy |
| 3 | Block | Conduit Net | Any | Any | Default deny |

No DNS exception needed — the Conduit container uses public DNS directly for its proxy function.

### Trusted VLAN

The most permissive internal VLAN:

| # | Action | Source | Destination | Port | Purpose |
|---|--------|--------|-------------|------|---------|
| 1 | Allow | Trusted Net | Management Net | Any | Admin access to infrastructure |
| 2 | Allow | Trusted Net | Any | Any | Full internet access |

### Management VLAN

Infrastructure-only. No user devices should be here:

| # | Action | Source | Destination | Port | Purpose |
|---|--------|--------|-------------|------|---------|
| 1 | Allow | Management Net | Any | Any | Infrastructure needs full access |

## Anti-Lockout Considerations

**Always keep a backup access path.** Before modifying firewall rules:

1. Verify WireGuard VPN works from your phone
2. Confirm you can reach OPNsense through the VPN
3. Test the change
4. If locked out, VPN in and revert

The WireGuard VLAN has its own firewall rules allowing access to Management, which means VPN access is independent of any changes to the Trusted or other VLAN rules.
