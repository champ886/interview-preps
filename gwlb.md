# AWS Gateway Load Balancer (GWLB) — Cheat Sheet

## What It Is
- Layer 3/4 load balancer purpose-built for inserting third-party virtual network appliances (firewalls, IDS/IPS, DPI, WAF) transparently into a traffic path.
- Uses the **GENEVE** protocol (UDP port 6081) to encapsulate traffic between the GWLB and appliance targets.
- Same provider/consumer family as PrivateLink Interface Endpoints, but purpose-built for appliance traffic instead of API/service traffic.

## Core Components
| Component | Role |
|---|---|
| GWLB | The load balancer itself, sits in the "appliance/security VPC" |
| Target Group | Pool of appliance instances (firewalls, NVAs), often an Auto Scaling Group |
| Endpoint Service | Created by the provider on top of the GWLB — this is the `vpce-svc-xxxx` |
| GWLB Endpoint (GWLBe) | Created by the consumer VPC — the entry/exit point that redirects traffic into the GWLB pipeline |

## Traffic Flow (Typical Inspection Pattern)
```
Workload VPC                     Security VPC
─────────────                    ─────────────
Subnet route table                 GWLB
  0.0.0.0/0 → GWLBe                  │
        │                        Target Group
        ▼                        (firewall/NVA instances)
   GWLBe (ENI)  ──── GENEVE ────►     │
        ▲                             │
        └───────── GENEVE ◄───────────┘
   (inspected traffic returns to GWLBe, then continues to IGW/next hop)
```
- GWLBe acts as a **route table target** (like a NAT Gateway) — apps don't connect to it directly, traffic is redirected to it via route entries.

## Provider vs Consumer
- **Provider** (security/appliance VPC owner): builds the GWLB + target group, publishes it as an Endpoint Service.
- **Consumer** (workload VPC owner): creates a GWLB Endpoint attached to that Endpoint Service, then routes traffic to it.
- Endpoint Service **can** require connection acceptance, same as regular Interface Endpoints.

## Key Gotchas
- **Symmetric routing is mandatory** — both legs of a flow must hit the same appliance instance, or stateful inspection breaks.
- **Flow stickiness** — GWLB hashes on the 5-tuple by default; this must hold for stateful appliances to work.
- **Cross-zone load balancing is off by default** — turn it on if appliance capacity isn't in every AZ.
- Deploy GWLBe in **dedicated endpoint subnets**, separate from workload subnets, one per AZ.
- Health checks run against the target group like any standard ELB target group.
- Billed hourly (GWLB + GWLBe) plus per-GB data processing — factor this into cost models for high-throughput inspection VPCs.

## Comparison vs Route 53 Resolver Endpoint
| | GWLB / PrivateLink | Route 53 Resolver |
|---|---|---|
| Provider creates a service? | Yes | No |
| Consumer creates an endpoint? | Yes | No |
| `vpce-svc-xxxx`? | Yes | No |
| Acceptance required? | Can be | No |
| Uses ENIs? | Endpoint creates ENI(s) | Resolver endpoint *is* ENIs |
| Main purpose | Private service connectivity | DNS forwarding/resolution |

**Mental shortcut:** PrivateLink/GWLB = *"connect my VPC to someone else's service."*
