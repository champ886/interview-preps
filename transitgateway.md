# AWS Transit Gateway Cheat Sheet — Interview Cram Sheet

## 1. Mental model

- TGW is a **regional, managed Layer 3 router that lives outside every VPC**.
- VPCs, VPNs, Direct Connect, other TGWs, and SD-WAN appliances all "plug in" as **attachments**.
- No routing protocol to manage yourself — except where the attachment type requires BGP (VPN/DX/Connect). You configure route tables; AWS handles the forwarding plane.
- **Say this out loud:** TGW replaces full-mesh VPC peering with hub-and-spoke — and unlike peering, it's **transitive**. Spoke A reaches spoke B through the hub with no direct peering connection between them.

## 2. Core building blocks

| Component | What it is |
|---|---|
| Transit Gateway | The router itself — one per region (connect regions via TGW peering) |
| Attachment | The "plug" — VPC, VPN, Direct Connect Gateway, Peering, or Connect (SD-WAN/GRE) |
| TGW Route Table | A routing domain — defines what an attachment can reach |
| Association | Which route table an attachment uses to make **its own** routing decisions — exactly **one** per attachment |
| Propagation | Which route table(s) **receive** an attachment's CIDR as a route — many-to-many |

## 3. Sharing a TGW across accounts (RAM)

- **Standard pattern:** network/security account owns the TGW → creates `aws_ram_resource_share` → associates the TGW's ARN to it → adds a **principal association** per consuming account (or OU / whole Organization). RAM grants a *permission*, not a copy — the TGW still lives in, and bills to, the owning account.
- **Who calls the attach API:** the **consumer** account calls `CreateTransitGatewayVpcAttachment` against its own VPC, using the shared TGW's ID. The owner never touches the consumer's VPC. **Interview trap:** people assume the network team creates attachments for everyone — they don't, they just grant the right to attach.
- **Two separate gates, don't conflate them:**
  - RAM share = authorization. Never automatic — always an explicit resource share + principal association.
  - `auto_accept_shared_attachments` = a later, independent gate. `enable` → attachment requests from RAM-shared accounts auto-approve. `disable` → RAM still grants the *right* to attach, but each request sits in `pending-acceptance` until the owner manually approves — a checkpoint some regulated shops keep on purpose for change control.
- **Scaling the share:** target a single account ID (fine for a handful), an **OU ARN**, or the whole **Organization**. OU/Org sharing means new accounts get attach rights automatically — zero Terraform changes per account. Hardcoded account IDs don't scale past a handful.
- **A genuinely different mechanism — TGW peering:** connects two *already-existing* TGWs via a request/accept handshake (`aws_ec2_transit_gateway_peering_attachment` + accepter) — **no RAM involved.** It answers a different question: not "can this account attach to my TGW" but "how do two *separately-owned* TGWs talk to each other" — common for two business units, regions, or (post-M&A) two AWS Organizations that each already run their own hub. Peering = static routes only, never propagation (§8).
- **Going global — AWS Cloud WAN:** once you're juggling many TGWs, RAM shares, and peering connections, Cloud WAN is AWS's managed evolution — one policy-based "core network" replacing manual TGW+RAM+peering sprawl. Adoptable incrementally; it can federate with existing TGWs rather than requiring a rip-and-replace.
- **Confirmed in your repo:** `aws_lza_plus_eks_nat_lean_template` — TGW built under `provider = aws.security`; `aws_ram_resource_share` + `aws_ram_principal_association` for `dev_account_id` and `prod_account_id`; dev/prod attachments run under `provider = aws.dev` / `aws.prod` — the consuming accounts creating their own attachments.

**What the consumer side actually looks like** (your repo, `modules/transit-gateway/main.tf`):

```hcl
resource "aws_ec2_transit_gateway_vpc_attachment" "dev" {
  provider           = aws.dev                        # runs IN the consumer account
  transit_gateway_id = aws_ec2_transit_gateway.main.id # the shared TGW's ID
  vpc_id             = var.dev_vpc_id
  subnet_ids         = var.dev_private_subnet_ids

  dns_support                                     = "enable"
  transit_gateway_default_route_table_association = true
  transit_gateway_default_route_table_propagation = true

  depends_on = [
    aws_ram_principal_association.dev, # RAM grant must exist first
    aws_ram_resource_association.tgw,
  ]
}
```

No console magic beyond this: in the UI it's the same "Create Transit Gateway Attachment" screen the consumer account always sees — the shared TGW's ID just now appears in the dropdown because RAM made it visible. Pick it, pick your VPC/subnets, done.

## 4. Association vs. Propagation — the question they will ask

This is the single concept that unlocks everything else on this sheet.

| | Association | Propagation |
|---|---|---|
| Cardinality | 1 attachment → 1 route table | 1 attachment → many route tables (or none) |
| Controls | Which table this attachment's outbound traffic is evaluated against | Which tables *learn* this attachment's CIDR as a reachable route |
| Analogy | "Which table do I read from" | "Which tables get told about me" |

**Memory trigger:** Association = **who uses** the table. Propagation = **whose routes get written into** it (route entries, not traffic flow).

**Worked example** — Prod VPC, Dev VPC, Shared-Services VPC, three route tables (`rt-prod`, `rt-dev`, `rt-shared`), one attachment per table:

- Prod attachment → **associated** with `rt-prod`. Shared-Services attachment → **associated** with `rt-shared`.
- Every lookup checks the **destination** CIDR against the table of the attachment the packet **arrived on** — not the destination's table. Two-way Prod ↔ Shared-Services traffic needs routes in *both* tables:
  - Shared-Services' CIDR **propagated into `rt-prod`** — or a packet leaving Prod has nowhere to go.
  - Prod's CIDR **propagated into `rt-shared`** — or the reply has nowhere to go.
- Dev's CIDR is propagated only into `rt-dev` — `rt-dev` never learns Prod's or Shared-Services' CIDR either.
- **Result:** Prod ↔ Shared-Services works both ways. Prod ↔ Dev doesn't — neither table ever learned the other's CIDR.

**Packet walk-through** — Prod (10.1.0.0/16) opens a connection to Shared-Services (10.99.0.0/16):
1. SYN arrives at the TGW on the Prod attachment → checks `10.99.0.0/16` against `rt-prod` → forwards to Shared-Services.
2. SYN-ACK arrives at the TGW on the Shared-Services attachment → checks `10.1.0.0/16` against `rt-shared` → forwards back to Prod.

**Key nuances:**
- Miss one direction → an **asymmetric break**, not a clean failure. Forget Prod's CIDR in `rt-shared` and the SYN gets through fine, but the SYN-ACK has nowhere to route — the connection just times out. "One-way TGW connectivity" almost always means exactly this.
- Propagating Prod's CIDR *into `rt-prod` itself* only matters with multiple attachments on that table. With one Prod attachment alone, self-propagation does nothing — intra-VPC traffic never transits the TGW anyway.
- **Propagation vs. static routes:** propagation is just the automated version of "put a route in the table." A hand-added static route does the same job — which is exactly what peering attachments require, since they can't propagate at all (§8).
- **Trap:** every new TGW ships with a default route table, and every new attachment **auto-associates and auto-propagates** into it by default. Disable both at TGW creation, or every spoke reaches every spoke immediately — there is no isolation until you do.

## 5. How VPC isolation is actually done (step by step)

1. **Disable default route table association and propagation** on the TGW — the step people forget, and interviewers listen for it.
2. **Create one route table per isolation domain** (e.g., `rt-prod`, `rt-dev`, `rt-shared`, `rt-inspection`).
3. **Associate** each VPC attachment with exactly the one route table for its domain.
4. **Propagate selectively** — only push a VPC's CIDR into the table(s) of attachments that should legitimately reach it.
5. **Absence of a route = the isolation.** TGW route tables are default-deny: no route in, no path out. You don't "block" Dev from Prod — you just never let Prod's table learn about Dev.
6. **Blackhole routes for explicit deny** — a static route to `blackhole` for a hard, deliberate drop (decommissioned ranges, defense-in-depth) rather than relying on "route never existed."

## 6. Three isolation patterns you'll be asked to describe

| Pattern | Association | Propagation | Use case |
|---|---|---|---|
| **Full segmentation** | Each VPC → its own route table | No cross-propagation between domains | Hard separation, e.g. regulatory tenant isolation |
| **Hub-and-spoke centralized inspection** | Spokes → `rt-spokes`; firewall/inspection VPC → `rt-inspection` | Spokes propagate only into `rt-inspection`; `rt-spokes` gets only a default route (0.0.0.0/0) to the inspection VPC | All north-south (and often east-west) traffic forced through a firewall/NGFW |
| **Shared services** | Each spoke → its own route table; shared-services VPC → its own | Shared-services CIDR propagates into every spoke table; each spoke's CIDR propagates only into shared-services (not into each other) | Spokes need AD/DNS/logging but must not talk to each other |

- **Appliance Mode matters for the inspection pattern:** without it, TGW's flow hashing can send outbound and return legs of the same connection to *different* firewall ENIs/AZs, breaking stateful inspection. Appliance Mode pins a flow to one ENI for its lifetime.

## 7. New capability (GA July 2026): Policy-Based Routing (PBR)

Worth naming to sound current. PBR adds **policy tables** as an alternative to route tables:

- Rules match on **source CIDR, destination CIDR, source/destination port, and protocol** — not just destination like a normal route table.
- First-match-wins; no match = dropped (implicit deny, same philosophy as route tables).
- An attachment sits on **either** a route table **or** a policy table — never both.
- AWS's own pitch is exactly this topic: isolating prod/dev into separate routing domains and steering flows into inspection **without extra hops or VPCs**. No additional charge over standard TGW fees.
- **If asked "classic vs. current":** route tables + selective propagation is the traditional answer; PBR is the newer, more granular tool for the same isolation goal.

## 8. Attachment types — quick reference

| Attachment | Routing | Propagation notes |
|---|---|---|
| VPC | Static (TGW learns VPC CIDR automatically) | Standard — propagates normally |
| VPN (Site-to-Site) | Static or BGP | Dynamic routes only appear if BGP is used |
| Direct Connect Gateway | Static or BGP | DXGW itself is a separate resource wrapping the physical/virtual interface |
| Peering (intra- or inter-region) | **Static only** | **No propagation across a peering attachment** — each side manually adds static routes pointing at the peering attachment ID |
| Connect (SD-WAN/GRE) | BGP over GRE | Third-party SD-WAN appliances; supports ECMP across Connect peers |

## 9. TGW vs. VPC Peering vs. PrivateLink — decision table

| | TGW | VPC Peering | PrivateLink |
|---|---|---|---|
| Transitive? | Yes | No (full mesh required) | N/A — one-directional service exposure |
| Scale pattern | Hub-and-spoke, hundreds/thousands of VPCs | O(n²) connections as VPCs grow | One provider, many consumers |
| What it connects | Whole networks (all CIDRs, all ports) | Whole networks | A single service/application, not the network |
| Overlapping CIDRs | Not supported between VPCs that need to talk | Not supported | Not an issue — consumer never sees provider's CIDR |
| Typical driver | Central connectivity, segmentation, multi-account networking | Simple 1:1 VPC connectivity, lowest cost | Expose a specific service without network-level access |

## 10. Gotchas / interview traps

- **No security groups or NACLs at the TGW itself.** Route segmentation controls whether a *path* exists — it's not a firewall. Don't answer "how is this secured" with routing alone; SGs/NACLs or an inspection appliance still do the real access control.
- **Intra-VPC traffic never transits the TGW** — attaching a VPC doesn't change how its own subnets talk to each other.
- **One VPC = one attachment per TGW** — can't attach the same VPC twice.
- **Route quota is per-TGW, across all tables combined** — 10,000 combined dynamic + static routes per transit gateway (adjustable), not 10,000 per individual table.
- **Other default quotas:** 5,000 attachments per TGW (adjustable), 5 TGWs per VPC, ~20 route tables per TGW (adjustable), up to 100 Gbps per VPC attachment per AZ.
- **Cost model:** billed per attachment-hour + per-GB data processing fee for traffic crossing the TGW — good ammo for "why not just peer everything" answers.

## 11. 30-second verbal answer

*"Transit Gateway isolation is a routing-plane control, not a firewall. I disable default association and propagation on the TGW, then build one route table per isolation domain. Each VPC attachment associates with exactly one table — that's what it routes against — but propagation is many-to-many, so I selectively push each VPC's CIDR only into the tables of attachments that should reach it. If a route was never propagated into a table, that table's attachments simply can't reach it — that's the default-deny. Where I want an explicit, deliberate block rather than an absent route, I add a blackhole static route. For anything that needs actual inspection rather than just segmentation, I route spokes through a firewall VPC with Appliance Mode enabled so return traffic stays symmetric."*
