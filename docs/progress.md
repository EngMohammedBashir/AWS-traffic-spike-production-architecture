# Project Progress

## Current Status

**Phase 4 — Auto Scaling Preparation (in progress)**

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
- [x] Public and private route tables configured
- [x] NAT Gateway created and private outbound route configured
- [x] `ALB-SG` and `App-SG` created with layered ingress
- [x] `web-server` launched privately with no public IPv4
- [x] EC2 status checks passed
- [x] `web-tg` created and `web-server` registered on HTTP/80
- [x] `web-alb` created across both public subnets
- [x] HTTP:80 listener forwards 100% to `web-tg`
- [x] `web-alb` reached `Active`
- [x] `web-server` reached `Healthy` in `web-tg`
- [x] End-to-end HTTP browser test through ALB succeeded
- [x] Launch Template created: `web-launch-template`
- [x] Launch Template version: `1` (default/latest)
- [x] AMI: Amazon Linux 2023 x86_64
- [x] Instance type: `t3.micro`
- [x] Key pair: `Web-Key`
- [x] Security group: `App-SG`
- [x] Subnet intentionally omitted from Launch Template for ASG placement control
- [x] User data included to install/start Apache and generate hostname test page
- [x] Plain Bash user data used; pre-Base64-encoded option left disabled
- [ ] Create Auto Scaling Group using both private subnets
- [ ] Attach Auto Scaling Group to `web-tg`
- [ ] Verify ASG-created instances bootstrap and become Healthy
- [ ] Test scaling/replacement behavior

## Current Architecture

```text
Internet
   |
   v
web-alb (internet-facing)
   ├── public-subnet-a / us-east-1a
   └── public-subnet-b / us-east-1b
   |
   v
ALB-SG
   |
   v
web-tg
   |
   v
App-SG
   |
   v
Private application instances
   ^
   |
web-launch-template
   |
   +--> future Auto Scaling Group across private-subnet-a + private-subnet-b
```

## Verified End-to-End Result

External HTTP request through the ALB returned:

```text
AWS Traffic Spike Project
Web server is running successfully.
Hostname: ip-10-0-16-137.ec2.internal
```

This verified the complete path:

```text
Browser -> web-alb -> web-tg -> App-SG -> private EC2 -> Apache
```

The initial HTTPS attempt failed because HTTPS/443 is not configured on the ALB yet; repeating the request over the configured HTTP/80 listener succeeded.

## Launch Template Details

`web-launch-template` is the reusable server definition for the next Auto Scaling phase.

Configuration:

- AMI: Amazon Linux 2023 x86_64
- Instance type: `t3.micro`
- Key pair: `Web-Key`
- Security group: `App-SG`
- Subnet: omitted intentionally
- User data: automated Apache bootstrap

Subnet placement is intentionally controlled by the Auto Scaling Group so the same template can launch instances into multiple private subnets/AZs.

## User Data Purpose

The bootstrap script automatically:

1. Updates packages.
2. Installs Apache (`httpd`).
3. Enables Apache for future boots.
4. Starts Apache immediately.
5. Reads the instance hostname.
6. Creates `/var/www/html/index.html` with a simple project page and hostname.

This matters because an Auto Scaling Group must be able to launch usable application instances without manual login/configuration. A newly created EC2 instance should become application-ready and eventually pass the target-group health check automatically.

## Cost Warning

NAT Gateway and Application Load Balancer are active billable resources. Auto Scaling will also create additional EC2/EBS usage once enabled. Cleanup remains mandatory after testing, and actual AWS billing data will be documented rather than estimated as observed fact.

## Documentation Evidence

Screenshots have been captured locally for key networking, EC2, target-group, ALB, end-to-end browser, and Launch Template milestones. Screenshot files are not yet committed to GitHub.

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
- Created `main-vpc` and four subnets across two Availability Zones.

### Session 3 — Routing, NAT & Security
- Created IGW, route tables, NAT Gateway, `ALB-SG`, and `App-SG`.

### Session 4 — EC2 Bootstrap
- Corrected first-launch public-IP mistake.
- Re-launched `web-server` privately and verified status checks.

### Session 5 — Target Group & ALB
- Created `web-tg` and internet-facing `web-alb`.
- Verified ALB `Active`, target `Healthy`, and successful browser response over HTTP.

### Session 6 — Launch Template
- Created `web-launch-template` version 1.
- Reused Amazon Linux 2023, `t3.micro`, `Web-Key`, and `App-SG`.
- Intentionally omitted subnet placement for ASG control.
- Added automated Apache user data and documented why the Base64 pre-encoded option remains disabled for plain Bash input.

## Next Verified Step

Create an Auto Scaling Group from `web-launch-template`, place it across `private-subnet-a` and `private-subnet-b`, attach it to `web-tg`, and verify that ASG-created instances bootstrap automatically and become healthy targets.
