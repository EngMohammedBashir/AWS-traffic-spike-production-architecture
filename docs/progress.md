# Project Progress

## Current Status

**Phase 2 — Networking & Compute Preparation (in progress)**

- [x] Repository created
- [x] Project README initialized
- [x] Root account MFA enabled
- [x] AWS monthly cost budget created: `$10`
- [x] Budget alerts configured
- [x] IAM administrative user/group configured for routine console work
- [x] Project Region confirmed: `us-east-1` (N. Virginia)
- [x] Custom VPC created: `main-vpc` — `10.0.0.0/16`
- [x] Public subnet A: `10.0.0.0/21` — `us-east-1a`
- [x] Public subnet B: `10.0.8.0/21` — `us-east-1b`
- [x] Private subnet A: `10.0.16.0/21` — `us-east-1a`
- [x] Private subnet B: `10.0.24.0/21` — `us-east-1b`
- [x] Internet Gateway created and attached to `main-vpc`
- [x] Public route table created
- [x] Public route `0.0.0.0/0 -> Internet Gateway` configured
- [x] Both public subnets explicitly associated with public route table
- [x] Private route table created
- [x] Both private subnets explicitly associated with private route table
- [x] NAT Gateway created and reached `Available`
- [x] Private default route `0.0.0.0/0 -> NAT Gateway` configured
- [x] ALB security group (`alb-sg`) created
- [x] `alb-sg` inbound HTTP/80 from `0.0.0.0/0`
- [x] `alb-sg` inbound HTTPS/443 from `0.0.0.0/0`
- [x] Application security group (`app-sg`) created
- [x] `app-sg` inbound HTTP/80 restricted to source `alb-sg`
- [x] First EC2 launch attempt terminated after detecting unintended public IPv4 assignment
- [x] Replacement EC2 instance launched successfully in `private-subnet-a`
- [x] Replacement EC2 has no public IPv4 address
- [x] Replacement EC2 uses `App-SG`
- [x] Replacement EC2 reached all status checks passed (`3/3` in current console)
- [ ] Verify Apache bootstrap through an ALB target health check
- [ ] Create target group and Application Load Balancer

## Current Network Layout

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
main-vpc — 10.0.0.0/16
│
├── us-east-1a
│   ├── public-subnet-a   10.0.0.0/21
│   └── private-subnet-a  10.0.16.0/21
│        └── web-server   private-only EC2
│
└── us-east-1b
    ├── public-subnet-b   10.0.8.0/21
    └── private-subnet-b  10.0.24.0/21

Public route table:
  0.0.0.0/0 -> Internet Gateway

Private route table:
  0.0.0.0/0 -> NAT Gateway
```

## Security Design

```text
Internet
   ↓
alb-sg
   ├── TCP/80  from 0.0.0.0/0
   └── TCP/443 from 0.0.0.0/0
   ↓
app-sg
   └── TCP/80 from alb-sg only
   ↓
private EC2
```

This keeps the application instance from accepting direct Internet web traffic. The ALB will be the public entry point.

## EC2 Implementation

Current successful instance configuration:

- Name: `web-server`
- AMI: Amazon Linux 2023
- Instance type: `t3.micro`
- VPC: `main-vpc`
- Subnet: `private-subnet-a`
- Public IPv4: none
- Security Group: `App-SG`
- Key pair: `Web-Key`
- User data: installs/enables/starts Apache (`httpd`) and writes a simple test page with the instance hostname
- Status checks: passed

### Learning Incident — Public IP Misconfiguration

The first EC2 launch accidentally received an auto-assigned public IPv4 address even though it was placed in the private application subnet. The instance was terminated and recreated with public IP assignment disabled.

**Lesson:** A subnet being named or routed as "private" is not enough by itself. Public IPv4 assignment must also be disabled for the application instance, and ingress should be restricted with a dedicated application security group.

## Cost Warning

The NAT Gateway is active and billable. It incurs hourly charges while provisioned plus data-processing charges. It is currently required so the private EC2 instance can bootstrap packages from the Internet. It must be deleted during cleanup when no longer needed.

## Documentation Evidence

AWS Console screenshots have been captured locally for networking steps, NAT routing, security groups, the initial EC2 launch issue, and the corrected private EC2 launch. Screenshot files are not yet committed to GitHub.

## Rules

- One implementation step at a time.
- Do not move forward until the current step is verified.
- GitHub is the source of truth for project state.
- Sync meaningful completed steps to this repository as soon as they are verified.
- Record architecture/security/cost decisions and trade-offs.
- Record actual measurements; never invent performance, recovery, availability, or cost results.
- Track cleanup after every AWS lab session.
- Warn before creating resources that generate ongoing cost.
- Never commit passwords, access keys, account IDs, private keys, secret values, or other credentials.

## Session Log

### Session 0 — Repository Initialization
- Repository connected and initialized.

### Session 1 — AWS Account Guardrails
- Enabled root MFA.
- Created `$10` monthly AWS budget and alerts.
- Created safe daily IAM administrative access.

### Session 2 — VPC & Subnets
- Created `main-vpc` (`10.0.0.0/16`).
- Created two public and two private subnets across two Availability Zones.

### Session 3 — Routing, NAT & Security
- Created and attached the Internet Gateway.
- Created public/private route tables and explicit subnet associations.
- Routed public Internet traffic through the Internet Gateway.
- Created a NAT Gateway and routed private outbound traffic through it.
- Created `alb-sg` for future ALB ingress.
- Created `app-sg` with HTTP ingress from `alb-sg` only.

### Session 4 — EC2 Bootstrap
- Prepared Amazon Linux 2023 Apache user data.
- First EC2 launch exposed an unintended public IPv4; instance terminated.
- Re-launched `web-server` in `private-subnet-a` with no public IPv4 and `App-SG`.
- Verified the replacement instance is Running and all status checks passed.

## Next Verified Step

Create an HTTP target group for the private EC2 instance, register `web-server`, verify target health, then create the internet-facing Application Load Balancer across both public subnets using `alb-sg`.
