---
name: Change AWS Site-to-Site VPN tunnel options safely
description: >-
  Modify the cryptographic options, pre-shared key or certificate on one VPN tunnel,
  understanding that the change replaces the tunnel endpoint and drops traffic on it.
api: openapi/amazon-vpn-aws-vpn-api-amazon-ec2-query-api-subset-api-openapi.yml
operations:
  - invokeVpnAction
actions:
  - DescribeVpnConnections
  - ModifyVpnTunnelOptions
  - ModifyVpnTunnelCertificate
  - ModifyVpnConnectionOptions
generated: '2026-09-01'
method: generated
---

# Change AWS Site-to-Site VPN tunnel options safely

This is the most consequential flow on the VPN surface, because the API stores no prior
state and offers no undo.

## Capture the current state FIRST

Call `Action=DescribeVpnConnections` with `VpnConnectionIds` and persist the full
`Options` block before changing anything. The only way to reverse
`ModifyVpnTunnelOptions` is to re-apply the previous values, and only the caller has
them.

## Steps

1. **Read and store** the current tunnel options (above).
2. **Rehearse.** Re-issue the intended modify call with `DryRun=true`. Expect
   `DryRunOperation` if permitted.
3. **Modify one tunnel at a time.** `Action=ModifyVpnTunnelOptions` with
   `VpnConnectionId`, `VpnTunnelOutsideIpAddress` (which tunnel), and
   `TunnelOptions.*`. A VPN connection has exactly two tunnels; changing both at once
   removes the redundancy that keeps the connection up.
4. **Wait for the tunnel to come back before touching the second one.** Poll
   `DescribeVpnConnections` and check the tunnel's status. AWS emits a **Tunnel endpoint
   replacement notification** to the AWS Health Dashboard when the replacement completes
   — see `asyncapi/amazon-vpn-events.yml`.
5. **Repeat for the second tunnel.**

## Rules

- **A tunnel option change replaces the tunnel endpoint and drops traffic on that
  tunnel.** This is documented behaviour, not a fault.
- **`ModifyVpnTunnelOptions` is NOT idempotent** and takes no `ClientToken`. A blind
  retry after a timeout re-applies the change and can trigger a second endpoint
  replacement. Re-read state before retrying.
- **Certificate changes** use `Action=ModifyVpnTunnelCertificate` with the same
  one-tunnel-at-a-time discipline.
- **Connection-level options** — including tunnel bandwidth and the accepted algorithm
  list — use `Action=ModifyVpnConnectionOptions`.
- **Prefer scheduling.** If the connection has tunnel endpoint lifecycle control enabled,
  AWS-initiated maintenance can be accepted on your own schedule rather than applied for
  you. See `lifecycle/amazon-vpn-lifecycle.yml`.
- **A connection running on one tunnel for more than an hour in a day** produces a VPN
  single tunnel notification. That event is your check that the change did not leave the
  connection degraded.
