Design & IaC Interview Prep — On-Prem Monolith → Cloud-Native
Quick-recall cheat sheet (read this five minutes before you walk in)
The whole flow: ASK → BUILD → MOVE → PROVE

Step	Meaning	Anchor
Ask	Should this even move?	Section 3 — cloud-smart, not cloud-first
Build	Guardrails before app	3 SCPs: nothing root, nothing untracked, nothing off-region
Move	Connect → decide → cut over	"Move the app, not the data — yet"
Prove	The 8 NFRs, live	PRSS + TODD
The 8 NFRs in two four-letter clusters:

PRSS — "can it handle the pressure?" → Performance, Reliability, Scalability, Security
TODD — "would TODD sign off on shipping this?" → Testing, Observability, Deployment, Documentation
The one line that carries the hardest technical bit: "Move the app, not the data — yet." During a cutover, shift compute gradually (canary, weighted traffic) while both old and new instances keep using the same database/cache — don't split the data until the app side is fully proven.

How to use this: the brief says this is an open-ended discussion, not a right/wrong quiz. So treat the model answers below as talking points, not a script — the goal is to sound like you're reasoning live, then drop into specifics from aws_lza_plus_eks_nat_lean as evidence. I've grounded every answer in what's actually in the repo (account IDs, module names, real numbers) rather than generic best-practice fluff, because that specificity is what will separate you from someone who's only read a whitepaper.

The repo is a landing zone + EKS platform built cloud-native from scratch — it doesn't include the "should we even leave the data centre" mechanics. That's fine and worth being upfront about: it's strong evidence for where you'd land and how the platform is built, not for the decision to migrate in the first place. Section 3 below is the placement/TCO argument for why (and how much), Section 4 is the how, including how the repo's own network design (the Transit Gateway hub) extends naturally to a hybrid on-prem connection during any transition.

1. The scenario (use this to open the discussion, or expect something like it)
Meridian Financial Services is a mid-market Australian non-bank lender. Their core loan-origination-and-servicing platform, LoanHub, is a 12-year-old monolithic Java/Spring application running entirely on-premises — roughly 14 VMware VMs in Meridian's own Sydney data centre, behind a physical F5 load balancer, backed by one large on-prem PostgreSQL instance. There is no meaningful AWS footprint today beyond a shadow-IT proof-of-concept account a developer spun up eight months ago and never governed.

The forcing function: the data centre hosting contract and hardware refresh cycle are both up for renewal next year, and the quoted refresh cost is roughly what three years of cloud spend would run. Leadership wants out of the data centre, not just a modernised app inside it.
Deployments are manual, out-of-hours SSH + Ansible runs against physical hosts. A bad rollback means a multi-hour incident, and nobody can spin up a like-for-like test environment without provisioning more hardware. Releases happen roughly once a month.
Governance: the shadow-IT AWS account has no guardrails at all. A contractor disabled CloudTrail in it during a "cleanup" and nobody noticed for several days; an S3 bucket in the same account was found publicly exposed for 11 days.
Cost: Finance can't attribute the data-centre spend to a product line — it's one shared bill split by headcount. Capacity is sized for peak load year-round even though average utilisation sits around 8% CPU, because ordering more physical capacity takes months, so the team over-provisions defensively.
Observability: on-prem monitoring is Nagios plus default OS metrics, no tracing. The last major incident took 6 hours to resolve because nobody could isolate which component was failing.
Compliance: an upcoming audit will apply PCI-DSS-style controls to the card-payment workflow inside LoanHub, and data residency requires customer data to stay in Australia.
Leadership wants LoanHub fully off the data centre and running cloud-native — safe multiple-deploys-per-day, isolated environments, real cost attribution, real guardrails — within roughly 12 months, without a single big-bang cutover weekend.

Notice most pain points map to something specific your repo actually does, and the ones that don't (getting out of a data centre) are exactly the kind of "beyond a narrow technical solution" thinking the brief says they're assessing. Use both halves.

2. Opening / framing questions
Q: Walk me through your overall approach to this. Where do you start?

Before infrastructure, with a question that's easy to skip past: should this move at all, and how much of it? "Cloud-native by default" is a trap I'd rather catch myself in than have the panel catch me in — a flat, predictable workload on hardware you already own can genuinely be cheaper than the cloud equivalent, because cloud's price premium is the cost of elasticity, and if nothing's using the elasticity, you're paying for optionality you don't need. Section 3 is that argument in full; I raise it here because it changes what "step one" even means.

Once that placement call is made — and I'd expect some of LoanHub to come back "retain," not everything moves — step one for whatever does move is still the landing zone, for the same reason regardless of whether the workload starts on-prem or already in the cloud: a flat, ungoverned account is a governance problem before it's an architecture problem. The CloudTrail-disable and the exposed S3 bucket in this scenario both happened in an account with zero structure — that's not a LoanHub problem, it's an "anyone can do anything" problem. So: account/OU boundary and guardrails first, then a network big enough to host what's moving and connect back to whatever stays on-prem, and only then the application itself.

Q: This is a genuine on-prem migration, not a cloud-to-cloud refactor. What does that change about your plan?

Two things mainly. First, there's a hybrid period — LoanHub can't move overnight, so for some months (maybe most of the 12) the data centre and AWS both need to be live and able to reach each other, which means connectivity is a day-one workstream, not an afterthought. Second, I wouldn't decompose the whole monolith before moving anything — I'd get a landing spot in AWS proven first (which is what this repo is), then move LoanHub there largely as-is initially for the pieces that aren't ready to be re-architected, and decompose incrementally once it's actually running in AWS. Trying to fully re-platform and relocate and decompose in one motion is how a 12-month plan turns into a 3-year one.

Q: If you only had time to do one thing in month one, what would it be?

The account/OU boundary and the three SCPs — cheapest thing to build, mostly free, and it's the one change that immediately closes the two worst incidents in the brief. An SCP blocking cloudtrail:StopLogging/config:StopConfigurationRecorder would have stopped the contractor incident outright, and IAM Access Analyzer running org-wide from day one would have caught the exposed S3 bucket in hours, not 11 days — and critically, both of those apply to the existing shadow-IT account immediately, before a single EKS node exists.

3. Should LoanHub move at all? Workload placement & TCO
Q: "Cloud-smart," not "cloud-first" — do we really need to move this? If it's a predictable workload, isn't DC/colo cheaper and more controlled, with cloud TCO only paying off once you actually ramp elasticity over time?

That's the right question, and I'd rather answer it head-on than assume it away — this is exactly where I'd expect a strong panel to push, and I'd rather push on myself first. Cloud's price premium over owned hardware is the cost of elasticity and managed-service leverage. If a workload isn't using either, "migrate everything" is the wrong default, full stop.

Two numbers in the brief deserve more scrutiny than I gave them at first pass:

"8% average utilisation" is a mean, not a shape. Loan volume for a lender rarely sits flat — it moves with rate announcements, EOFY, marketing pushes. If peak-to-trough is something like 5:1, that's a genuine elasticity story cloud is good at. If the workload really is flat year-round (steady back-office batch, say), 8% average is just "we over-bought hardware," and right-sizing a smaller DC or colo footprint fixes that exact problem without leaving the building.
The lease/hardware-refresh forcing function tells me "you can't stay on this contract," not "you must go to public cloud." Renewing on right-sized hardware, or relocating to a colo facility on owned or leased gear, are both real options I'd want ruled out on their own economics — not skipped past because "cloud-native" was the word in the brief.
So my actual first deliverable isn't Terraform — it's a placement decision, workload by workload, and I'd genuinely expect part of LoanHub to come back "retain." Where I'd still land on "move," even holding myself to that scrutiny, is on the two things a fair TCO comparison doesn't erase: this is a single data centre with no real disaster-recovery story today, and a system that can't stand up an isolated, reproducible test environment without ordering hardware is a real liability walking into a PCI-style audit. Those are risk arguments, not "cloud is cheaper" arguments, and they're the ones I'd actually stand behind in the room rather than defaulting to a migration narrative because it's the more interesting build.

One more distinction I'd make explicitly, because it's easy to blur: cloud-native is an operating model, not a location. If part of LoanHub comes back "retain in DC/colo," I'd still push for containers, IaC, and GitOps on-prem instead of the manual Ansible-and-SSH pattern — that's not hypothetical for me, it's the exact stack I run in my own home lab on bare-metal Kubernetes (Rook+Ceph, ArgoCD, Prometheus/Grafana). Deployment safety and observability gains don't require leaving the building; only the elasticity and DR gains do.

Everything from here on in this document applies only to what actually earns its way onto cloud under that test — not to LoanHub wholesale.

4. Migration path: moving the pieces that earn their way to cloud
Q: Before any application work happens, how do you connect the data centre to AWS?

This is where I'd point straight at a specific piece of the repo's design rather than talk in the abstract. The Transit Gateway in aws_lza_plus_eks_nat_lean is currently scoped as a hub for internet egress only — Dev and Prod attach to it, Security's NAT Gateway is the only path out. That same hub-and-spoke shape extends naturally to hybrid connectivity: add a Direct Connect (or, faster to stand up, a Site-to-Site VPN as an interim) attachment to the same TGW, and the on-prem data centre becomes another spoke with routable access to the Security VPC and, through it, to Dev/Prod. I wouldn't redesign the network for the migration — I'd extend the hub I already have one attachment at a time, same as I'd extend it for a third workload account.

I'd run VPN in parallel with Direct Connect provisioning rather than wait — DX lead times can run 4-8+ weeks, and the migration timeline doesn't have slack to lose a month waiting on a circuit before any data can move.

Q: How do you decide what gets rehosted, replatformed, or rearchitected — do you rewrite the whole monolith?

No — a full rewrite is exactly the "big-bang" the brief says to avoid, and it's also the highest-risk path for a regulated lender under audit pressure. I'd assess LoanHub module by module against the standard migration patterns (rehost / replatform / refactor) rather than pick one strategy for the whole app:

Anything low-risk and rarely touched (reporting, batch jobs) — rehost onto EC2 or containerise as-is with minimal change, just to get it off physical hardware and inside the new landing zone quickly.
Anything that benefits immediately from managed services with little app change — replatform, e.g. the on-prem Postgres instance onto a managed RDS/Aurora Postgres, containerise the app tier onto EKS without splitting it up yet.
Anything actively causing pain today (the release cadence, the card-payment workflow under audit scope) — refactor into services, extracted one capability at a time, because those are the pieces where cloud-native actually pays for itself fastest.
Q: What does the actual cutover look like — how do you move traffic and data without a big-bang weekend?

Data first, traffic second, always with a way back:

Database: stand up the managed Postgres target early, and run continuous replication (AWS DMS or native logical replication) from the on-prem primary so the cloud copy stays current throughout the migration, not just during a cutover window. Cutover becomes "stop writes, let replication drain, flip the app's connection string" — minutes, not a weekend, and reversible by flipping it back if something's wrong.
Traffic: put a router in front of LoanHub — even something as simple as path-based rules on the existing F5, or an edge proxy — that can send specific request paths to the new AWS-hosted services while everything else still hits the on-prem monolith. That's the strangler fig pattern applied to a genuinely on-prem starting point: new/extracted capabilities go live in AWS behind a route rule, unmigrated capability keeps running exactly where it is, and the blast radius of any one cutover is one route rule, not the whole app.
Rollback: at every step the previous path still exists until the new one has proven itself, mirroring the same discipline as the Karpenter migration in the repo — nothing gets decommissioned until its replacement has run in parallel and been validated.
Q: For a given capability, would you switch it over all at once, or gradually?

Gradually — this is often called a "canary" rollout. Instead of flipping a feature from on-prem to cloud in one go, I'd send it a small slice of real traffic first, say 5%, and watch how it behaves. If it looks healthy I increase the slice a bit at a time — 25%, 50%, 100% — until everything's running in the cloud. If anything looks wrong at any point, I turn the slice back down to zero and nothing has broken.

I wouldn't build anything new to do the traffic-splitting — the F5 already sitting in front of LoanHub can split traffic by percentage between two destinations. I'd just add the new cloud version as a second destination and turn the dial. Less new infrastructure, and it's something the ops team already knows how to run.

Two things to watch for, and one of them is the part people usually get wrong:

Sessions — depends how LoanHub tracks logins today. If it already uses a shared cache like Redis that any server can read (not memory sitting on one specific server), treat that cache exactly like the database: leave it wherever it lives for now, let the cloud servers reach it over the same link, and the problem disappears — any server can handle any request. If it doesn't — each server keeps sessions in its own memory, common for an app this age — that's usually why the F5 has a "stickiness" setting on it today. A quick way to check: look at whether the F5 pool has a persistence rule configured for LoanHub right now. If it does, plan to add a shared cache as part of the move (can sit on either side, same principle as the database) rather than assume it's already solved.
The database — the important one. If both the on-prem version and the cloud version of a feature are live at once, and both can save changes, you now have two places that can disagree with each other. If the same loan record gets updated on both sides, which one's right? Keeping two live databases in sync is a real headache and a real risk.
The fix is to not move the database yet. Leave it on-prem, and have the new cloud version reach back and use that same on-prem database — over the private network link — exactly like the old version does. There's only ever one copy of the data, no matter which side handled the request, so nothing can get out of sync. The cloud version just has a slightly longer trip to fetch and save data during this phase — a bit slower, but nothing breaks. Only once a feature is fully running in the cloud (100% of its traffic, proven stable) do I move its data over too — by then it's a clean, one-time move instead of juggling two databases at once.

Before turning the dial at all, I'd set clear "stop and go back" rules — error rate, response time, and ideally a real business number like how many loan applications actually complete, not just server health. If any of those get worse, the traffic slice goes back to zero automatically, rather than relying on someone to notice and react. And for a regulated lender, I'd want each increase in the slice approved and logged, the same way the platform already requires sign-off before any infrastructure change — not something anyone can just dial up on the F5 late at night.

Worth saying plainly in the room: this is the same idea as the Karpenter migration already in the repo — bring the new thing up alongside the old one, watch it for a while, only then cut over, with an easy way back at every step. Same habit, just applied to traffic instead of servers.

Q: What's the organisational risk here, not just the technical one?

Skills. An ops team that's spent years running VMware and Ansible against physical hosts doesn't become a Kubernetes/GitOps team by osmosis, and if the platform lands but nobody on Meridian's side can operate it, I've just moved the single-point-of-failure from a data centre to me. I'd treat enablement as a real workstream — pairing, documentation the team actually reads (see Section 12), and deliberately not making the new platform so clever that it's illegible to the people inheriting it.

This is also where I'd bring up EKS Auto Mode rather than the self-managed setup in the repo. I built the repo self-managed on purpose — Karpenter, the ALB controller, EBS CSI as their own Terraform-managed pieces — to prove I understand what's actually happening underneath. But for Meridian, given the skills gap I've just described, I'd start Auto Mode: AWS runs node provisioning, the ALB controller, and storage for you, which cuts the Day-2 surface a VMware-background team has to learn on day one. I'd plan to graduate to self-managed once the platform team's Kubernetes maturity catches up — start simple for the people inheriting it, not to show off what I can build.

5. Performance
Q: How do you make sure this architecture actually performs well once workloads land on it, not just that it's "cloud-native" on paper?

Q: Where are the latency-sensitive paths in your design, and how did you protect them?

A few concrete decisions, not just principles:

The ALB Controller uses target-type: ip, so traffic goes ALB → pod IP directly, skipping the extra NodePort hop you'd get with target-type: instance.
AWS API calls that are on the hot path for scheduling and pulling images — ECR, STS, CloudWatch Logs, EC2 — go through interface VPC endpoints, not out through NAT/Transit Gateway. That's a latency win as a side effect of a cost decision: traffic to ecr.api/ecr.dkr/sts/logs/ec2 stays inside the VPC instead of hairpinning through the Security account's NAT Gateway.
gp3 is the default StorageClass instead of gp2 — better baseline IOPS at a lower price, which matters once Postgres and Prometheus are doing real I/O.
Karpenter's consolidateAfter: 30s means the platform reacts to load quickly without thrashing — it's not scaling on a 5-minute cron, it's reacting near-real-time to unscheduled pods.
The honest caveat I'd raise myself: during the hybrid period, LoanHub's performance is also bounded by whatever the Direct Connect/VPN link to the data centre gives you — any component still talking cross-site is now paying WAN latency it never paid on a LAN. I'd want that flagged early and factored into which modules move first.

Q: From a FinOps angle — is that VPC endpoint decision just a performance win, or does it save real money too?

Both, and it's the same decision doing double duty, which is worth calling out explicitly. NAT Gateway and Transit Gateway both charge per GB processed, on top of their hourly cost. Every AWS-service call — ECR pulls, STS, CloudWatch Logs, EC2 API — that goes through a VPC endpoint instead never touches NAT and never crosses the TGW hop at all, so that data-processing charge doesn't exist for that traffic. It's not "reduce NAT usage" for AWS-bound traffic — it's closer to zero.

What's left needing an actual internet path is genuinely external traffic — OS package updates, third-party APIs, anything outside AWS's network — and that still has to go somewhere. The FinOps move there isn't to remove NAT and TGW, it's to consolidate them: one shared NAT Gateway behind the TGW hub instead of one NAT per VPC, so the fixed hourly cost is paid once instead of three times over. So the precise framing: eliminate NAT/TGW entirely for AWS-service traffic via endpoints, and centralise what's left for genuine internet egress rather than trying to remove it — you can't remove the path for traffic that's actually leaving AWS's network.

6. Scalability
Q: The on-prem environment is sized for peak load year-round at 8% average utilisation because ordering hardware takes months. How does your design fix that?

Q: How does the platform itself scale — new environments, new teams, more clusters — not just pods?

Two different scaling problems:

Workload scaling — Karpenter replaces the "provision for worst case because a hardware order takes months" model entirely. The NodePool mixes spot and on-demand across t3/t3a medium/large, bounded at 16 vCPU / 32Gi, consolidating unused capacity after 30 seconds idle. That's the direct fix for the 8%-utilisation problem: capacity now tracks actual demand in minutes, not a procurement cycle.

Platform scaling — because SCPs are attached at the OU level, not the account level, onboarding a new environment is "create an account, drop it in the Workload OU" — it inherits all three guardrail policies automatically, zero extra Terraform. Same logic for IAM Access Analyzer: one org-wide analyzer (type = ORGANIZATION) covers every account that ever gets added. I designed it that way specifically so scaling the platform — which matters once other Meridian systems want to follow LoanHub off the data centre — doesn't become its own bottleneck.

7. Reliability
Q: What happens if a component fails? Walk me through your resilience story.

Q: You migrated onto Karpenter — how did you do that without risking an outage?

Reliability shows up at three layers in this repo, and I'd apply the same discipline to the app migration itself:

Network: per-AZ route tables on the VPC peering connections keep traffic within the same AZ instead of hairpinning cross-AZ. During the hybrid period I'd want the same thinking applied to the on-prem link — VPN as a backup path if Direct Connect is the primary, so a single circuit failure doesn't take out connectivity to a data centre that's still hosting part of the app.
State isolation: every environment has its own Terraform state file in S3 with DynamoDB locking — dev/vpc, prod/vpc, peering, transit-gateway, each eks-* component. Destroying or fixing one never risks another.
The Karpenter migration was deliberately phased, never destructive — my best concrete story here, and the direct template for the LoanHub cutover itself. Karpenter goes in alongside the existing managed node group. I taint the managed group (dedicated=managed:NoSchedule) so new pods stop landing there, then watch for Karpenter-provisioned nodes over a 24–48 hour soak. Only once that's stable do I scale the managed group to zero, and only after that do I remove it from Terraform. At every step there's a rollback: untaint the managed group and you're back to the previous state. That's exactly the pattern I'd run for strangling LoanHub off the data centre — nothing decommissioned until its replacement has proven itself running in parallel.
One honest gap I'd name myself: the repo only isolates dev and prod today, no staging. For LoanHub specifically I'd want a staging environment as part of the build — the canary/weighted cutover in Section 4 needs somewhere to rehearse against production-like data and load before it ever touches a real customer's loan record.

8. Security
Q: This scenario has two real security incidents — a disabled CloudTrail and an exposed S3 bucket sitting unnoticed for 11 days, both in the ungoverned shadow-IT account. How does your design prevent both, and how do you stop a repeat once LoanHub actually lands there?

Q: How do your pipelines authenticate to AWS — are there long-lived credentials anywhere, and what about the on-prem side?

Direct answers to both incidents, because I built for exactly this shape of failure:

CloudTrail/Config tamper → SCP 2 explicitly denies cloudtrail:StopLogging, cloudtrail:DeleteTrail, and the equivalent Config actions, attached at the OU level so it applies to every account underneath — including the shadow-IT account once it's brought under the org, and every account created afterward. SCP 1 additionally blocks the root user from doing anything, and blocks SCPs themselves from being modified or deleted. SCP 3 pins everything to ap-southeast-2 (covering the data-residency requirement in the brief) and blocks new security services from being disabled later. This is a preventive control — the action is denied before it happens, not caught afterward.
Silent public exposure → two layers, and worth being precise about which is which. Prevention: S3 Block Public Access enforced account-wide (and, better, at the org level via SCP) stops the bucket from ever becoming public in the first place — that's the control that would have actually stopped this incident rather than just shortened it. Detection, as a backstop: IAM Access Analyzer runs org-wide from the management account (type = ORGANIZATION), continuously scanning S3, IAM roles, KMS keys, Lambda, SQS, and Secrets Manager for anything reachable from outside the org — that's the external access analyzer, and it's what's built in the repo today. There's a second type worth naming as something I'd add: the unused access analyzer, which flags over-permissioned roles and unused permissions inside the org — the internal counterpart to the external one, catching privilege creep rather than public exposure.
Network segmentation → isolation isn't only at the account/IAM layer. Dev and Prod VPCs are never directly peered to each other in this design — each only peers with the shared Security VPC, so there's no route between them at all. A compromised workload in Dev has nowhere to go; reaching Prod isn't a permissions problem to work around, it structurally doesn't have a path. For the PCI-scoped card-payment workflow specifically, I'd go a layer further and isolate it into its own subnet and security group — or, once it's containerised, its own Kubernetes namespace with NetworkPolicies restricting east-west traffic. That's the same two-namespace PCI-DSS segmentation pattern I've already built on my own CloudCompass project — the point is to shrink the audit's actual scope, not just document a boundary and hope.
On credentials: GitHub Actions authenticates via OIDC, assuming a role in the management account — no long-lived AWS access key sitting in a repo secret. Application secrets (Postgres, pgAdmin) are created as Kubernetes secrets by Terraform, never committed to the gitops/ directory ArgoCD watches. On the on-prem side specifically, the DX/VPN connection and any replication credentials for the database migration are the new sensitive surface — I'd want those scoped as tightly as everything else here, not treated as a temporary exception because it's "just for the migration."

9. Deployment
Q: How do changes get from a developer's commit to production safely?

Q: Your pipeline has both an apply and a destroy path — how do you stop a destroy from accidentally running when someone meant to apply?

Two separate concerns, deployed differently on purpose:

Infrastructure deploys through GitHub Actions: 11 components (component-1 through component-9, plus 4b) are each a reusable workflow_call workflow, individually testable via workflow_dispatch. An orchestrator.yml chains them 1→9 using needs:, so the whole run shows as one job graph, with a GitHub Environments approval gate (shared/dev/prod) before every apply. Deliberately, destroy is not an option inside the apply orchestrator — it's a separate destroy-pipeline.yml with the action hardcoded to destroy and the components run in strict reverse (9→1). Running that workflow at all is the destructive decision. I originally chained things with workflow_run, which fires on any completion including a destroy — meaning a destroy could technically cascade into the next workflow's apply. Splitting into two structurally separate pipelines made that impossible rather than just "discouraged."

Application workloads deploy through GitOps, not Terraform. Terraform creates exactly one ArgoCD Application — root — pointed at gitops/. Everything else — apps.yaml (pgAdmin, Postgres) and platform.yaml (metrics-server) — is picked up by ArgoCD itself. Anything needing an IRSA role (Karpenter, Kubecost, the ALB Controller, EBS CSI) stays in Terraform, avoiding a chicken-and-egg problem provisioning IAM before the controller exists. Anything pure-Kubernetes goes to platform.yaml — a platform engineer can add a new app with a git push, no Terraform change. That split is exactly how LoanHub's extracted services would ship once they exist: new microservice, new ArgoCD Application, no infra pipeline in the critical path.

10. Testing
Q: What testing exists in this pipeline today, and — being honest — what's missing?

I'd lead with what's real rather than oversell it:

What exists: every apply is plan-then-approve, gated per environment. Post-deploy, there's a scripted verification pass — a busybox pod doing an outbound wget to prove the NAT/TGW egress path works, kubectl get applications to confirm ArgoCD apps are Synced/Healthy, kubectl get nodepool/ec2nodeclass to confirm Karpenter is ready. Those came from real failures I hit — an ALB 504 traced to a missing node security group ingress rule, and PVCs stuck Pending because there was no default StorageClass yet.

What's missing: automated policy-as-code testing on the SCPs and IAM boundaries (Conftest/OPA or Checkov against Terraform plan output, so a bad SCP change fails in CI, not in an account), and Terratest-style integration tests for the modules. For this specific migration I'd add one more category entirely: replication validation — automated row-count/checksum comparisons between the on-prem Postgres primary and the AWS replica throughout the migration window, run continuously, not just eyeballed once before cutover. Getting that wrong is the single highest-consequence mistake in a data migration for a lender.

11. Observability
Q: How would you know something's wrong before a customer tells you? What's your MTTR story compared to the on-prem 6 hours?

What's genuinely in place today: Kubecost with a bundled Prometheus gives per-pod cost visibility with 15-day retention, and the management account ships CloudWatch logs (90-day retention) plus AWS Config for compliance recording — a real foundation, and notably already better than the on-prem Nagios-plus-OS-metrics baseline.

What I'd build next, in two separate tracks:

Operational observability — extend the existing Prometheus into a full metrics/logging stack with Grafana on top for dashboards, plus CloudWatch Container Insights or an OTel collector for logs. This is exactly the pattern in my home lab, Prometheus + Grafana on bare-metal Kubernetes, so it's not new ground for me.

Security observability / SIEM — I'd start AWS-native before reaching for a third-party tool: GuardDuty for threat detection, Security Hub to aggregate findings across every account in the org, and Security Lake to normalise CloudTrail, Config, and VPC Flow Logs into one queryable store. Given the landing zone already centralises logging, that's close to zero extra pipeline to stand up, and it's the answer a panel expects from someone building AWS-native rather than bolting on an external SIEM by default. I'd only bring in something like Splunk if Meridian already has an enterprise SIEM contract and an analyst team built around it — otherwise it's a second system doing what Security Hub already does.

During the hybrid period specifically, I'd also want the DX/VPN link itself monitored and alerting — that path becomes a critical dependency the moment any component is calling across it, and it isn't something Nagios or CloudWatch alone will cover end-to-end.

12. Documentation
Q: How do you make sure this is operable by someone other than you — including a team that's never run Kubernetes?

Docs-as-code, living next to the thing they describe:

Every environment folder has its own README.md — purpose, variables, dependencies, how to run it standalone.
A top-level RUNBOOK.md covers deployment order, required secrets, post-deploy verification commands, and a known-gotchas table written from real incidents: the gavinbunney/kubectl vs hashicorp/kubectl provider mismatch, the GitHub Actions permissions vs secrets: inherit distinction, the ALB 504 root cause, the ArgoCD child-apps-not-appearing gotcha (files must live at the top level of gitops/, not one level deeper, because directory.recurse: false).
A "Key design decisions" table in the README pairs every non-obvious choice with its why — including why Transit Gateway exists alongside VPC peering instead of one or the other.
For this specific migration I'd add a document none of the above cover yet: a plain-language runbook for Meridian's own ops team, written assuming VMware/Ansible experience and zero Kubernetes background — because the skills gap named in Section 4 is a documentation problem as much as a training one.

13. Live showcase — what to actually pull up
The README architecture diagrams — organisation structure, then the network+EKS diagram, then the data-plane diagram showing which traffic goes via NAT/TGW vs which stays inside VPC endpoints.
The GitHub Actions tab — the orchestrator's job graph (1️⃣→9️⃣ as one visual DAG), and the environment approval gate in action if you have a run to show.
RUNBOOK.md's gotchas table — your best "critical thinking under pressure" evidence. Pick one and tell it as a 90-second story: symptom → root cause → fix.
A live (or recent) kubectl get applications -n argocd showing root/pgadmin/postgres/metrics-server all Synced/Healthy.
The "Key design decisions" table — fastest way to justify five decisions in five minutes without fumbling.
14. Trade-off / critical-thinking probes (expect these as follow-ups)
"Why not just lift-and-shift everything to EC2 first, then containerise later — isn't that lower risk?" It feels lower risk but it usually isn't lower effort — you re-platform the app twice instead of once, and you still carry the operational burden of a manually-deployed monolith for however long phase one takes, delaying the actual benefits (autoscaling, GitOps, guardrails) that are the point of the exercise. The middle ground I'd actually take is the strangler-fig approach from Section 4: the landing zone and hybrid connectivity go in regardless, but individual capabilities get built cloud-native from day one as they're extracted, while unmigrated pieces keep running on-prem behind a router. Nothing is lifted-and-shifted wholesale, and nothing requires a big-bang rewrite either.

"Why not a NAT Gateway per VPC — isn't a shared one a single point of failure?" Cost was the driver — one shared NAT vs. three saves real money at this scale — and the blast radius of a NAT outage is "egress is degraded," not "the platform is down," since in-cluster traffic and AWS API calls through VPC endpoints are unaffected. For a workload with a harder SLA on outbound connectivity I'd revisit this.

"Why Transit Gateway and peering — why not just pick one?" They solve different problems. Peering can't do transitive routing, so it can't centralise internet (or on-prem) egress through one NAT. TGW can, but costs roughly $36/attachment/month plus data — too expensive for VPC-to-VPC traffic peering already handles for free. Peering for direct VPC↔VPC, TGW for the thing peering structurally can't do — including, now, the hybrid DX/VPN attachment.

"You're using an ALB and Ingress — Gateway API is GA now. Would you use that instead?" Yes, if building this today. This repo predates the Gateway API GA / ingress-nginx deprecation, and I've since gone deep on the distinction updating my CKA study. For a platform that's about to host multiple extracted LoanHub services, a shared Gateway with per-team HTTPRoutes is a better fit than one ALB-backed Ingress per app — it separates the platform team's concern (Gateway, TLS, listeners) from the app team's concern (routes) more cleanly. I'm applying the same shift in my own CloudCompass refactor right now, moving it off ingress-nginx.

"Prod only has networking today — no EKS, no workloads. Isn't that a gap?" Deliberate scope boundary — dev fully proven end-to-end before spending on a second EKS control plane. For LoanHub, prod being stood up and proven quiet is the actual cutover gate: nothing about the migration plan changes, it's the same nine components promoted through the same orchestrator pattern with prod's own approval gate.

15. Bridging questions (if they probe beyond this repo)
"This is all platform — have you done this for application code too?" Yes — I'm mid-refactor on my own full-stack app, CloudCompass, going from a monolith to microservices on Kubernetes: containerising the frontend, working through ingress/gateway architecture, and building it out with ArgoCD ApplicationSets rather than one Application per service.

"What about a completely different cloud, or without Terraform at all?" I've built the same landing-zone pattern on Azure (azure-lza-lean) — hub-and-spoke across three subscriptions, AVNM, Azure Firewall, private DNS — plus a follow-on AKS repo reading its state. The principles (blast-radius isolation, guardrails at the OU/management-group level, centralised egress) carry across providers even though the primitives differ.

"Do you operate this stuff day-to-day, or is it just a portfolio piece?" Both — I run a bare-metal Kubernetes home lab (Rook+Ceph storage, ArgoCD, Prometheus/Grafana) for hands-on experimentation, and I'm CKA-track. This repo is where I prove out the AWS-managed-service side of the same platform-engineering instincts.

16. Self-check: first-pass summary vs refined
Useful to keep — this is what happened when the whole flow got compressed into a quick summary, and where the compression needed a correction. Good to skim once before the interview so the same slips don't happen live. All corrections below are already folded into the sections above.

Item	First pass	Correction
Monolith → cloud-native	Rationale left blank	The two things that survive scrutiny: DR gap (single DC, no failover) + audit risk (can't stand up a test env without ordering hardware) — not "cloud is cheaper," that's the Section 3 trap
EKS Auto Mode	Named as the starting point	Repo is self-managed by design (to prove the mechanics); Auto Mode is the right call for Meridian specifically, given the skills gap — say both, not just one
Connectivity (edge→core→compute→DB→storage)	"CDN/GA or CloudFront"	CloudFront, not GA — LoanHub is HTTP(S); Global Accelerator is for L4/UDP or multi-region anycast
Landing zone pillars	Governance-accounts-identity-networks-workloads-data	Correct as-is — maps almost 1:1 to AWS's own Landing Zone Accelerator pillars, worth naming the framework out loud
Security — public S3	"Deny public S3"	That's detection (Access Analyzer catches it after); the preventive control is S3 Block Public Access enforced account-wide
Security — IAM Access Analyzer	"internal/external"	Precisely: external access analyzer = public/cross-account exposure (what's built); unused access analyzer = over-permissioned roles inside the org (what's missing)
Security — segmentation	Not mentioned	Added: Dev/Prod VPCs never directly peered — no route exists, not just no permission; PCI-scoped workflow gets its own namespace/NetworkPolicies
Observability/SIEM	Splunk + AlienVault + Cloudera + USM	AlienVault is USM, same product — redundant; Cloudera isn't a SIEM play here; lead with GuardDuty + Security Hub + Security Lake (AWS-native, near-zero extra pipeline), bring in an external SIEM only if one's already under contract
FinOps — NAT/TGW	"Remove wherever possible"	More precise: eliminate entirely for AWS-service traffic via VPC endpoints; for genuine internet egress, consolidate to one shared NAT behind the TGW hub rather than removing it
Want to turn this into a live mock instead? Say the word and I'll ask one question at a time, you answer, and I'll push back the way a panel would.
