# AWS Direct Connect & DXGW Cheat Sheet

Nov 9, 2026 · @Champ

## Mental model

One DX connection carries several VIFs, and each VIF's type fixes where it ends: private to VPCs, transit to TGWs, public to AWS public services.

&#91;embedded content: VIF types and where they end · 3 VIFs, 1 DXGW\]

The DXGW sits between private or transit VIFs and the regional gateways behind them, which is how one connection reaches many Regions. A private VIF can also skip it and attach to a single VGW in its own Region.

**Easy way to remember: the DXGW is the front door, the TGW is the hallways.**

- **Front door (DXGW):** which TGWs your DX connection can reach, and what gets advertised back to on-prem.
- **Hallways (TGW):** VPC attachments, route tables and VPC-to-VPC traffic.
- **TGW peering** is only for VPC-to-VPC between Regions. On-prem reaches each Region's TGW through the same DXGW, and the DXGW does not route between those TGWs.

## Building blocks

A DX connection is only a physical wire; the VIF decides what it reaches, and the DXGW is what lets a VIF reach beyond its own region.

| Component | What it is | Scope |
| --- | --- | --- |
| DX connection | Physical port (dedicated, or hosted through a partner) cross-connected at a DX location | One DX location, one parent region |
| VIF | A VLAN (802.1Q) plus a BGP session riding on the connection; its type fixes what it can reach | Lives on one connection |
| VGW | VPC-side gateway; one VGW serves exactly one VPC | Regional |
| TGW | Regional hub with many VPC attachments (up to 5,000 by default) | Regional |
| DXGW | Broker that associates VIFs with VGWs or TGWs in any region or account (not China) | Global, not tied to a region |

## Attachment versus association

AWS says a VIF is attached to a DXGW, and a DXGW is associated with a VGW or TGW.

| Link | AWS term | What it means |
| --- | --- | --- |
| VIF to DXGW | Attachment | The VIF terminates on the DXGW; the API lists these as gateway attachments |
| DXGW to VGW or TGW | Association | A permission with rules: an allowed-prefixes list sets what AWS advertises to on-prem, and a cross-account link needs a proposal the other account accepts |

AWS does not explain the naming; the useful reading is that an association is a control-plane grant, not a plug-in. From the TGW side, a DXGW appears as an attachment of type Direct Connect gateway, which you associate with a TGW route table.

## VIF types

The VIF type decides the target: private reaches VPCs, transit reaches TGWs, public reaches AWS public services.

| VIF type | Reaches | DXGW | Notes |
| --- | --- | --- | --- |
| Private | A VPC through its VGW, or up to 20 VGWs through a DXGW | Optional | Private IPs. Amazon side uses the VGW's ASN, or the DXGW's ASN once attached to one |
| Transit | One or more TGWs, or a Cloud WAN core network | Required | Works on any dedicated or hosted connection of any speed; up to 4 per dedicated connection |
| Public | AWS public services (S3, API endpoints) on public IPs, any Region except China | No | You advertise 1 to 1,000 public IPv4 or IPv6 prefixes you own; AWS does not re-advertise them outside AWS |

Jumbo frames: a private VIF is 1500 or 9001 bytes, a transit VIF 1500 or 8500 (default 1500). Raising the MTU can update the underlying connection and drop all its VIFs for up to 30 seconds.

## Connectivity patterns

There are four valid paths; only a transit VIF reaches a TGW, and it always goes through a DXGW.

| Pattern | Path | Use when | Ceiling |
| --- | --- | --- | --- |
| Direct | Private VIF → VGW → VPC | One VPC in the same Region as the VIF | One VPC per VGW |
| Multi-VPC | Private VIF → DXGW → VGWs → VPCs | A handful of VPCs across Regions or accounts | 20 VGWs per DXGW |
| Hub | Transit VIF → DXGW → TGWs → VPC and VPN attachments | An enterprise estate with many VPCs | 6 TGWs per DXGW |
| Public | Public VIF → AWS public endpoints | S3 or public APIs without crossing the internet | 1,000 prefixes advertised |

## Routing rules

Three rules cover most design questions: no VPC-to-VPC through a DXGW, no on-prem-to-on-prem unless SiteLink is on, and BGP prefix caps that drop the session.

- **No VPC to VPC.** VGWs behind one DXGW cannot reach each other through it; use a TGW or peering for that.
- **On-prem to on-prem needs SiteLink.** It is off by default. Enable it on private or transit VIFs attached to a common DXGW and traffic takes the shortest path between Direct Connect locations without touching a Region. It is not supported on a private VIF attached straight to a VGW or on a public VIF, not available in GovCloud or China, billed separately, and it breaks if your router advertises the same route on several VIFs.
- **Path preference.** Without SiteLink, AWS prefers the DX location tied to the traffic's own Region even when another path has a shorter AS path. With SiteLink, the shorter AS path wins regardless of Region.
- **ASNs.** Your ASN must differ from the VGW or DXGW ASN on the VIF. A TGW and a DXGW sharing an ASN (for example both on the default 64512) makes the association fail.
- **Prefix caps.** On-prem to AWS, a BGP session defaults to 100 IPv4 plus 100 IPv6 routes (up to 1,000 each with prefix controls); exceed it and the session goes idle and reports DOWN. AWS to on-prem through a TGW, the cap is 200 prefixes combined.
- **BFD.** Asynchronous BFD is on for every VIF from the AWS side but does nothing until you configure it on your router.

## BGP peering

The BGP session runs between your router and the AWS Direct Connect router at the DX location, over the peer IPs set on the VIF.

- **AWS side:** presents the DXGW's ASN (the Amazon side ASN chosen when the DXGW is created). A private VIF attached straight to a VGW presents the VGW's ASN instead.
- **Your side:** presents your own ASN, public or private, and it cannot match the DXGW's.
- **Transit VIF:** still peers on the DXGW's ASN, never the TGW's.
- **Sessions:** each VIF carries one BGP session per address family, IPv4, IPv6, or both.

## Key quotas

The two limits that decide VGW-versus-TGW design are 20 VGWs and 6 TGWs per DXGW, and neither can be raised.

| Item | Limit | Raisable |
| --- | --- | --- |
| VGWs per DXGW | 20 | No |
| TGWs per DXGW | 6 | No |
| DXGWs per TGW | 20 | No |
| VIFs (private or transit) per DXGW | 30 | No |
| DXGWs per account | 200 | Via SA or TAM |
| VIFs per dedicated connection or LAG | 51 total: 50 private or public, plus up to 4 transit counted within it | No |
| VIFs per hosted connection | 1 (any type) | No |
| Active connections per DX location, Region and account | 10 | Via SA or TAM |
| Connections per LAG | 4 (2 at 100 Gbps) | Fixed |
| LAGs per Region | 10 | Via SA or TAM |
| On-prem to AWS routes per BGP session (private or transit VIF) | 100 IPv4 + 100 IPv6 | Up to 1,000 each with prefix controls |
| AWS to on-prem prefixes per TGW (transit VIF) | 200 combined | Via SA or TAM |
| Public VIF routes per BGP session | 1,000 | No |

Port speeds are 1, 10, 100 and 400 Gbps over single-mode fiber. A TGW holds 5,000 attachments by default (adjustable), which is why the hub pattern scales and the VGW pattern does not.

## Resiliency

Resiliency comes from how many dedicated connections you order and where they land, not from the DXGW.

| Model | Layout | Survives | SLA |
| --- | --- | --- | --- |
| Maximum | Separate connections on separate devices in more than one location | Device, connectivity and whole-location failure | 99.99% |
| High | Two single connections to multiple locations | Fiber cut or device failure; helps against location failure | 99.9% |
| Development and test | Separate connections on separate devices in one location | Device failure only | None stated |

The Resiliency Toolkit wizard keeps redundant connections at the same speed and stops you landing two on the same Direct Connect device. A Site-to-Site VPN to the same gateway is the usual backup path.

## One-liners and gotchas

If you remember one thing: VIFs, VGWs and TGWs are regional, and the DXGW is the global piece that lets one VIF reach across them.

- A transit VIF can only land on a DXGW, never straight on a TGW. A private VIF can land on a VGW or on a DXGW.
- Choose the VGW path for a handful of VPCs (20 per DXGW, hard cap) and the TGW path beyond that.
- A DXGW does not route VPC to VPC; a TGW does.
- On-prem to on-prem over the AWS backbone is SiteLink, not default DXGW behaviour.
- TGW and DXGW ASNs must differ, and so must your ASN and the DXGW's.
- Advertise more than 100 prefixes per address family from on-prem without raised limits and the BGP session drops.
- A hosted connection carries exactly one VIF.
- DX is private, not encrypted; add MACsec or an IPsec VPN if you need encryption.

## Sources

- [Direct Connect quotas](https://docs.aws.amazon.com/directconnect/latest/UserGuide/limits.html)
- [Direct Connect virtual interfaces and hosted virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
- [AWS Direct Connect Resiliency Toolkit](https://docs.aws.amazon.com/directconnect/latest/UserGuide/resiliency_toolkit.html)
- [What is Direct Connect?](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)
- [AWS Transit Gateway quotas](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas)
- [AWS Direct Connect locations](https://aws.amazon.com/directconnect/locations)
- [Routing scenarios with Direct Connect SiteLink](https://docs.aws.amazon.com/architecture-diagrams/latest/routing-scenarios-with-aws-direct-connect-sitelink/routing-scenarios-with-aws-direct-connect-sitelink.html)
