# Implemented AWS Architecture Walkthrough

This document explains the infrastructure as it is actually implemented and verified. It focuses on **what was built, why each component exists, how traffic flows, the security reasoning, and the evidence used to validate the design**.

> The repository intentionally distinguishes **Configured** from **Verified**. A resource can be successfully created and configured without its runtime behavior having been proven yet.

## 1. Current Architecture

```text
Internet
   |
   v
Internet Gateway
   |
   v
web-alb (public subnets, ALB-SG)
   |
   v
web-tg (HTTP/80)
   |
   v
web-asg
   |
   +--> EC2 instances in private-subnet-a / private-subnet-b (App-SG)
             |
             +--> NAT Gateway for outbound Internet access
```

The public entry point and private application tier are intentionally separated. The ALB accepts Internet traffic while Auto Scaling-managed EC2 application instances remain in private subnets.

## 2. VPC and Subnet Design

- VPC: `main-vpc`
- CIDR: `10.0.0.0/16`
- Region: `us-east-1`

| Subnet | CIDR | AZ | Role |
|---|---|---|---|
| `public-subnet-a` | `10.0.0.0/21` | `us-east-1a` | Public ALB/NAT-facing resources |
| `public-subnet-b` | `10.0.8.0/21` | `us-east-1b` | Public ALB resources |
| `private-subnet-a` | `10.0.16.0/21` | `us-east-1a` | Private application resources |
| `private-subnet-b` | `10.0.24.0/21` | `us-east-1b` | Private application capacity |

The VPC acts like a company campus. Public subnets contain controlled entrances; private subnets contain internal application servers that do not need their own public doors.

## 3. Internet and NAT Routing

The public route table sends `0.0.0.0/0` to the Internet Gateway. Private subnets use a NAT Gateway for outbound Internet connectivity.

```text
Private EC2 -> NAT -> Internet     outbound connectivity
Internet -> NAT -> Private EC2     not a public inbound path
```

This allows package installation and software updates without assigning public IPv4 addresses to application instances.

## 4. Security Groups

### `ALB-SG`

Public-facing security boundary for the Application Load Balancer.

### `App-SG`

Allows HTTP/80 from `ALB-SG` rather than from the entire Internet.

```text
Internet -> ALB-SG -> App-SG -> Private EC2
```

This enforces the intended application path and reduces direct exposure of the compute tier.

## 5. EC2 Application Server

Verified configuration includes Amazon Linux 2023, `t3.micro`, `App-SG`, placement in `private-subnet-a`, and no public IPv4 address. Apache is installed and configured through EC2 user data.

The first EC2 launch accidentally received a public IPv4 address. It was terminated and recreated correctly. This demonstrated that a subnet name alone does not make an instance private; routing, public-IP assignment, security groups, and ingress design all matter.

## 6. Target Group — `web-tg`

- Target type: Instance
- Protocol: HTTP
- Port: 80
- Health check: `HTTP /`
- Registered application target reached `Healthy`

The target group is the backend pool between the ALB and application servers. It also gives the ALB a health signal so requests can be sent only to usable targets.

## 7. Application Load Balancer — `web-alb`

- Internet-facing
- IPv4
- `public-subnet-a` + `public-subnet-b`
- `ALB-SG`
- Listener: HTTP/80
- Default action: forward 100% to `web-tg`
- State verified as `Active`

The ALB is the public front door while EC2 remains private.

## 8. End-to-End Validation

An external browser request through the ALB DNS name over HTTP returned:

```text
AWS Traffic Spike Project
Web server is running successfully.
Hostname: ip-10-0-16-137.ec2.internal
```

This validated the original complete path:

```text
Browser -> web-alb -> web-tg -> App-SG -> private web-server -> Apache
```

An earlier HTTPS attempt was refused because HTTPS/443 is not configured on the ALB yet. Repeating the request over the configured HTTP/80 listener succeeded.

---

## 9. Launch Template — Repeatable Server Definition

The application tier now has a reusable Launch Template for Auto Scaling.

Configured Launch Template:

- Name: `web-launch-template`
- AMI: Amazon Linux 2023 x86_64
- Instance type: `t3.micro`
- Key pair: `Web-Key`
- Subnet: intentionally omitted from the template
- Security group: `App-SG`
- User data: application bootstrap script

### Why is the subnet omitted?

Subnet placement belongs to the Auto Scaling Group rather than the Launch Template. This lets one server definition be launched into multiple private subnets/AZs instead of hard-coding every future instance into one subnet.

Think of the Launch Template as the **manufacturing blueprint** for a server. The Auto Scaling Group is the factory manager that decides how many servers to manufacture and in which approved private locations to place them.

## 10. User Data — Automated Application Bootstrap

The Launch Template includes this user-data script:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd

systemctl enable httpd
systemctl start httpd

HOSTNAME=$(hostname)

cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>AWS Traffic Spike Project</title>
</head>
<body>
    <h1>AWS Traffic Spike Project</h1>
    <p>Web server is running successfully.</p>
    <p>Hostname: $HOSTNAME</p>
</body>
</html>
EOF
```

### What each part does

- `#!/bin/bash` — executes the bootstrap script with Bash.
- `dnf update -y` — updates packages without interactive confirmation.
- `dnf install -y httpd` — installs Apache HTTP Server.
- `systemctl enable httpd` — makes Apache start automatically on future boots.
- `systemctl start httpd` — starts Apache immediately.
- `HOSTNAME=$(hostname)` — captures the unique EC2 hostname, useful for identifying which backend served a request.
- `cat > /var/www/html/index.html <<EOF ... EOF` — creates the simple application page in Apache's document root.

### Why User Data matters for Auto Scaling

Without bootstrap automation, Auto Scaling could launch a technically running EC2 instance that has no Apache service or application page. The target-group health check would fail and that capacity would not serve users.

```text
ASG needs capacity
      |
      v
Launch Template -> EC2
      |
      v
User Data
      |
      +--> Install/start Apache
      +--> Create application page
      |
      v
web-tg health check
      |
      v
Healthy target -> ALB traffic
```

Manual configuration does not scale. User data makes each instance repeatable, automatic, and disposable.

The AWS console option indicating user data is already Base64 encoded was intentionally left disabled because the supplied content is plain Bash; AWS performs the required encoding.

---

## 11. Auto Scaling Group — `web-asg`

**Status: Configured. Runtime scaling behavior is not yet documented as Verified.**

Configured values:

- Auto Scaling Group: `web-asg`
- Launch Template: `web-launch-template`
- AMI: Amazon Linux 2023
- Instance type: `t3.micro`
- Security group inherited from Launch Template: `App-SG`
- Availability Zones: `us-east-1a` and `us-east-1b`
- Subnets:
  - `private-subnet-a` — `us-east-1a`
  - `private-subnet-b` — `us-east-1b`
- Availability Zone distribution: Balanced best effort
- Desired capacity: `2`
- Minimum capacity: `2`
- Maximum capacity: `4`
- Load balancer: `web-alb`
- Target group: `web-tg`
- Health checks: `EC2, ELB`
- Health check grace period: `300 seconds`
- Tag propagated to new instances: `Project = AWS-Traffic-Spike`

The group reached **At desired capacity** with two instances reported healthy at the infrastructure level. This proves the ASG can maintain its configured baseline capacity. It does **not** by itself prove the complete scale-out/scale-in experiment, which remains pending.

### Why private subnets?

The application servers do not need direct Internet exposure. The ALB acts as the controlled public front door, while application instances live behind it.

```text
Internet -> ALB -> Target Group -> ASG -> Private EC2
```

This is similar to a company reception desk: visitors enter through reception rather than walking directly into internal offices.

### Why Desired 2, Min 2, Max 4?

`Min = 2` keeps two application instances as the baseline, improving availability across two Availability Zones. `Desired = 2` starts the system at that baseline. `Max = 4` gives the project enough headroom to demonstrate scale-out without allowing an uncontrolled number of instances to launch during testing.

```text
Normal baseline: 2 instances
Traffic rises:   2 -> 3 -> 4
Upper guardrail: 4 instances
Traffic falls:   4 -> 3 -> 2
Lower guardrail: 2 instances
```

For this lab, the `2 -> 4` range is deliberately small: large enough to demonstrate elasticity, but bounded for cost control.

### Health checks

`EC2` health checks detect infrastructure-level instance problems. `ELB` health checks allow Auto Scaling to consider whether instances attached to the load-balancing path are healthy.

The grace period is `300 seconds`, giving a newly launched instance time to boot, execute user data, install/start Apache, and become ready before health-check failures are acted upon.

### Why EBS health checks are not enabled here?

EBS health checks are not required to demonstrate the core application Auto Scaling behavior in this project. The key signals for this architecture are whether the EC2 instance is operational and whether the load-balancing application path considers it healthy. Additional storage-health integration can be introduced when the workload has storage-specific recovery requirements.

---

## 12. Dynamic Scaling Policy — `cpu-target-tracking`

**Status: Configured. Scale-out and scale-in behavior still requires load-test verification.**

The initial policy creation attempt occurred immediately after creation of the Auto Scaling service-linked role and failed while IAM role propagation was still completing. After the role became available, the policy was created successfully.

Configured policy:

- Policy type: Target tracking scaling
- Policy name: `cpu-target-tracking`
- Metric: Average CPU utilization
- Target: `50%`
- Instance warmup: `300 seconds`
- Scale-in: Enabled

### Target Tracking as a thermostat

Target Tracking behaves like a **thermostat**.

A thermostat does not simply say "turn cooling on once." It continuously compares the current temperature with the desired temperature and adjusts the system to stay near the target.

The scaling policy follows the same idea:

```text
Average CPU > target
        |
        v
Need more capacity
        |
        v
Scale Out 📈

Average CPU returns toward 50%
        |
        v
Maintain appropriate capacity

Demand falls
        |
        v
Scale In 📉
```

### Why CPU at 50%?

For this demonstration, `50%` gives the scaling system meaningful headroom before instances become heavily saturated. It also makes it practical to generate enough CPU load during the experiment to trigger a scale-out event.

This is a lab design choice, not a universal production value. A production target should be selected from application behavior, latency objectives, workload characteristics, and measured capacity.

### Instance warmup = 300 seconds

New instances need time to become useful. During bootstrap they are installing packages, starting Apache, and entering the target group. The `300`-second warmup prevents fresh-instance CPU metrics from immediately distorting the group's scaling decisions while initialization is still underway.

### Scale-in enabled

Scale-in is enabled so the experiment can demonstrate both halves of elasticity:

```text
CPU ↑ -> Scale Out
CPU ↓ -> Scale In
```

The expected floor remains `2` because the ASG minimum capacity is two instances.

---

## 13. Notifications and Tags

### SNS notifications

SNS notifications were intentionally skipped at this stage. Notifications can be useful operationally, but they are not required to prove that the ASG can launch capacity, attach instances to the target group, and react to load.

### Project tag

```text
Project = AWS-Traffic-Spike
```

The tag is configured to propagate to newly launched instances. This improves resource identification and supports later filtering, inventory, and cost analysis.

---

## 14. Verification Plan — Pending Runtime Test

Creating the ASG and scaling policy proves **configuration**, not complete runtime behavior.

The remaining verification chain is:

```text
ASG
 |
 v
EC2 instances launched
 |
 v
User Data executes
 |
 v
Apache starts
 |
 v
web-tg reports targets Healthy
 |
 v
web-alb serves the application
 |
 v
Generate load
 |
 v
CPU rises above target
 |
 v
Scale Out
 |
 v
Remove load
 |
 v
CPU falls
 |
 v
Scale In
```

Or, compactly:

```text
ASG -> EC2 -> User Data -> Apache -> web-tg Healthy -> ALB -> Load Test -> Scale Out -> Scale In
```

Evidence to capture during verification:

1. ASG instance count at the baseline of `2`.
2. Both ASG-created targets becoming `Healthy` in `web-tg`.
3. Successful HTTP requests through `web-alb`.
4. Hostname responses showing traffic reaching Auto Scaling-managed backends.
5. CPU metric rising during the load test.
6. ASG Activity showing scale-out and creation of additional instance capacity.
7. Desired/actual capacity increasing above `2` without exceeding `4`.
8. CPU falling after load is removed.
9. ASG Activity showing scale-in.
10. Capacity returning to the minimum/baseline of `2`.

**Do not mark Auto Scaling as Verified until this chain has actually been observed.**

---

## 15. Current Project State

### Verified

- Custom VPC and public/private subnet layout
- Internet Gateway and NAT-based private outbound connectivity
- `ALB-SG -> App-SG` security boundary
- Private EC2 without public IPv4
- Apache application bootstrap on the original server
- `web-tg` target health on the original application path: `Healthy`
- `web-alb`: `Active`
- External browser request through the ALB: successful

### Configured

- `web-launch-template`
- `web-asg`
- ASG baseline: Desired `2`, Min `2`, Max `4`
- ASG placement across `private-subnet-a` and `private-subnet-b`
- `web-asg -> web-tg -> web-alb` integration
- EC2 + ELB health checks with 300-second grace period
- `cpu-target-tracking`
- Average CPU target: `50%`
- Instance warmup: `300 seconds`
- Scale-in enabled
- `Project = AWS-Traffic-Spike` tag propagation

### Pending verification

- ASG-created application targets confirmed `Healthy` in `web-tg`
- Application response through ALB specifically from ASG-created instances
- CPU load test
- Automatic scale-out from baseline capacity
- New scaled-out instances becoming healthy and serving traffic
- Automatic scale-in after load removal
- Return to minimum capacity of `2`

The next milestone is the controlled runtime verification of the Auto Scaling behavior. Once that test succeeds, this document should be updated with the actual observed timestamps, capacity changes, target health, and screenshots/evidence rather than merely expected behavior.
