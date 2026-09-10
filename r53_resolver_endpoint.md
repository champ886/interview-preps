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
