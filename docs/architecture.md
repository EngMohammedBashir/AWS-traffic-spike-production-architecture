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
   v
web-alb (public subnets, ALB-SG)
   |
   v
web-tg (HTTP/80)
   |
   v
Private EC2 application tier (App-SG)
   |
   +--> NAT Gateway for outbound Internet access
```

The public entry point and private application tier are intentionally separated. The next phase introduces a reusable Launch Template and Auto Scaling Group so instances can be created consistently and later scaled across private subnets.

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

This validated the complete path:

```text
Browser -> web-alb -> web-tg -> App-SG -> private web-server -> Apache
```

An earlier HTTPS attempt was refused because HTTPS/443 is not configured on the ALB yet. Repeating the request over the configured HTTP/80 listener succeeded.

---

## 9. Launch Template — Repeatable Server Definition

The application tier is now being converted from a manually launched EC2 instance into a repeatable server definition for Auto Scaling.

Current Launch Template configuration:

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

#### `#!/bin/bash`

Tells the instance to execute the bootstrap script with Bash.

#### `dnf update -y`

Updates available package metadata/packages without requiring interactive confirmation. The `-y` matters because Auto Scaling launches must complete without a human answering prompts.

#### `dnf install -y httpd`

Installs Apache HTTP Server. A newly launched Amazon Linux instance does not automatically contain our running web application, so the bootstrap process prepares the web-server software.

#### `systemctl enable httpd`

Configures Apache to start automatically on future boots.

#### `systemctl start httpd`

Starts Apache immediately during the initial bootstrap so the instance can begin serving HTTP and eventually pass the target-group health check.

#### `HOSTNAME=$(hostname)`

Reads the unique hostname of the current EC2 instance and stores it in a shell variable.

This is especially useful for the Auto Scaling phase. When multiple EC2 instances exist behind the ALB, the page can reveal which backend generated a particular response.

#### `cat > /var/www/html/index.html <<EOF ... EOF`

Creates the web application's simple HTML landing page in Apache's document root. The generated page confirms that the server is running and prints its hostname.

### Why User Data matters for Auto Scaling

Without bootstrap automation, an Auto Scaling Group could successfully create a new EC2 instance that is technically `Running` but has no Apache service or application page. The ALB health check would then fail and the new capacity would not actually serve users.

User data turns a generic AMI into an application-ready server automatically:

```text
Auto Scaling decides capacity is needed
              |
              v
Launch Template creates EC2
              |
              v
User Data runs automatically
              |
      +-------+--------+
      |                |
      v                v
Install Apache     Create web page
      |                |
      +-------+--------+
              |
              v
Start HTTP service on port 80
              |
              v
Target Group health check
              |
              v
Healthy target can receive ALB traffic
```

### Why not configure the server manually?

Manual configuration does not scale. If five replacement instances were launched during a traffic spike, manually logging into each server to install and configure Apache would defeat the purpose of Auto Scaling.

The bootstrap script makes instance creation **repeatable, automatic, and disposable**: any instance can be terminated and recreated from the same definition.

### Important Base64 detail

The AWS console offers an option indicating that user data has **already been Base64 encoded**. This option was intentionally left disabled because the content entered here is plain Bash text. AWS handles the required encoding when the Launch Template is created.

Marking plain Bash as already encoded could prevent cloud-init from receiving the intended script correctly, causing new instances to launch without the expected application bootstrap.

### Verification plan

The Launch Template is not considered fully validated merely because AWS accepts its configuration. After the Auto Scaling Group creates an instance, verification will include:

1. EC2 reaches the running/healthy infrastructure state.
2. Apache is started by user data.
3. The new instance registers with `web-tg`.
4. Target health becomes `Healthy`.
5. Requests through `web-alb` successfully reach Auto Scaling-managed instances.
6. Hostname output demonstrates which backend handled a request.

This separates **configuration created** from **behavior verified**, which is important for credible engineering documentation.

---

## 11. Current Verified State

Verified so far:

- Custom VPC and public/private subnet layout
- Internet Gateway and NAT-based private outbound connectivity
- `ALB-SG -> App-SG` security boundary
- Private EC2 without public IPv4
- Apache application bootstrap on the original server
- `web-tg` target health: `Healthy`
- `web-alb`: `Active`
- External browser request through the ALB: successful

In progress:

- `web-launch-template`
- Auto Scaling application tier

The next milestone is to create the Launch Template successfully, then create an Auto Scaling Group using the private subnets and integrate it with `web-tg`.
