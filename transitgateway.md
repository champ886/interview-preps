# AWS Transit Gateway Cheat Sheet — Interview Cram Sheet

## 1. Mental model

TGW is a **regional, managed Layer 3 router that lives outside every VPC**. VPCs, VPNs, Direct Connect, other TGWs, and SD-WAN appliances all "plug into" it as **attachments**. It doesn't run a routing protocol you manage (except where the attachment type requires BGP, e.g. VPN/DX/Connect) — you configure route tables, and AWS handles the forwarding plane.

Key distinction to say out loud: **TGW replaces full-mesh VPC peering with a hub-and-spoke model, and unlike peering, it's transitive** — spoke A can reach spoke B through the hub without a direct peering connection between them.

## 2. Core building blocks

| Component | What it is |
|---|---|
| Transit Gateway | The router itself — one per region (connect regions via TGW peering) |
| Attachment | The "plug" — VPC, VPN, Direct Connect Gateway, Peering, or Connect (SD-WAN/GRE) |
| TGW Route Table | A routing domain — defines what an attachment can reach |
| Association | Which route table an attachment uses to make **its own** routing decisions — exactly **one** per attachment |
| Propagation | Which route table(s) **receive** an attachment's CIDR as a route — many-to-many |

## 3. Sharing a TGW across accounts (RAM)

**Standard pattern:** the network/security account creates and owns the TGW. It creates an `aws_ram_resource_share`, associates the TGW's ARN to that share, then adds a **principal association** per consuming account (or per OU/the whole Organization). RAM sharing grants a permission, not a resource copy — the TGW still lives in, and is billed to, the owning account.

**Who calls the attach API matters:** once shared, the *consuming* account is the one that calls `CreateTransitGatewayVpcAttachment` against its own VPC, using the shared TGW's ID. The owning account never touches the consumer's VPC. This is a common interview trap — people assume the network team creates attachments on everyone's behalf; they don't, they just grant the right to attach.

**Scaling the share — individual accounts vs. OU/Organization:** `aws_ram_principal_association` can target a single account ID (fine for a handful of accounts), an **Organizational Unit ARN**, or the whole **Organization**. Sharing to an OU means any account that lands in that OU automatically gets attach rights — no Terraform change needed per new account. Hardcoding individual account IDs works at small scale but doesn't scale past a handful.

**A genuinely different mechanism — TGW peering:** peering connects two *already-existing* TGWs together (each typically already owned by its own account/region) via a request/accept handshake (`aws_ec2_transit_gateway_peering_attachment` + an accepter resource) — **it doesn't use RAM at all.** This solves a different problem than sharing: RAM answers "how does another account attach *its VPC* to *my* TGW"; peering answers "how do two *separately owned* TGWs talk to each other" — common when two business units, regions, or (post-M&A) two AWS Organizations each already run their own hub. Peering attachments only support static routes, never propagation (see §8).

**If this needs to go global — AWS Cloud WAN:** once you're juggling many TGWs, many RAM shares, and many peering connections across regions, Cloud WAN is AWS's managed evolution — a single policy-based "core network" with segments that replaces manual TGW+RAM+peering sprawl. It's adoptable incrementally — Cloud WAN can peer with and federate existing TGWs rather than requiring a rip-and-replace.

**Confirmed in your repo:** this is exactly what `aws_lza_plus_eks_nat_lean_template` does — TGW built under `provider = aws.security`, an `aws_ram_resource_share` plus `aws_ram_principal_association` for `dev_account_id` and `prod_account_id`, and the dev/prod attachment resources run under `provider = aws.dev` / `aws.prod` — the consuming accounts creating their own attachments, exactly as described above.

## 4. Association vs. Propagation — the question they will ask

This is the single concept that unlocks everything else on this sheet.

| | Association | Propagation |
|---|---|---|
| Cardinality | 1 attachment → 1 route table | 1 attachment → many route tables (or none) |
| Controls | Which table this attachment's outbound traffic is evaluated against | Which tables *learn* this attachment's CIDR as a reachable route |
| Analogy | "Which table do I read from" | "Which tables get told about me" |

**Memory trigger:** Association = **who uses** the route table (to route their own outbound traffic). Propagation = **whose routes get written into** it (as entries other attachments can then match on — not traffic flow, just route entries).

**Worked example** — Prod VPC, Dev VPC, Shared-Services VPC, three route tables (`rt-prod`, `rt-dev`, `rt-shared`), one attachment per table:

- Prod attachment → **associated** with `rt-prod`. Shared-Services attachment → **associated** with `rt-shared`.
- Every TGW routing decision looks up the **destination** CIDR in the route table of the attachment the packet **arrived on** — not the destination's table. So two-way Prod ↔ Shared-Services traffic needs routes in *both* tables:
  - Shared-Services' CIDR **propagated into `rt-prod`** — otherwise a packet leaving Prod toward Shared-Services has nowhere to go.
  - Prod's CIDR **propagated into `rt-shared`** — otherwise the reply (or anything Shared-Services initiates) has nowhere to go.
- Dev's CIDR is propagated only into `rt-dev`, and `rt-dev` never learns Prod's or Shared-Services' CIDR either.
- Result: Prod ↔ Shared-Services works both ways. Prod ↔ Dev doesn't, because neither table ever learned the other's CIDR.

**Packet walk-through** — Prod (10.1.0.0/16) opens a connection to Shared-Services (10.99.0.0/16):
1. SYN arrives at the TGW on the Prod attachment → TGW checks `10.99.0.0/16` against `rt-prod` → forwards to Shared-Services.
2. SYN-ACK arrives at the TGW on the Shared-Services attachment → TGW checks `10.1.0.0/16` against `rt-shared` → forwards back to Prod.

Miss one direction and you get an asymmetric break, not a clean failure. Forget to propagate Prod's CIDR into `rt-shared` and the SYN gets through fine, but the SYN-ACK has nowhere to route — the connection just times out. "One-way TGW connectivity" as a symptom almost always means a CIDR propagated in one direction and not the other — a good troubleshooting story to have ready.

**Nuance:** propagating Prod's CIDR *into `rt-prod` itself* only matters if more than one attachment shares that table (e.g. two Prod VPCs both associated with `rt-prod` needing to reach each other). With a single Prod attachment on its own table, that self-propagation does nothing — intra-VPC traffic never transits the TGW anyway.

**Propagation vs. static routes:** propagation is just the automated version of "put a route in the table." A hand-added static route pointing a CIDR at an attachment achieves the identical result — which is exactly what peering attachments require, since they don't support propagation at all (see §8).

**Trap to flag in the interview:** every new TGW ships with a **default route table**, and by default every new attachment **auto-associates and auto-propagates** into it. If you don't turn those two defaults off at TGW creation, every spoke can reach every other spoke immediately — there is no isolation until you deliberately disable default association/propagation and build dedicated tables.

## 5. How VPC isolation is actually done (step by step)

1. **Disable default route table association and propagation** on the TGW (a creation-time setting — this is the step people forget, and interviewers listen for it).
2. **Create one route table per isolation domain** (e.g., `rt-prod`, `rt-dev`, `rt-shared`, `rt-inspection`).
3. **Associate** each VPC attachment with exactly the one route table that represents its domain.
4. **Propagate selectively** — only push a VPC's CIDR into the route table(s) of attachments that should legitimately reach it.
5. **Absence of a route is the isolation mechanism.** TGW route tables are default-deny: if a destination CIDR was never propagated (or statically added) into the table an attachment is associated with, the TGW drops the packet. You don't need to "block" Dev from Prod — you just never let Prod's route table learn about Dev.
6. **Blackhole routes for explicit deny.** Add a static route to a CIDR with target `blackhole` when you want a hard, explicit drop regardless of what would otherwise be reachable — typically for decommissioned ranges or a defense-in-depth statement rather than relying purely on "route never existed."

## 6. Three isolation patterns you'll be asked to describe

| Pattern | Association | Propagation | Use case |
|---|---|---|---|
| **Full segmentation** | Each VPC → its own route table | No cross-propagation between domains | Hard separation, e.g. regulatory tenant isolation |
| **Hub-and-spoke centralized inspection** | Spokes → `rt-spokes`; firewall/inspection VPC → `rt-inspection` | Spokes propagate only into `rt-inspection`; `rt-spokes` gets only a default route (0.0.0.0/0) pointing at the inspection VPC | All north-south (and often east-west) traffic forced through a firewall/NGFW appliance |
| **Shared services** | Each spoke → its own route table; shared-services VPC → its own | Shared-services CIDR propagates into every spoke table; each spoke's CIDR propagates only into the shared-services table (not into each other) | Spokes need AD/DNS/logging but must not talk to each other |

For the inspection pattern specifically: pair it with **Appliance Mode** on the VPC attachment to the firewall VPC. Without it, TGW's flow hashing can send the outbound and return leg of the same connection to *different* firewall ENIs/AZs, breaking stateful inspection. Appliance Mode pins a flow to one ENI for its lifetime.

## 7. New capability (GA July 2026): Policy-Based Routing (PBR)

Worth naming if you want to sound current. PBR adds **policy tables** as an alternative to route tables:

- A policy table is an ordered list of rules matching on **source CIDR, destination CIDR, source/destination port, and protocol** — not just destination like a normal route table.
- First-match-wins; if nothing matches, the packet is dropped (implicit deny, same philosophy as route tables).
- An attachment is associated with **either** a route table **or** a policy table — never both.
- AWS's own pitch for it is exactly this topic: isolating prod/dev into separate routing domains and steering specific flows into inspection **without adding extra hops or VPCs**. No additional charge over standard TGW fees.

If asked "how would you do this today vs. how AWS is evolving it" — route tables + selective propagation is the classic answer; PBR is the newer, more granular tool for the same isolation goal.

## 8. Attachment types — quick reference

| Attachment | Routing | Propagation notes |
|---|---|---|
| VPC | Static (TGW learns VPC CIDR automatically) | Standard — propagates normally |
| VPN (Site-to-Site) | Static or BGP | Dynamic routes only appear if BGP is used |
| Direct Connect Gateway | Static or BGP | DXGW itself is a separate resource wrapping the physical/virtual interface |
| Peering (intra- or inter-region) | **Static only** | **No propagation across a peering attachment** — each side manually adds static routes pointing at the peering attachment ID |
| Connect (SD-WAN/GRE) | BGP over GRE | Used for third-party SD-WAN appliances; supports ECMP across Connect peers |

## 9. TGW vs. VPC Peering vs. PrivateLink — decision table

| | TGW | VPC Peering | PrivateLink |
|---|---|---|---|
| Transitive? | Yes | No (full mesh required) | N/A — one-directional service exposure |
| Scale pattern | Hub-and-spoke, hundreds/thousands of VPCs | O(n²) connections as VPCs grow | One provider, many consumers |
| What it connects | Whole networks (all CIDRs, all ports) | Whole networks | A single service/application, not the network |
| Overlapping CIDRs | Not supported between VPCs that need to talk | Not supported | Not an issue — consumer never sees provider's CIDR |
| Typical driver | Central connectivity, segmentation, multi-account networking | Simple 1:1 VPC connectivity, lowest cost | Expose a specific service without network-level access |

## 10. Gotchas / interview traps

- **TGW has no security groups or NACLs of its own.** Route table segmentation controls whether a *path* exists — it is not a firewall. Don't answer "how do you secure this" with routing alone; SGs/NACLs at the VPC layer or an inspection appliance still do the actual access control.
- **Intra-VPC traffic never transits the TGW** — attaching a VPC doesn't change how subnets inside that VPC talk to each other.
- **One VPC = one attachment per TGW** (can't attach the same VPC twice to the same TGW).
- **Total route quota is per-TGW, across all route tables combined** — 10,000 combined dynamic + static routes per transit gateway (adjustable via support), not 10,000 per individual table.
- Default quotas worth knowing: 5,000 attachments per TGW (adjustable), 5 TGWs per VPC, ~20 route tables per TGW by default (adjustable), up to 100 Gbps per VPC attachment per AZ.
- **Cost model:** billed per attachment-hour plus a per-GB data processing fee for traffic that crosses the TGW — factor this into "why not just peer everything" answers.

## 11. 30-second verbal answer

*"Transit Gateway isolation is a routing-plane control, not a firewall. I disable default association and propagation on the TGW, then build one route table per isolation domain. Each VPC attachment associates with exactly one table — that's what it routes against — but propagation is many-to-many, so I selectively push each VPC's CIDR only into the tables of attachments that should reach it. If a route was never propagated into a table, that table's attachments simply can't reach it — that's the default-deny. Where I want an explicit, deliberate block rather than an absent route, I add a blackhole static route. For anything that needs actual inspection rather than just segmentation, I route spokes through a firewall VPC with Appliance Mode enabled so return traffic stays symmetric."*
