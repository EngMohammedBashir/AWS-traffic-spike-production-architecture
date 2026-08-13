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
- [ ] Create the application EC2 security group separately from `alb-sg`
- [ ] Finalize EC2 launch configuration
- [ ] Launch and verify first EC2 web server

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
│
└── us-east-1b
    ├── public-subnet-b   10.0.8.0/21
    └── private-subnet-b  10.0.24.0/21

Public route table:
  0.0.0.0/0 -> Internet Gateway

Private route table:
  0.0.0.0/0 -> NAT Gateway
```

## Security Design Note

`alb-sg` is intended for the Application Load Balancer, not as the final EC2 application security group. Before launching the production-style application instance, create a separate application security group so inbound web traffic can be restricted to the ALB security group rather than exposing the EC2 application tier directly to the Internet.

## EC2 Preparation

An EC2 launch configuration was started with:

- Name: `web-server`
- AMI: Amazon Linux 2023
- Instance type under consideration/selected in console: `t3.micro`
- Key pair created: `web-key`
- Custom VPC selected
- Initial subnet selection was `public-subnet-a`
- User data prepared to install and start Apache (`httpd`) and create a simple test page containing the instance hostname.

**Important:** EC2 has not yet been recorded as successfully launched. Network/security settings must be corrected/reviewed first so the final architecture keeps the application tier private behind the ALB.

## Cost Warning

The NAT Gateway is now active and is a billable resource. It incurs hourly charges while provisioned plus data-processing charges. It must be deleted during final cleanup (or earlier whenever the lab no longer needs private outbound Internet access). Cost will be compared against actual AWS billing data rather than invented estimates.

## Documentation Evidence

AWS Console screenshots have been captured locally for several completed networking steps, including VPC/subnets, Internet Gateway, route tables, subnet associations, NAT routing, and security group work. Screenshot files are not yet committed to GitHub.

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
- README and progress tracker created.

### Session 1 — AWS Account Guardrails
- Enabled root MFA.
- Created `$10` monthly AWS budget and alerts.
- Created safe daily IAM administrative access.

### Session 2 — VPC & Subnets
- Created `main-vpc` (`10.0.0.0/16`).
- Created two public and two private subnets across two Availability Zones.

### Session 3 — Routing, NAT & Compute Preparation
- Created and attached the Internet Gateway.
- Created public/private route tables and explicit subnet associations.
- Routed public Internet traffic through the Internet Gateway.
- Created a NAT Gateway and routed private outbound traffic through it.
- Created `alb-sg` with public HTTP/HTTPS ingress for the future ALB.
- Began EC2 launch preparation and created `web-key`.
- Prepared Amazon Linux 2023 Apache user data.
- Identified an architecture correction before launch: EC2 should use its own application security group and ultimately live in a private application subnet behind the ALB.

## Next Verified Step

Before launching EC2, create the dedicated application security group (`app-sg`) and configure HTTP ingress from `alb-sg`. Then launch the test/application instance in the appropriate private subnet and verify its bootstrap path.
