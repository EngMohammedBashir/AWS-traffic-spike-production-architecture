# MADAR Cloud Transformation — Phase 1

## Resilient AWS Web Architecture for Traffic Spikes

## Project Status

**Completed and verified.**

## Company Context

**MADAR (مدار)** is a fictional growing digital commerce company used as a continuous business case across this cloud engineering journey.

At the beginning of MADAR's cloud transformation, its web application depended on a legacy single-server architecture. As customer traffic grew, this design became a reliability and scaling risk. Phase 1 focuses on rebuilding that web tier into a resilient AWS architecture that can absorb traffic spikes while keeping the application and database tiers isolated from direct Internet access.

This phase establishes the infrastructure foundation that later MADAR projects build on as the company grows and introduces new workloads.

## Business Problem

MADAR's legacy single-server application suffers from:

- Single point of failure
- No horizontal scaling
- Weak network isolation
- Limited health visibility
- Database exposure risk
- Manual server configuration that does not scale

The immediate engineering goal is therefore not simply to move a server to AWS. It is to redesign the application path so MADAR can scale horizontally, recover unhealthy capacity, protect private tiers, and observe production behavior.

## Implemented Architecture

```text
MADAR Customers
   |
   v
Internet
   |
   v
Application Load Balancer
(public subnets)
   |
   v
Target Group
   |
   v
EC2 Auto Scaling Group
(private subnets across two AZs)
   |
   +--> Apache + PHP application
   |
   v
Private RDS MySQL
   |
   v
webapp.visits
```

Supporting services:

```text
NAT Gateway       -> outbound access for private EC2
Systems Manager   -> private EC2 administration
CloudWatch        -> scaling and health monitoring
SNS               -> email notifications
IAM               -> instance permissions
```

## What Was Actually Verified

- Multi-AZ VPC design with public and private subnets
- Internet-facing ALB with private EC2 backends
- Auto Scaling baseline of `2`, maximum of `4`
- Full elasticity cycle: `2 -> 3 -> 4 -> 3 -> 2`
- Systems Manager access to private EC2 instances
- Private RDS MySQL with `DB-SG` allowing TCP/3306 only from `App-SG`
- EC2-to-RDS TCP connectivity and MySQL authentication
- SQL database/table creation plus INSERT and SELECT
- PHP application reading from and writing to RDS
- Repeated ALB requests reaching different EC2 backends while sharing one persistent RDS visit counter
- Configuration drift diagnosed and fixed by moving application bootstrap into the Launch Template
- CloudWatch unhealthy-target alarm with SNS email notification
- Final dashboard verified after rolling instance replacement
- Project resources cleaned up after evidence collection
- Final AWS Bills page showed estimated grand total `USD 0.00` under the Free Plan at cleanup time

## Final Application Path

```text
MADAR Customer
   -> ALB
   -> Auto Scaling EC2
   -> Apache/PHP
   -> Private RDS MySQL
```

The dashboard displayed the RDS connection state, total visits, current backend hostname, and last request time. Refreshing the page demonstrated both load balancing and shared database persistence.

## Incident During Validation: Configuration Drift

During testing, one backend had the new PHP file while another did not. The ALB therefore alternated between a successful response and `404 Not Found` even though both targets were healthy.

This exposed an operational problem MADAR would face as the fleet scaled: manually configured servers could diverge from one another.

The fix was not to keep editing servers manually. The application bootstrap was moved into the Launch Template and the fleet was refreshed so every replacement or scaled instance launched consistently.

```text
Traffic growth
   -> multiple EC2 instances
   -> configuration inconsistency discovered
   -> application bootstrap moved to Launch Template
   -> fleet refreshed
   -> consistent replacement/scaling behavior
```

## Production Hardening Recommendations

The following are intentionally documented as **recommended, not implemented**:

- **HTTPS + ACM** — terminate TLS on the ALB and redirect HTTP to HTTPS. Not implemented because no owned domain was available for certificate validation.
- **AWS WAF** — add managed and rate-based application-layer protection where justified.
- **Secure database credentials** — replace static passwords in User Data/application files with Secrets Manager and/or IAM Database Authentication, least-privilege IAM, a dedicated application DB user, TLS, and rotation.
- **RDS Multi-AZ** — recommended for production availability; the lab used Single-AZ because the Free Plan restricted deployment options.

## Cost Control and Cleanup

Temporary resources were deleted after final testing, including Auto Scaling capacity, ALB, NAT Gateway, RDS, target group, Launch Template, project security groups, project IAM role, alarms, and SNS topic.

Final checks also confirmed:

- Elastic IP addresses: `0`
- EBS volumes: `0`
- RDS manual snapshots: `0`
- CloudWatch alarms: `0`
- SNS project topics: `0`

## MADAR Cloud Journey

Phase 1 solves MADAR's first infrastructure problem: **a fragile web application that cannot safely handle growth**.

The result is a scalable and observable web foundation:

```text
Legacy single server
   -> resilient AWS network
   -> load-balanced private compute
   -> Auto Scaling
   -> private persistent database
   -> monitoring and operational alerts
```

As MADAR continues to grow, not every workload should pass synchronously through this web stack. Later phases extend the company architecture with asynchronous and serverless processing for background jobs and bursty event workloads.

## Documentation

- `docs/architecture.md` — implemented architecture and technical reasoning
- `docs/progress.md` — complete implementation/verification checklist and cleanup record
- `docs/runtime-verification.md` — runtime test evidence
- `docs/rds-and-elasticity-verification.md` — RDS and Auto Scaling verification details
- `evidence/` — complete project screenshot evidence

## Key Takeaway

This phase demonstrated the complete engineering loop around MADAR's first cloud reliability problem:

```text
Business Problem -> Design -> Build -> Secure -> Scale -> Observe -> Debug -> Verify -> Clean Up
```

## Final Result

![Final MADAR AWS Traffic Spike Dashboard 1](evidence/final-aws-traffic-spike-dashboard-1.png)

![Final MADAR AWS Traffic Spike Dashboard 2](evidence/final-aws-traffic-spike-dashboard-2.png)
