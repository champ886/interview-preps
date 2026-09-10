# Route 53 Resolver Endpoint — Cheat Sheet

## What It Is
- A DNS entry/exit point inside a VPC — **not** a PrivateLink-style service. No Endpoint Service, no `vpce-svc-xxxx`, no acceptance workflow.
- Two flavors: **Inbound** (on-prem → AWS DNS) and **Outbound** (AWS → on-prem DNS).
- Each endpoint deploys ENIs across the subnets/AZs you choose (2+ AZs recommended for resilience).

## Inbound Endpoint
```
On-prem DNS
    │  DNS query
    ▼
 DX / VPN
    │
    ▼
Resolver Inbound Endpoint (ENIs, e.g. 10.1.10.10 / 10.1.20.10)
    │
    ▼
Route 53 Resolver → Private Hosted Zone / Amazon DNS
```
Lets on-prem clients/DNS servers resolve records in your VPC (PHZs, EC2 private DNS, etc.) by querying the Inbound Endpoint IPs directly.

## Outbound Endpoint
```
EC2 / EKS
    │
    ▼
Amazon DNS / Resolver
    │
    ▼
Resolver Rule (forward rule: domain → target DNS IP)
    │
    ▼
Resolver Outbound Endpoint
    │
    ▼
DX / VPN → On-prem DNS
```
Lets AWS workloads resolve on-prem domain names. Requires a **Resolver Rule** mapping a domain to target DNS server IP(s).

## Sharing Model: RAM, Not Endpoint Services
- Resolver Rules can be **centrally owned** (e.g., a shared-services/DNS account) and shared to other accounts/VPCs via **AWS RAM**.
- RAM shares the *configuration* only — it does **not** provide network reachability.
- **Route 53 Profiles** are the newer way to bundle and share DNS config (PHZ associations, resolver rule associations) across VPCs/accounts — still config-sharing, not a PrivateLink-style endpoint.

## Without a Transit Gateway
```
Central DNS Account          VPC-A
──────────────────           ─────
Resolver Rule
     │ RAM share
     ▼
  Rule associated with VPC-A
```
Works only if VPC-A already has network reachability to wherever the Resolver Outbound Endpoint sits (e.g., rule + endpoint both local, or existing peering).

## With a Transit Gateway
```
                Central DNS VPC
               ┌────────────────┐
               │ Resolver        │
               │ Outbound EP     │
               └────────┬────────┘
                         │
                        TGW
                ┌────────┴────────┐
              VPC-A              VPC-B
```
Steps:
1. Create the Resolver Rule centrally.
2. Share it via RAM.
3. Associate it with VPC-A / VPC-B.
4. Confirm TGW + VPC route tables provide a path to the Resolver Endpoint's subnet.

TGW's only job here is **network path**, not DNS logic. No Endpoint Service involved either way.

## Worked Example: Forward Rule for `corp.internal`

Scenario: EKS workloads need to resolve on-prem AD domain `corp.internal`, forwarded to `10.0.0.5` and `10.0.0.6`.

### Option A — Non-Central (single account owns everything)
The Outbound Endpoint, the rule, and the consuming VPC all live in the same account. No RAM involved.

**CLI**
```bash
aws route53resolver create-resolver-rule \
  --creator-request-id "corp-internal-fwd-$(date +%s)" \
  --name "corp-internal-forward" \
  --rule-type FORWARD \
  --domain-name "corp.internal" \
  --resolver-endpoint-id rslvr-out-0123456789abcdef0 \
  --target-ips Ip=10.0.0.5,Port=53 Ip=10.0.0.6,Port=53

aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-0123456789abcdef0 \
  --vpc-id vpc-0123456789abcdef0
```

**Terraform**
```hcl
resource "aws_route53_resolver_rule" "corp_internal" {
  domain_name          = "corp.internal"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip { ip = "10.0.0.5", port = 53 }
  target_ip { ip = "10.0.0.6", port = 53 }
}

resource "aws_route53_resolver_rule_association" "corp_internal_vpc" {
  resolver_rule_id = aws_route53_resolver_rule.corp_internal.id
  vpc_id           = aws_vpc.workload.id
}
```
Good fit when: one account, one VPC (or a handful you're happy to manage rule-by-rule). No shared-services account, no RAM overhead.

### Option B — Central DNS Account (hub-and-spoke via RAM)
The Outbound Endpoint and the rule live in a central DNS/network account. Spoke accounts consume the rule via RAM and associate it with their own VPCs. Requires network reachability (TGW/peering) from each spoke VPC to the central DNS VPC.

**CLI — in the central DNS account**
```bash
# 1. Create the rule against the central account's outbound endpoint
aws route53resolver create-resolver-rule \
  --creator-request-id "corp-internal-fwd-$(date +%s)" \
  --name "corp-internal-forward" \
  --rule-type FORWARD \
  --domain-name "corp.internal" \
  --resolver-endpoint-id rslvr-out-0123456789abcdef0 \
  --target-ips Ip=10.0.0.5,Port=53 Ip=10.0.0.6,Port=53

# 2. Share it via RAM to spoke accounts (or an AWS Organizations OU)
aws ram create-resource-share \
  --name "corp-internal-rule-share" \
  --resource-arns arn:aws:route53resolver:ap-southeast-2:111111111111:resolver-rule/rslvr-rr-0123456789abcdef0 \
  --principals 222222222222 333333333333
```

**CLI — in each spoke account**
```bash
# Only needed if NOT sharing within AWS Organizations (Org-shared resources auto-associate)
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn <arn-from-invite>

# Associate the shared rule with the local VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-0123456789abcdef0 \
  --vpc-id vpc-0123456789abcdef0
```

**Terraform — central account**
```hcl
resource "aws_route53_resolver_rule" "corp_internal" {
  domain_name          = "corp.internal"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip { ip = "10.0.0.5", port = 53 }
  target_ip { ip = "10.0.0.6", port = 53 }
}

resource "aws_ram_resource_share" "corp_internal_share" {
  name                      = "corp-internal-rule-share"
  allow_external_principals = false
}

resource "aws_ram_resource_association" "corp_internal_rule" {
  resource_arn       = aws_route53_resolver_rule.corp_internal.arn
  resource_share_arn = aws_ram_resource_share.corp_internal_share.arn
}

resource "aws_ram_principal_association" "spoke_accounts" {
  for_each           = toset(["222222222222", "333333333333"])
  principal          = each.value
  resource_share_arn = aws_ram_resource_share.corp_internal_share.arn
}
```

**Terraform — each spoke account**
```hcl
resource "aws_route53_resolver_rule_association" "corp_internal_vpc" {
  resolver_rule_id = "rslvr-rr-0123456789abcdef0" # shared rule ID from central account
  vpc_id           = aws_vpc.spoke.id
}
```
Good fit when: multiple accounts/VPCs need the same forwarding rule, you want one team owning DNS config, or the account structure is already hub-and-spoke. Remember: RAM only shares the *rule* — the spoke VPC still needs a real network path (TGW attachment, peering, etc.) to reach the central Outbound Endpoint's subnet, or the forwarded query has nowhere to go.

## Key Gotchas
- RAM sharing ≠ connectivity. A shared rule pointing at a Resolver Endpoint in another VPC is useless without a route (TGW, peering, DX gateway, etc.) to reach it.
- Inbound/Outbound endpoints each need ENIs in **at least 2 AZs** for HA.
- Resolver Rules only fire for the domain(s) they're scoped to — unmatched queries fall through to standard VPC/Amazon DNS resolution.

## Comparison vs GWLB / PrivateLink
| | Route 53 Resolver | GWLB / PrivateLink |
|---|---|---|
| Provider creates a service? | No | Yes |
| Consumer creates an endpoint? | No | Yes |
| `vpce-svc-xxxx`? | No | Yes |
| Acceptance required? | No | Can be |
| Uses ENIs? | Resolver endpoint *is* ENIs | Endpoint creates ENI(s) |
| Main purpose | DNS forwarding/resolution | Private service connectivity |

**Mental shortcut:** Resolver endpoint = *"give DNS a network entry/exit point."*

**TL;DR:** Share rules via RAM; use TGW only when you need network connectivity between VPCs. No `vpce-svc` / Endpoint Service, ever.
