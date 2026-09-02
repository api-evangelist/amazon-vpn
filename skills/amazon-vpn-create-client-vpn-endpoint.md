---
name: Create and authorize an AWS Client VPN endpoint
description: >-
  Provision a managed OpenVPN-based remote-access endpoint, associate it with a VPC
  subnet, and grant client access to a destination network.
api: openapi/amazon-vpn-aws-vpn-api-amazon-ec2-query-api-subset-api-openapi.yml
operations:
  - invokeVpnAction
actions:
  - CreateClientVpnEndpoint
  - AssociateClientVpnTargetNetwork
  - AuthorizeClientVpnIngress
  - CreateClientVpnRoute
  - DescribeClientVpnEndpoints
generated: '2026-09-01'
method: generated
---

# Create and authorize an AWS Client VPN endpoint

An endpoint that exists but is neither associated nor authorized accepts no traffic.
All three steps are required.

## Before you start

- Have a server certificate in AWS Certificate Manager. For mutual authentication you
  also need at least one client certificate; AWS documents generating both with the
  OpenVPN easy-rsa utility.
- Every action in this flow accepts `ClientToken` — a unique, case-sensitive string of up
  to 64 ASCII characters. **Use it.** Without it a retried create makes a second resource.
- `AuthorizeClientVpnIngress` and `CreateClientVpnRoute` have their own throttling
  buckets of 5 burst / 2 per second — an order of magnitude tighter than the 50/5
  mutating default. Pace bulk authorization work accordingly.

## Steps

1. **Create the endpoint.** `Action=CreateClientVpnEndpoint` with
   `ClientCidrBlock` (must not overlap the VPC CIDR), `ServerCertificateArn`,
   `AuthenticationOptions.N` and `ConnectionLogOptions`. Pass `ClientToken`. Keep the
   returned `cvpn-endpoint-` id.
2. **Associate a target network.** `Action=AssociateClientVpnTargetNetwork` with
   `ClientVpnEndpointId` and `SubnetId`. This is what gives the endpoint a path into the
   VPC. Pass `ClientToken`. The inverse is `DisassociateClientVpnTargetNetwork`.
3. **Authorize ingress.** `Action=AuthorizeClientVpnIngress` with
   `ClientVpnEndpointId`, `TargetNetworkCidr`, and either `AuthorizeAllGroups=true` or an
   `AccessGroupId`. Pass `ClientToken`. The inverse is `RevokeClientVpnIngress`.
4. **Add routes for anything outside the associated subnet's VPC.**
   `Action=CreateClientVpnRoute` with `ClientVpnEndpointId`, `DestinationCidrBlock` and
   `TargetVpcSubnetId`. Pass `ClientToken`. The inverse is `DeleteClientVpnRoute`.
5. **Verify.** `Action=DescribeClientVpnEndpoints` with `ClientVpnEndpointIds`. Unlike
   the Site-to-Site Describe actions, the Client VPN Describes do support
   `MaxResults`/`NextToken` — page them.

## Rules

- **Reuse a `ClientToken` only for a true retry of the same call.** Same token with
  different parameters fails with `IdempotentParameterMismatch`.
- **Rehearse destructive steps with `DryRun=true`.** It checks permissions only, not
  parameter validity or resource state.
- **Every step here is reversible**, and each inverse is named above. That is not true of
  `TerminateClientVpnConnections`, which ends a live client session with no undo.
- Errors and remediation: `errors/amazon-vpn-problem-types.yml`. Conventions:
  `conventions/amazon-vpn-conventions.yml`.
