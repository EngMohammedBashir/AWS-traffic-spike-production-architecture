# Project Progress

## Current Status

**Phase 3 — Load Balancing (in progress)**

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
- [x] Public route table created and public subnets associated
- [x] Private route table created and private subnets associated
- [x] NAT Gateway created and private default route configured
- [x] ALB security group (`ALB-SG`) created
- [x] Application security group (`App-SG`) created; HTTP/80 allowed from `ALB-SG` only
- [x] Corrected EC2 public-IP misconfiguration by recreating `web-server` privately
- [x] `web-server` running in `private-subnet-a` with no public IPv4 and `App-SG`
- [x] EC2 status checks passed
- [x] Target group `web-tg` created
- [x] Target type: Instance, HTTP:80, HTTP1, VPC `main-vpc`
- [x] Health check: HTTP `/`, traffic port, success code 200
- [x] `web-server` registered in `web-tg` on port 80
- [x] Application Load Balancer `web-alb` created
- [x] `web-alb` is internet-facing and IPv4
- [x] `web-alb` spans `public-subnet-a` (`us-east-1a`) and `public-subnet-b` (`us-east-1b`)
- [x] `web-alb` uses `ALB-SG`
- [x] Listener HTTP:80 forwards 100% to `web-tg`
- [ ] Wait for ALB state to become `Active`
- [ ] Verify target health becomes `Healthy`
- [ ] Test application through ALB DNS name

## Current Architecture

```text
Internet
   │
   ▼
web-alb (internet-facing)
   ├── public-subnet-a / us-east-1a
   └── public-subnet-b / us-east-1b
   │
   ▼
ALB-SG
   │ HTTP:80
   ▼
web-tg
   │
   ▼
App-SG
   │ HTTP:80 from ALB-SG only
   ▼
web-server
   └── private-subnet-a / us-east-1a
```

## Current Network Layout

```text
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
ALB-SG
   ├── TCP/80  from 0.0.0.0/0
   └── TCP/443 from 0.0.0.0/0
   ↓
App-SG
   └── TCP/80 from ALB-SG only
   ↓
private EC2
```

This keeps the application instance from accepting direct Internet web traffic. The ALB is the public entry point.

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
- User data installs/enables/starts Apache (`httpd`) and writes a simple test page with the instance hostname
- Status checks: passed

### Learning Incident — Public IP Misconfiguration

The first EC2 launch accidentally received an auto-assigned public IPv4 address even though it was placed in the private application subnet. The instance was terminated and recreated with public IP assignment disabled.

**Lesson:** A subnet being named or routed as private is not enough by itself. Public IPv4 assignment must also be disabled for the application instance, and ingress should be restricted with a dedicated application security group.

## Cost Warning

The NAT Gateway is active and billable. The Application Load Balancer is also now provisioned and billable. Both must be tracked and deleted during cleanup when the lab no longer needs them. Actual AWS billing data will be used later for cost documentation.

## Documentation Evidence

AWS Console screenshots have been captured locally for VPC/subnets, Internet Gateway, route tables, NAT routing, security groups, EC2 correction, target group creation, and ALB creation/provisioning. Screenshot files are not yet committed to GitHub.

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
- Created `ALB-SG` and `App-SG` with layered ingress.

### Session 4 — EC2 Bootstrap
- Prepared Amazon Linux 2023 Apache user data.
- First EC2 launch exposed an unintended public IPv4; instance terminated.
- Re-launched `web-server` in `private-subnet-a` with no public IPv4 and `App-SG`.
- Verified the replacement instance is Running and all status checks passed.

### Session 5 — Target Group & ALB
- Created `web-tg` using instance targets on HTTP port 80.
- Registered `web-server` in the target group.
- Created internet-facing `web-alb` across the two public subnets.
- Attached `ALB-SG` and configured HTTP:80 listener to forward to `web-tg`.
- ALB was still `Provisioning` immediately after creation, which is expected.

## Next Verified Step

Wait for `web-alb` to become `Active`, then open `web-tg` and verify `web-server` target health. If the target becomes `Healthy`, test the application through the ALB DNS name and capture the result.
