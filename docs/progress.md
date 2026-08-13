# Project Progress

## Current Status

**COMPLETED — implementation, runtime verification, monitoring, final application rollout, and cost cleanup finished.**

## Completed Checklist

- [x] Repository and project documentation initialized
- [x] Root MFA, IAM administrative workflow, and `$10` AWS budget configured
- [x] Region selected: `us-east-1` (N. Virginia)
- [x] Custom VPC `main-vpc` created with two public and two private subnets across `us-east-1a` and `us-east-1b`
- [x] Internet Gateway, route tables, and NAT Gateway configured
- [x] Layered security groups implemented: `ALB-SG -> App-SG -> DB-SG`
- [x] Private EC2 application tier verified without public IPv4 addresses
- [x] Application Load Balancer `web-alb` and target group `web-tg` created and verified
- [x] Auto Scaling Group `web-asg` deployed across both private subnets
- [x] Baseline capacity configured: Desired `2`, Min `2`, Max `4`
- [x] Target Tracking policy configured at `50%` average CPU
- [x] Full elasticity cycle verified: `2 -> 3 -> 4 -> 3 -> 2`
- [x] AWS Systems Manager access implemented using an EC2 IAM instance role
- [x] Private RDS MySQL instance `web-db` created with public access disabled
- [x] DB subnet group created across both private subnets
- [x] `DB-SG` restricted MySQL/TCP 3306 access to `App-SG`
- [x] EC2-to-RDS TCP connectivity verified
- [x] Authenticated MySQL session from private EC2 to RDS verified
- [x] Database `webapp` and table `visits` created
- [x] SQL INSERT and SELECT verified
- [x] PHP and MySQL extensions installed and verified
- [x] PHP application successfully read from and wrote to RDS
- [x] End-to-end path verified: Browser -> ALB -> Auto Scaling EC2/PHP -> private RDS
- [x] Configuration drift discovered during testing when only one backend had the new PHP application
- [x] Application bootstrap moved into the Launch Template so replacement/scaled instances are reproducible
- [x] Rolling Instance Refresh completed successfully after application bootstrap updates
- [x] Intermittent `404 Not Found` issue eliminated
- [x] Multiple Auto Scaling backends verified while sharing one persistent RDS visit counter
- [x] Final dashboard UI verified after the final fleet refresh
- [x] CloudWatch unhealthy-target alarm configured: `UnHealthyHostCount >= 1`
- [x] SNS email notification topic configured and subscription confirmed
- [x] Production hardening recommendations documented: HTTPS/ACM, AWS WAF, secure DB credential handling, and RDS Multi-AZ
- [x] Final resource cleanup completed
- [x] Final billing page reviewed: estimated grand total `USD 0.00` under the AWS Free Plan at the time of cleanup

## Verified Runtime Architecture

```text
Internet
   |
   v
Application Load Balancer
(public subnets / ALB-SG)
   |
   v
Target Group
   |
   v
Auto Scaling Group
(2 baseline, scales to 4)
   |
   +--> Private EC2 / us-east-1a
   +--> Private EC2 / us-east-1b
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
Systems Manager -> private EC2 administration
CloudWatch -> scaling metrics and unhealthy-target alarm
SNS -> email notification
```

## Key Verification Results

### Auto Scaling

Observed complete elasticity cycle:

```text
2 -> 3 -> 4 -> 3 -> 2
```

This verified both automatic scale-out and scale-in while respecting configured Min/Max capacity.

### Database

Verified from a private EC2 instance:

```text
EC2 -> App-SG -> DB-SG:3306 -> RDS MySQL
```

TCP reachability, MySQL authentication, database creation, table creation, INSERT, and SELECT all succeeded.

### End-to-End Application

The final PHP dashboard displayed:

- RDS connection status
- Total visit count stored in RDS
- Current backend EC2 hostname
- Last request time

Repeated ALB requests reached different EC2 backends while the same visit counter continued increasing. This demonstrated load balancing plus shared persistent state.

## Engineering Issue Found During Testing

A useful failure was discovered after the PHP page was manually added to only one backend. Requests alternated between success and `404 Not Found` even though both targets were healthy.

Root cause:

```text
ALB
├── Backend A -> new application present
└── Backend B -> new application missing
```

This was application configuration drift, not an ALB failure. The final fix was to move the application bootstrap into the Launch Template and roll the fleet, making replacement and scaled instances reproducible.

## Monitoring

Operational alarm implemented:

- Namespace: `AWS/ApplicationELB`
- Metric: `UnHealthyHostCount`
- Statistic: `Maximum`
- Period: `1 minute`
- Threshold: `>= 1`
- Datapoints: `1 out of 1`
- Missing data: treated as not breaching
- Action: SNS email notification

```text
Unhealthy backend -> CloudWatch Alarm -> SNS -> Email
```

Target Tracking CloudWatch alarms were also used by Auto Scaling during the elasticity test.

## Production Hardening Recommendations

These were evaluated and documented but are **not claimed as implemented**:

- **HTTPS / ACM:** terminate TLS on the ALB using an ACM certificate and redirect HTTP to HTTPS. Not implemented because no owned domain was available for DNS validation.
- **AWS WAF:** place WAF in front of the ALB for managed application-layer protections and rate-based rules where justified.
- **Database credentials:** avoid long-lived passwords in EC2 User Data or application files. Prefer Secrets Manager and/or IAM Database Authentication, a dedicated least-privilege DB user, TLS, and credential rotation.
- **RDS Multi-AZ:** recommended for production database availability. The lab remained Single-AZ because the account Free Plan restricted deployment options during creation.

## Cleanup / Cost Control

After final evidence collection, temporary project resources were removed rather than left running.

Verified cleanup included:

- [x] Auto Scaling capacity reduced to zero and `web-asg` deleted
- [x] Remaining project EC2 instance terminated
- [x] `web-alb` deleted
- [x] NAT Gateway deleted
- [x] RDS `web-db` deleted without a final snapshot or retained automated backups
- [x] `web-tg` deleted
- [x] Launch Template deleted
- [x] Project security groups removed
- [x] Project IAM role `WebServer-SSM-Role` removed
- [x] CloudWatch alarms removed
- [x] SNS project topic removed
- [x] RDS manual snapshots verified: `0`
- [x] Elastic IP addresses verified: `0`
- [x] EBS volumes verified: `0`
- [x] Billing page reviewed after cleanup: estimated grand total `USD 0.00`

Resource Explorer briefly displayed stale deleted resources while its index was still updating; live service consoles were used as the source of truth for cleanup verification.

## Project Outcome

The project demonstrated a complete AWS web workload lifecycle:

```text
Design
  -> Build
  -> Secure
  -> Scale
  -> Observe
  -> Integrate Data
  -> Detect/Fix Drift
  -> Verify
  -> Clean Up
```

The lab is now complete. Future work belongs to separate production-hardening or follow-on projects rather than expanding this lab indefinitely.
