# Interface VPC Endpoints — Simple Guide + Cheat Sheet

## What it is (30 seconds)

An **Interface VPC Endpoint (VPCE)** puts a private network interface (ENI) with a private IP in your subnets, so workloads can reach a service **without an internet gateway, NAT, VPN or peering**. It is powered by AWS PrivateLink.

Two common uses:

| Use case | Service you connect to | Example service name |
|---|---|---|
| **AWS service** | An AWS-managed service | `com.amazonaws.ap-southeast-2.ssm` |
| **Someone else's service** | A provider's VPC Endpoint Service (behind an NLB) | `com.amazonaws.vpce.ap-southeast-2.vpce-svc-0123456789abcdef0` |

The consumer-side steps are almost identical for both. Section 4 covers the extra provider-approval step.

```text
Workload ──► VPCE ENI (private IP, your subnet) ══ PrivateLink ══► Service
             └─ protected by your security group
```

---

## 1. Prerequisites

- [ ] VPC with **DNS resolution** and **DNS hostnames** both enabled (needed for Private DNS)
- [ ] Subnets for the endpoint ENIs (one per AZ you want resilience in; two or more recommended)
- [ ] A security group for the endpoint (inbound from your workloads)
- [ ] Permission to run `ec2:CreateVpcEndpoint`
- [ ] The service name (for AWS services, look it up with the CLI below)

---

## 2. Create an Interface Endpoint (Console)

1. **VPC console → Endpoints → Create endpoint**
2. **Type:**
   - AWS service → *AWS services*
   - Provider's service → *Endpoint services that use NLBs and GWLBs* (or *PrivateLink Ready partner services*)
3. **Service name:** pick from the list, or paste the provider's service name and click **Verify service**
4. **VPC:** select your VPC
5. **Subnets:** select one subnet per AZ
6. **Security group:** allow inbound **TCP 443** (or the service port) from your workloads
7. **Enable DNS name** (Private DNS): tick it if you want AWS to manage DNS for you (AWS services, or custom services where the provider has a verified private DNS name). **Leave it unticked if you manage DNS yourself with a custom Route 53 PHZ** (see "Private DNS vs your own PHZ" below)
8. **Policy:** leave as *Full access*, or restrict (see section 6)
9. **Create endpoint**

Wait until state is **Available**.

---

## 3. Create an Interface Endpoint (CLI)

```bash
aws ec2 create-vpc-endpoint \
  --vpc-endpoint-type Interface \
  --vpc-id vpc-0abc123 \
  --service-name com.amazonaws.ap-southeast-2.ssm \
  --subnet-ids subnet-0aaa subnet-0bbb \
  --security-group-ids sg-0abc123 \
  --private-dns-enabled
```

### Terraform

```hcl
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-2.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpce.id]
  private_dns_enabled = true
}
```

### Security group rule (endpoint SG)

| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | TCP | 443 | Workload security group (or VPC CIDR) |

Workload SG outbound must also allow 443 to the endpoint.

---

## 4. Connecting to a Provider's Endpoint Service (extra steps)

Same creation steps as above, plus:

**Consumer gives provider:** your AWS account ID (principal ARN `arn:aws:iam::<account-id>:root`)

**Provider does:**
1. Adds your principal to the Endpoint Service's **Allow principals**
2. If **Acceptance required** is on: **Endpoint connections → select pending → Accept**

**Consumer then:**
1. Creates the endpoint using the provider's service name
2. Waits for state `pendingAcceptance` → `Available`
3. Enables **Private DNS** only if the provider has configured and verified a private DNS name

```bash
# Provider: allow a consumer
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0123456789abcdef0 \
  --add-allowed-principals arn:aws:iam::123456789012:root

# Provider: accept a pending connection
aws ec2 accept-vpc-endpoint-connections \
  --service-id vpce-svc-0123456789abcdef0 \
  --vpc-endpoint-ids vpce-0abc123

# Consumer: turn on Private DNS after acceptance
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-0abc123 \
  --private-dns-enabled
```

> Allowed principals = *who may request*. Acceptance = *approve each request*. They are separate gates.

---

## 5. Test It

```bash
# Should return PRIVATE IPs (the endpoint ENIs), not public ones
nslookup ssm.ap-southeast-2.amazonaws.com
dig +short ssm.ap-southeast-2.amazonaws.com

# Reachability on port 443
curl -v https://ssm.ap-southeast-2.amazonaws.com
nc -zv <endpoint-private-ip> 443
```

If Private DNS is off, use the endpoint-specific name instead:
`vpce-0abc123-xxxx.ssm.ap-southeast-2.vpce.amazonaws.com`

### Private DNS vs your own PHZ

Pick **one** approach per VPC and service name:

| | AWS-managed Private DNS | Your own Route 53 PHZ |
|---|---|---|
| Endpoint setting | Private DNS **enabled** | Private DNS **disabled** |
| Who owns the DNS | AWS (hidden PHZ in the VPC) | You |
| Setup | Tick the box | Create a PHZ named after the service/hostname, add an **alias A record** (or CNAME) to the endpoint's `vpce-...` DNS name, associate the PHZ with each VPC |
| Good for | Simple, single VPC | Hub-and-spoke / centralised endpoints shared across many VPCs, custom hostnames, or a provider service with no verified private DNS name |

Notes for the PHZ approach:
- The VPC needs `enableDnsSupport` and `enableDnsHostnames` on for PHZ resolution to work.
- Name the PHZ after the service hostname (e.g. `ssm.ap-southeast-2.amazonaws.com`) with an alias A record to the regional endpoint DNS name, so SDKs and the CLI work without code changes.
- Don't run both for the same name in the same VPC: the AWS-managed zone and your PHZ will conflict. Keep Private DNS off.
- Associate the PHZ with every VPC that should resolve it (cross-account associations need an authorisation first).

---

## 6. Endpoint Policy (optional lockdown)

Controls what can be done *through* the endpoint. Default is full access.

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["ssm:GetParameter", "ssm:GetParameters"],
    "Resource": "*"
  }]
}
```

---

# Cheat Sheet

## Common AWS service names (replace region as needed)

| Service | Service name suffix | Notes |
|---|---|---|
| SSM | `ssm`, `ssmmessages`, `ec2messages` | All three needed for Session Manager |
| ECR | `ecr.api`, `ecr.dkr` | Also needs an **S3 Gateway endpoint** for image layers |
| CloudWatch Logs | `logs` | |
| Secrets Manager | `secretsmanager` | |
| STS | `sts` | Needed for IRSA / role assumption from private subnets |
| KMS | `kms` | |
| SQS / SNS | `sqs`, `sns` | |
| EKS API | `eks` | |
| EC2 API | `ec2` | |
| S3 | `s3` | Gateway endpoint is free and usually preferred; Interface endpoint is for on-prem / cross-region access |
| DynamoDB | `dynamodb` | Gateway endpoint also available |

Prefix: `com.amazonaws.ap-southeast-2.`

## Handy CLI

```bash
# List services available in the region
aws ec2 describe-vpc-endpoint-services --query 'ServiceNames' --output text

# Which AZs support a given service
aws ec2 describe-vpc-endpoint-services \
  --service-names com.amazonaws.ap-southeast-2.ssm \
  --query 'ServiceDetails[].AvailabilityZones'

# List your endpoints and their state
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[].[VpcEndpointId,ServiceName,State]' --output table

# Check VPC DNS attributes
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc123 --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc123 --attribute enableDnsHostnames

# Enable them if needed
aws ec2 modify-vpc-attribute --vpc-id vpc-0abc123 --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id vpc-0abc123 --enable-dns-hostnames

# Add a subnet / change SG on an existing endpoint
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc123 \
  --add-subnet-ids subnet-0ccc --add-security-group-ids sg-0def456

# Delete
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-0abc123
```

## Rules of thumb

- **Private DNS needs both VPC DNS attributes on.** Most "it resolves to a public IP" problems are this.
- **One Private-DNS endpoint per service per VPC.** A second one for the same service name will fail to enable Private DNS.
- **Security group is on the endpoint, not the service.** It must allow inbound from the workload.
- **Deploy in every AZ your workloads run in.** It avoids cross-AZ data charges and AZ-failure dependency.
- **Interface endpoints are regional to the VPC.** They don't automatically work from peered VPCs by DNS unless you share DNS (e.g. Route 53 Resolver / private hosted zone association); on-prem access needs DNS forwarding to the endpoint.
- **Cost:** an hourly charge per endpoint per AZ plus a per-GB data charge. Don't create one per workload; share them, and use S3/DynamoDB Gateway endpoints where possible (free).
- **Gateway vs Interface:** Gateway = route table entry, S3 and DynamoDB only, free. Interface = ENI + SG, almost everything else, paid.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| DNS returns public IP | Private DNS not enabled, or VPC DNS attributes off | Enable Private DNS; enable `enableDnsSupport` and `enableDnsHostnames` |
| Timeout / hang | Endpoint SG blocks inbound, or workload SG/NACL blocks outbound | Allow TCP 443 both ways |
| Endpoint stuck `pendingAcceptance` | Provider hasn't accepted | Provider accepts the connection request |
| Create fails: not allowed | Principal not on provider's allow list | Provider adds your account/role |
| Can't enable Private DNS | Another endpoint in the VPC already uses it, or provider's DNS name isn't verified | Remove the duplicate; provider verifies domain |
| Works in one AZ only | Endpoint has no ENI in the workload's AZ | Add a subnet in that AZ |
| `AccessDenied` through endpoint | Endpoint policy too restrictive | Review the endpoint policy |
| TLS error on custom name | Cert doesn't match the DNS name | Fix the provider's certificate |
| ECR pulls fail | Missing `ecr.dkr`, `ecr.api` or S3 gateway endpoint | Add all three |

## Quick decision flow

```text
Need private access to an AWS service?
 ├─ S3 / DynamoDB, same region ─► Gateway endpoint (free)
 └─ Anything else ─────────────► Interface endpoint
                                   ├─ Private DNS on  ─► use the normal service hostname
                                   └─ Private DNS off ─► use the vpce-xxxx... hostname

Connecting to another account's service?
 └─ Interface endpoint ─► provider allows your account ─► provider accepts (if required)
                          ─► enable Private DNS (if provider verified a DNS name)
```
