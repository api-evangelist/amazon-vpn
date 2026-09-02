---
name: Create an AWS Site-to-Site VPN connection
description: >-
  Stand up an IPsec VPN between an on-premises network and an Amazon VPC or transit
  gateway, then read back the tunnel configuration needed to configure the customer
  gateway device.
api: openapi/amazon-vpn-aws-vpn-api-amazon-ec2-query-api-subset-api-openapi.yml
operations:
  - invokeVpnAction
actions:
  - CreateCustomerGateway
  - CreateVpnGateway
  - AttachVpnGateway
  - CreateVpnConnection
  - DescribeVpnConnections
  - EnableVgwRoutePropagation
generated: '2026-09-01'
method: generated
---

# Create an AWS Site-to-Site VPN connection

Every call is an HTTPS POST of form-encoded parameters to the regional Amazon EC2
endpoint (`https://ec2.{region}.amazonaws.com`) carrying `Action=<Name>` and
`Version=2016-11-15`, signed with AWS Signature Version 4. There is one path (`/`) and
one HTTP method. Responses are XML.

## Before you start

- Confirm the caller's IAM policy grants the `ec2:` permission for each action. Rehearse
  with `DryRun=true` — a permitted call returns the error code `DryRunOperation`, a denied
  one returns `UnauthorizedOperation`, and nothing is created either way.
- Pick the Region and stay in it. VPN resources, quotas and throttling buckets are all
  per-Region.
- Default quotas: 50 customer gateways, 5 virtual private gateways, 50 VPN connections
  per Region. Check `rate-limits/amazon-vpn-rate-limits.yml` before bulk work.

## Steps

1. **Create the customer gateway.** `Action=CreateCustomerGateway` with `Type=ipsec.1`,
   the on-premises device's `IpAddress`, and `BgpAsn` if the device runs BGP. Keep the
   returned `cgw-` id.
2. **Create the AWS-side gateway.** `Action=CreateVpnGateway` with `Type=ipsec.1`. Keep
   the `vgw-` id. Skip this step if the VPN will attach to a transit gateway instead.
3. **Attach it to the VPC.** `Action=AttachVpnGateway` with `VpnGatewayId` and `VpcId`.
   A virtual private gateway attaches to one VPC at a time. The inverse is
   `DetachVpnGateway`.
4. **Create the connection.** `Action=CreateVpnConnection` with `Type=ipsec.1`,
   `CustomerGatewayId`, and either `VpnGatewayId` or `TransitGatewayId`. Set
   `Options.StaticRoutesOnly=true` when the customer device does not run BGP.
   - This action is **idempotent by default** — repeating it does not return an error and
     does not create a second connection.
   - Use HTTPS. The response contains the tunnel pre-shared keys.
5. **Read back the configuration.** `Action=DescribeVpnConnections` with
   `VpnConnectionIds`. The `CustomerGatewayConfiguration` element carries the device
   configuration. This action is **unpaginated** — there is no `NextToken`; narrow the
   result with `Filter.1.Name=state&Filter.1.Value.1=available` instead.
6. **Propagate routes** (dynamic routing only). `Action=EnableVgwRoutePropagation` with
   `RouteTableId` and `GatewayId`. For static routing, add each prefix with
   `Action=CreateVpnConnectionRoute` instead.

## Rules

- **Poll, do not assume.** The connection returns before the tunnels come up. Re-call
  `DescribeVpnConnections` with an increasing sleep interval; there is no completion
  webhook.
- **Retry only on 5xx and on `RequestLimitExceeded`**, with exponential backoff and
  jitter. On a 4xx, fix the request first. See `errors/amazon-vpn-problem-types.yml`.
- **Handle the missing rate-limit headers.** There are none. `RequestLimitExceeded` is
  the only in-band signal, and the mutating bucket is 50 burst / 5 per second.
- **Capture `Response.RequestID` on every call**, success or failure. It is the only
  identifier AWS Support acts on.
- **Deletion is not reversible.** `DeleteVpnConnection` is the inverse of step 4, but
  recreating a connection issues new tunnel credentials and a new `vpn-` id, and the
  customer gateway device must be reconfigured. AWS publishes no restore window.
