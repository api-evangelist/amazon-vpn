---
name: Audit AWS VPN inventory in a Region
description: >-
  Enumerate every Site-to-Site VPN connection, gateway and Client VPN endpoint in one
  Region read-only, and find degraded or orphaned resources.
api: openapi/amazon-vpn-aws-vpn-api-amazon-ec2-query-api-subset-api-openapi.yml
operations:
  - invokeVpnAction
actions:
  - DescribeVpnConnections
  - DescribeVpnGateways
  - DescribeCustomerGateways
  - DescribeClientVpnEndpoints
  - DescribeClientVpnConnections
generated: '2026-09-01'
method: generated
---

# Audit AWS VPN inventory in a Region

A read-only pass. Nothing here mutates state, so it is safe to run unattended — but it
is throttled, and the throttle is per Region per action.

## Steps

1. `Action=DescribeVpnConnections` — the Site-to-Site connections. **Unpaginated.**
   Narrow with filters rather than paging:
   `Filter.1.Name=state&Filter.1.Value.1=available`,
   `Filter.1.Name=customer-gateway-id&Filter.1.Value.1=cgw-…`, or
   `Filter.1.Name=tag:Owner&Filter.1.Value.1=TeamA`.
2. `Action=DescribeVpnGateways` — the virtual private gateways and their VPC attachments.
   Unattached gateways are the classic orphan.
3. `Action=DescribeCustomerGateways` — the on-premises anchors. A customer gateway with no
   connection referencing it is the second orphan class.
4. `Action=DescribeClientVpnEndpoints` — Client VPN endpoints. This one **is** paginated;
   follow `NextToken` until it is absent.
5. `Action=DescribeClientVpnConnections` per endpoint — active client sessions. Paginated.

## Reading the result

- **Both tunnels up** is the healthy state for a Site-to-Site connection. One tunnel up is
  a working but unprotected connection, and AWS raises a VPN single tunnel notification
  after an hour in a day.
- **Cross-check against quota**: 50 VPN connections, 5 virtual private gateways and 50
  customer gateways per Region by default.

## Rules

- **Pace the sweep.** Non-mutating actions get 100 burst / 20 per second per action per
  Region. A multi-Region audit multiplies the calls but not the buckets — each Region has
  its own.
- **There are no rate-limit headers**, so back off on `RequestLimitExceeded` rather than
  trying to stay under a published remaining count.
- **Do not call `TerminateClientVpnConnections` during an audit.** It appears next to the
  Describes in the action list, ends a live user session, and has no reversal.
- The one purpose-built MCP tool over this surface, `list_vpn_connections` in
  `awslabs.aws-network-mcp-server`, covers step 1 only. See
  `mcp/amazon-vpn-tool-crosswalk.yml`.
