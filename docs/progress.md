# Project Progress

## Current Status

**Phase 7 — Final application rollout and project hardening (in progress)**

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
- [x] Private EC2 application tier verified without public IPv4
- [x] `web-tg` created on HTTP/80
- [x] `web-alb` created across both public subnets
- [x] HTTP:80 listener forwards to `web-tg`
- [x] ALB browser test succeeded
- [x] Launch Template v1 created with Apache bootstrap
- [x] Auto Scaling Group `web-asg` created across both private subnets
- [x] ASG attached to `web-tg`
- [x] Baseline capacity configured: Desired `2`, Min `2`, Max `4`
- [x] Target Tracking policy `cpu-target-tracking` configured at `50%` average CPU
- [x] Scale-out verified: `2 -> 3 -> 4`
- [x] Scale-in verified: `4 -> 3 -> 2`
- [x] Full elasticity cycle verified: `2 -> 3 -> 4 -> 3 -> 2`
- [x] SSM IAM instance role added through Launch Template v2
- [x] Rolling Instance Refresh to v2 completed successfully
- [x] Private ASG instances verified online in Systems Manager
- [x] RDS DB subnet group `web-db-subnet-group` created across both private subnets
- [x] Database security group `DB-SG` created
- [x] `DB-SG` permits MySQL/TCP 3306 only from `App-SG`
- [x] Private RDS MySQL instance `web-db` created
- [x] RDS public access disabled
- [x] EC2-to-RDS TCP/3306 connectivity verified
- [x] Authenticated MySQL session from private EC2 to RDS verified
- [x] Database `webapp` created
- [x] Table `visits` created
- [x] SQL INSERT and SELECT verified
- [x] PHP and MySQL extensions installed and verified
- [x] PHP application successfully reads from and writes to RDS
- [x] End-to-end path verified: Browser -> ALB -> EC2/PHP -> private RDS
- [x] Configuration drift discovered when only one backend had `db-test.php`
- [x] Launch Template v3 created to bootstrap PHP + RDS application consistently
- [x] Instance Refresh to v3 completed
- [x] Repeated ALB requests no longer returned intermittent 404 responses
- [x] Multiple ASG backends verified using one shared RDS visit counter
- [x] CloudWatch alarm created for `UnHealthyHostCount >= 1`
- [x] SNS topic `webapp-monitoring-alerts` created and email subscription confirmed
- [x] Final dashboard UI created and tested on both current backends
- [x] Launch Template v4 created and set as default with final dashboard bootstrap
- [ ] Complete and verify Instance Refresh to Launch Template v4
- [ ] Verify final dashboard through ALB after v4 replacement completes
- [ ] Final GitHub documentation pass
- [ ] Final resource cleanup after evidence collection

## Current Architecture

```text
Internet
   |
   v
web-alb (public subnets, ALB-SG)
   |
   v
web-tg
   |
   v
web-asg (2 baseline, scales to 4)
   |
   +--> private-subnet-a / us-east-1a
   +--> private-subnet-b / us-east-1b
           |
           v
        App-SG
           |
           | TCP/3306
           v
         DB-SG
           |
           v
     Private RDS MySQL
           |
           v
      webapp.visits
```

Operational support:

```text
CloudWatch -> Target Tracking / ALB health alarm
SNS -> Email notification
Systems Manager -> private EC2 administration
```

## Verified Runtime Results

### Elasticity

```text
2 -> 3 -> 4 -> 3 -> 2
```

The complete Auto Scaling scale-out and scale-in cycle was observed using controlled CPU load and the configured Target Tracking policy.

### Database path

```text
Private EC2 -> App-SG -> DB-SG:3306 -> RDS MySQL
```

TCP reachability, MySQL authentication, database creation, table creation, INSERT, and SELECT were all verified.

### Application integration

The PHP application now records visits in the shared RDS database and displays the total record count plus the backend hostname. Repeated requests through the ALB reached different EC2 backends while preserving the same shared visit counter.

## Launch Template Evolution

- **v1** — Apache web-server bootstrap.
- **v2** — added `WebServer-SSM-Role` for Systems Manager access.
- **v3** — added PHP, `php-mysqlnd`, and the RDS-backed web application bootstrap.
- **v4** — added the final dashboard UI and is currently the default Launch Template version.

Subnet placement remains intentionally omitted from the Launch Template. The Auto Scaling Group controls placement across the two private subnets/AZs.

## Monitoring

Existing Target Tracking alarms manage CPU-based elasticity.

A separate operational alarm was added:

- Metric: `AWS/ApplicationELB` → `UnHealthyHostCount`
- Statistic: `Maximum`
- Period: `1 minute`
- Threshold: `>= 1`
- Datapoints: `1 out of 1`
- Missing data: treated as not breaching
- Notification: `webapp-monitoring-alerts` SNS topic → confirmed email subscription

```text
Unhealthy target -> CloudWatch alarm -> SNS -> Email
```

## Production Hardening Notes

The lab intentionally distinguishes implemented features from production recommendations.

- **HTTPS / ACM:** not implemented because no owned domain is available for certificate validation. Production deployments should terminate HTTPS on the ALB and redirect HTTP to HTTPS.
- **AWS WAF:** recommended in front of the ALB for managed web protections and rate-based controls. Not enabled in this lab to avoid adding unnecessary cost.
- **Database credentials:** password-based authentication is used for the lab. Production should avoid static credentials in application files or EC2 User Data and should use AWS Secrets Manager and/or IAM Database Authentication with least-privilege IAM roles and a dedicated application DB user.
- **RDS Multi-AZ:** production target is Multi-AZ. The implemented lab database is Single-AZ because the account Free Plan restricted the available deployment options during creation.

## Cost Warning

NAT Gateway, Application Load Balancer, RDS, EC2/EBS, and optional services can generate ongoing charges. Cleanup remains mandatory after final testing and evidence collection.

## Rules

- Verify runtime behavior before marking it complete.
- GitHub is the source of truth for project state.
- Record actual observations; never invent availability, performance, recovery, or cost results.
- Never commit passwords, access keys, private keys, or other secret values.
- Clearly separate **Implemented & Verified** from **Production Recommendation**.

## Session Log

### Session 0 — Repository Initialization
Repository initialized.

### Session 1 — AWS Account Guardrails
Root MFA, budget, alerts, and IAM administrative workflow configured.

### Session 2 — VPC & Subnets
Custom VPC and four subnets created across two Availability Zones.

### Session 3 — Routing, NAT & Security
IGW, NAT, route tables, `ALB-SG`, and `App-SG` configured.

### Session 4 — Private EC2 Bootstrap
Private application server and Apache bootstrap verified.

### Session 5 — Target Group & ALB
`web-tg` and `web-alb` configured and end-to-end HTTP verified.

### Session 6 — Launch Template & Auto Scaling
Launch Template created, ASG deployed across both private subnets, Target Tracking configured, and full `2 -> 3 -> 4 -> 3 -> 2` elasticity cycle verified.

### Session 7 — Private Operations
Launch Template v2 added the SSM instance profile; rolling refresh and Session Manager access were verified.

### Session 8 — RDS Data Tier
Private RDS networking, `DB-SG`, MySQL connectivity, authentication, `webapp`, `visits`, INSERT, and SELECT were verified.

### Session 9 — Web Application Integration
PHP application integrated with RDS. Configuration drift between backends was discovered and corrected with Launch Template v3 and an Instance Refresh.

### Session 10 — Monitoring & Final UI
CloudWatch unhealthy-target monitoring plus SNS email notification were configured. A final dashboard UI was created, and Launch Template v4 was prepared for reproducible rollout.

## Next Verified Step

Complete the Instance Refresh to Launch Template v4, then verify that repeated ALB requests reach multiple newly replaced backends while the dashboard remains available and the shared RDS visit counter continues increasing.