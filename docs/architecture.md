# Implemented AWS Architecture Walkthrough

This document explains the infrastructure as it is actually implemented and verified. It focuses on **what was built, why each component exists, how traffic flows, the security reasoning, and the evidence used to validate the design**.

> The repository intentionally distinguishes the current implementation from the larger target architecture. Services are documented as implemented only after they are actually built and verified.

## 1. Current Architecture

```text
                         Internet
                            |
                            v
                    Internet Gateway
                            |
                 +----------+----------+
                 |    main-vpc         |
                 |    10.0.0.0/16      |
                 |                     |
       +---------+---------+  +--------+----------+
       | us-east-1a        |  | us-east-1b        |
       |                   |  |                   |
       | public-subnet-a   |  | public-subnet-b   |
       | 10.0.0.0/21       |  | 10.0.8.0/21       |
       |        \          |  |          /        |
       |         +---------+--+---------+         |
       |                   |                      |
       |               web-alb                    |
       |          Internet-facing ALB             |
       |               ALB-SG                     |
       |                   | HTTP/80              |
       |                   v                      |
       |                web-tg                    |
       |                   |                      |
       | private-subnet-a  |  private-subnet-b    |
       | 10.0.16.0/21      |  10.0.24.0/21        |
       |       |           |                      |
       |       v           |                      |
       |   web-server      |                      |
       |   App-SG          |                      |
       |   no public IP    |                      |
       +-------------------+----------------------+
                   |
                   v
              NAT Gateway
          outbound Internet only
```

The current application tier contains one EC2 target while the networking and ALB are prepared across two Availability Zones. A later phase can introduce Auto Scaling and additional targets without exposing the application instances directly to the Internet.

---

## 2. VPC — The Network Boundary

### What was created

- VPC: `main-vpc`
- CIDR: `10.0.0.0/16`
- Region: `us-east-1`

### Why

The VPC is the private network boundary for the workload. Using a custom VPC instead of relying on the default VPC makes subnetting, routing, Internet exposure, and security decisions explicit.

Think of the VPC as a company campus. Resources can live inside the campus, but being inside the same campus does not mean every building should have a public entrance.

---

## 3. Subnet Design — Public Edge vs Private Application Tier

Four subnets were created across two Availability Zones:

| Subnet | CIDR | AZ | Role |
|---|---|---|---|
| `public-subnet-a` | `10.0.0.0/21` | `us-east-1a` | Public ALB/NAT-facing resources |
| `public-subnet-b` | `10.0.8.0/21` | `us-east-1b` | Public ALB resources |
| `private-subnet-a` | `10.0.16.0/21` | `us-east-1a` | Private application resources |
| `private-subnet-b` | `10.0.24.0/21` | `us-east-1b` | Future private application capacity |

### Why two Availability Zones?

An Application Load Balancer requires multiple subnets/AZs for the intended highly available design. Splitting public and private tiers also prevents the application server from becoming the Internet entry point.

### Why is EC2 private?

The application server should receive web traffic through the ALB, not directly from arbitrary Internet clients. The current `web-server` therefore has **no public IPv4 address**.

---

## 4. Internet Gateway and Public Routing

An Internet Gateway is attached to `main-vpc`.

The public route table contains:

```text
0.0.0.0/0 -> Internet Gateway
```

Both public subnets are explicitly associated with this route table.

### Why

The Internet-facing ALB needs public subnet connectivity. The route to the Internet Gateway gives the public tier a path to/from the Internet, while the security groups still decide which traffic is permitted.

A route is a road; it does not automatically grant permission to enter a resource. Security groups are the guards at the destination.

---

## 5. NAT Gateway and Private Routing

A NAT Gateway was created for outbound Internet connectivity from the private tier.

The private route table contains:

```text
0.0.0.0/0 -> NAT Gateway
```

The private subnets are associated with the private route table.

### Why

The private EC2 instance needed outbound Internet access during bootstrap, for example to install packages, while remaining unreachable directly from the public Internet.

This creates an intentional asymmetry:

```text
Private EC2 -> NAT -> Internet     allowed for outbound connections
Internet -> NAT -> Private EC2     not a public inbound path
```

### Cost note

NAT Gateway is billable while provisioned and also has data-processing charges. It is tracked as a cleanup item for the lab.

---

## 6. Security Groups — Layered Access Control

Two security groups separate the public entry point from the application tier.

### `ALB-SG`

Inbound:

- HTTP/80 from `0.0.0.0/0`
- HTTPS/443 from `0.0.0.0/0`

Purpose: allow public web clients to reach the load balancer.

### `App-SG`

Inbound:

- HTTP/80 **from `ALB-SG` only**

Purpose: allow the application server to accept web traffic only when the source is a resource using the ALB security group.

### Security reasoning

```text
Internet
   |
   | HTTP/HTTPS
   v
ALB-SG
   |
   | HTTP/80
   v
App-SG
   |
   v
Private EC2
```

Instead of allowing `0.0.0.0/0` on the EC2 web port, the application security group references the ALB security group. This reduces the application tier's exposed attack surface and makes the intended traffic path enforceable by network policy.

---

## 7. EC2 Application Server

Verified configuration:

- Name: `web-server`
- Amazon Linux 2023
- Instance type: `t3.micro`
- VPC: `main-vpc`
- Subnet: `private-subnet-a`
- Public IPv4: **none**
- Security group: `App-SG`
- Key pair: `Web-Key`
- Web service: Apache (`httpd`) installed through user data

### Why user data?

User data bootstraps the web server automatically at instance launch. This is more repeatable than manually connecting to the instance and configuring Apache command by command.

### Verification

The replacement instance reached all EC2 status checks successfully. More importantly, the later ALB health check reached the application over HTTP/80 and reported the target as `Healthy`, providing application-path validation rather than relying only on EC2 infrastructure checks.

---

## 8. Engineering Incident — Public IP Misconfiguration

The first EC2 launch was placed in the intended private application subnet but accidentally received an auto-assigned public IPv4 address.

The instance was terminated and recreated correctly with public IP assignment disabled.

### Why this matters

A subnet name such as `private-subnet-a` does not itself make an instance private. Privacy is the result of several controls working together:

1. Route-table design.
2. Public IP assignment behavior.
3. Security-group rules.
4. The intended ingress architecture.

### Lesson learned

Infrastructure should be verified from the resulting resource state, not merely from the options we intended to select during creation.

---

## 9. Target Group — `web-tg`

Configuration:

- Target type: Instance
- Protocol: HTTP
- Port: 80
- Protocol version: HTTP/1.1
- VPC: `main-vpc`
- Health check protocol: HTTP
- Health check path: `/`
- Registered target: `web-server`

### Why a target group?

The target group is the backend pool known to the load balancer. It separates the public listener from individual server addresses and provides continuous health checking.

Conceptually:

```text
ALB Listener -> Target Group -> Healthy EC2 targets
```

The ALB does not need clients to know the EC2 private address. It forwards requests to healthy registered targets on their private network interfaces.

---

## 10. Application Load Balancer — `web-alb`

Verified configuration:

- Type: Application Load Balancer
- Scheme: Internet-facing
- IP type: IPv4
- VPC: `main-vpc`
- Subnets: `public-subnet-a`, `public-subnet-b`
- Security group: `ALB-SG`
- Listener: HTTP/80
- Default action: forward 100% to `web-tg`
- State: `Active`

### Why the ALB is public while EC2 is private

The ALB is the controlled front door. It accepts client requests in the public tier and forwards permitted requests to private application targets.

```text
Client
  |
  v
Internet-facing ALB
  |
  v
web-tg
  |
  v
Private EC2
```

This prevents the application server from requiring a public IP and creates the foundation for horizontal scaling later.

---

## 11. Health Checks — Proof of the Backend Path

The target group health check uses:

```text
HTTP GET /
Port 80
Expected success code: 200
```

The registered `web-server` is reported as:

```text
Healthy: 1
Unhealthy: 0
```

### What this proves

A healthy target is meaningful evidence that several components work together:

- The ALB/target-group relationship is configured.
- Network routing between the load-balancer nodes and target is functional.
- `App-SG` permits the required traffic from `ALB-SG`.
- The EC2 instance is running.
- Apache is listening on port 80.
- The `/` health-check endpoint returns an accepted HTTP response.

---

## 12. End-to-End Browser Validation

The ALB DNS endpoint was opened from an external browser over HTTP.

Observed result:

```text
AWS Traffic Spike Project
Web server is running successfully.
Hostname: ip-10-0-16-137.ec2.internal
```

### What this proves

This is the first full-path validation from the public Internet to the private application server:

```text
Browser
   |
   | HTTP/80
   v
web-alb
   |
   v
web-tg
   |
   v
App-SG
   |
   v
web-server
```

The successful browser response confirms that the public listener, ALB security group, target-group forwarding, application security-group reference, private EC2 networking, and Apache service all work together as intended.

### Small troubleshooting note

The first browser attempt used HTTPS and returned a connection-refused error. This was expected because the ALB currently has only an HTTP/80 listener and no HTTPS/443 listener/certificate configured yet. Repeating the test with `http://` succeeded.

This distinction is useful: the architecture was functioning correctly; the failed request used a protocol the listener was not configured to accept.

---

## 13. Verified Traffic Flow

```text
1. Client sends HTTP request to the ALB DNS name
                       |
2. Internet-facing web-alb receives request on TCP/80
                       |
3. ALB-SG permits HTTP ingress
                       |
4. Listener forwards request to web-tg
                       |
5. web-tg selects a healthy registered target
                       |
6. App-SG permits HTTP/80 from ALB-SG
                       |
7. Private web-server/Apache handles request
                       |
8. Browser receives the application response successfully
```

---

## 14. Evidence Captured During Implementation

Console screenshots have been captured locally for important milestones, including:

- VPC/subnet/routing configuration
- NAT and security-group configuration
- Initial EC2 public-IP mistake
- Corrected private EC2 launch
- Target group creation
- ALB provisioning
- ALB `Active` state
- Target group `Healthy` state
- Successful browser response through the ALB DNS name

Screenshots should contain no secrets, private keys, passwords, or access credentials before being committed.

---

## 15. Current State and Next Step

### Verified

- Custom VPC and four-subnet layout
- Public/private routing separation
- NAT-based private outbound connectivity
- Layered `ALB-SG` -> `App-SG` access
- Private EC2 without public IPv4
- Apache bootstrap
- `web-tg` registration
- Internet-facing `web-alb`
- HTTP listener forwarding to `web-tg`
- Target health: `Healthy`
- External browser request through ALB: successful

### Next phase

The networking and first end-to-end application path are now validated. The next engineering phase can move toward application-tier resilience and scaling, such as a launch template / Auto Scaling Group, followed later by database and observability layers according to the project plan.
