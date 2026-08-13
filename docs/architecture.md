# Implemented AWS Architecture Walkthrough

This document reflects the architecture as it is currently implemented and verified. It intentionally separates **Implemented / Verified** features from **Production Hardening Recommendations** that were evaluated but not deployed in this lab.

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
   +--> Private EC2 in us-east-1a (App-SG)
   |
   +--> Private EC2 in us-east-1b (App-SG)
             |
             +--> NAT Gateway for outbound package/update access
             |
             v
          DB-SG : TCP/3306
             |
             v
        web-db (Private RDS MySQL)
             |
             v
        webapp.visits

CloudWatch + SNS monitor application health.
AWS Systems Manager provides private EC2 administration without public SSH exposure.
```

## 2. VPC and Subnet Design

- VPC: `main-vpc`
- CIDR: `10.0.0.0/16`
- Region: `us-east-1`

| Subnet | CIDR | AZ | Role |
|---|---|---|---|
| `public-subnet-a` | `10.0.0.0/21` | `us-east-1a` | Public ALB / NAT-facing resources |
| `public-subnet-b` | `10.0.8.0/21` | `us-east-1b` | Public ALB resources |
| `private-subnet-a` | `10.0.16.0/21` | `us-east-1a` | Private application / RDS placement |
| `private-subnet-b` | `10.0.24.0/21` | `us-east-1b` | Private application / RDS placement |

The public subnets act as controlled entrances, while the application and database tiers remain private.

## 3. Routing and Internet Access

The public route table sends `0.0.0.0/0` to the Internet Gateway. The private subnets use a NAT Gateway for outbound connectivity.

```text
Private EC2 -> NAT Gateway -> Internet     allowed outbound path
Internet -> NAT Gateway -> Private EC2     not a direct inbound path
```

This lets private instances install packages and receive updates without assigning them public IPv4 addresses.

## 4. Security Groups

### `ALB-SG`

Public-facing security boundary for `web-alb`.

### `App-SG`

Allows application traffic from the ALB rather than from the entire Internet.

```text
Internet -> ALB-SG -> App-SG -> Private EC2
```

### `DB-SG`

Allows MySQL only from `App-SG` on TCP/3306.

```text
Private EC2 / App-SG -> TCP 3306 -> DB-SG -> RDS MySQL
```

The RDS instance is not publicly exposed.

## 5. Application Load Balancer and Target Group

### `web-alb`

- Internet-facing
- IPv4
- `public-subnet-a` + `public-subnet-b`
- Security group: `ALB-SG`
- Listener: HTTP/80
- Default action: forward to `web-tg`

### `web-tg`

- Target type: Instance
- Protocol: HTTP
- Port: 80
- Health check: HTTP `/`

The final runtime state showed two healthy targets across two Availability Zones.

## 6. Auto Scaling Group — `web-asg`

Configured values:

- Desired capacity: `2`
- Minimum capacity: `2`
- Maximum capacity: `4`
- Private subnet placement across `us-east-1a` and `us-east-1b`
- Target group: `web-tg`
- Health checks: EC2 + ELB
- Health check grace period: `300 seconds`

Target Tracking policy:

- Policy: `cpu-target-tracking`
- Metric: Average CPU Utilization
- Target: `50%`
- Instance warmup: `300 seconds`
- Scale-in: Enabled

### Verified elasticity cycle

A controlled CPU stress test verified the complete cycle:

```text
2 -> 3 -> 4 -> 3 -> 2
```

This proved both automatic scale-out and automatic scale-in while respecting the configured Min/Max boundaries.

## 7. Systems Manager and IAM Instance Role

Launch Template version 2 introduced the IAM instance profile:

- `WebServer-SSM-Role`
- Managed policy: `AmazonSSMManagedInstanceCore`

This enabled Session Manager access to private instances without exposing SSH to the Internet.

```text
Administrator
    |
    v
AWS Systems Manager
    |
    v
SSM Agent + IAM Role
    |
    v
Private EC2
```

## 8. Private RDS MySQL Tier

RDS configuration:

- DB identifier: `web-db`
- Engine: MySQL Community
- Engine version: `8.4.9`
- Instance class: `db.t4g.micro`
- VPC: `main-vpc`
- DB subnet group: `web-db-subnet-group`
- Public access: Disabled
- Security group: `DB-SG`

DB subnet group:

- `private-subnet-a`
- `private-subnet-b`

The Free Plan exposed only the Single-AZ deployment option, so the lab implementation remains Single-AZ. Multi-AZ is documented later as a production hardening recommendation.

## 9. EC2-to-RDS Runtime Verification

Basic TCP reachability was verified from a private Auto Scaling-managed EC2 instance:

```bash
timeout 5 bash -c '</dev/tcp/<RDS-ENDPOINT>/3306' \
  && echo "SUCCESS: RDS reachable" \
  || echo "FAILED"
```

Observed result:

```text
SUCCESS: RDS reachable
```

A MariaDB/MySQL-compatible client was then installed and used to authenticate to RDS:

```bash
sudo dnf install -y mariadb105
mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p
```

The server reported MySQL `8.4.9`.

## 10. Application Database Verification

A project database and table were created:

```sql
CREATE DATABASE webapp;
USE webapp;

CREATE TABLE visits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Write and read behavior were verified:

```sql
INSERT INTO visits (message)
VALUES ('Hello from AWS Traffic Spike Project');

SELECT * FROM visits;
```

The row was successfully returned from RDS.

## 11. Launch Template Evolution

The Launch Template evolved as the project matured:

- **v1** — Apache bootstrap and hostname test page
- **v2** — Added SSM IAM instance profile
- **v3** — Added PHP, MySQL driver, and RDS-connected application bootstrap
- **v4** — Added the final dashboard UI to make every replacement/scaled instance launch with the same application state

The subnet remains intentionally omitted from the Launch Template so the Auto Scaling Group controls placement across both private subnets/AZs.

## 12. Current User Data — v4

The current Launch Template bootstrap installs Apache, PHP, the MySQL PHP extension, generates the landing page, and creates the RDS-backed dashboard automatically.

> **Security note:** The real database password is intentionally omitted from this repository. `YOUR_DB_PASSWORD` is a placeholder only. The current lab used password-based authentication for demonstration; production alternatives are documented below.

```bash
#!/bin/bash

# Update packages
dnf update -y

# Install Apache, PHP, and MySQL PHP driver
dnf install -y httpd php php-mysqlnd

# Enable and start Apache
systemctl enable httpd
systemctl start httpd

# Create landing page
cat > /var/www/html/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>AWS Traffic Spike Project</title>
</head>
<body>
    <h1>AWS Traffic Spike Project</h1>
    <p>Web server is running successfully.</p>
    <p><a href="/db-test.php">Open Project Dashboard</a></p>
</body>
</html>
EOF

# Create PHP/RDS dashboard
cat > /var/www/html/db-test.php <<'PHP'
<?php

$host = "<RDS-ENDPOINT>";
$user = "admin";
$password = "YOUR_DB_PASSWORD";
$database = "webapp";

$conn = new mysqli($host, $user, $password, $database);

if ($conn->connect_error) {
    die("Database connection failed");
}

$stmt = $conn->prepare(
    "INSERT INTO visits (message) VALUES (?)"
);

$message = "Website visit through ALB";
$stmt->bind_param("s", $message);
$stmt->execute();

$result = $conn->query(
    "SELECT COUNT(*) AS total FROM visits"
);

$row = $result->fetch_assoc();
$total = $row["total"];
$backend = gethostname();
$time = date("Y-m-d H:i:s");

$stmt->close();
$conn->close();
?>

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AWS Traffic Spike Project</title>
<style>
body {
    font-family: Arial, sans-serif;
    background: #f4f6f8;
    margin: 0;
    padding: 40px;
}
.container {
    max-width: 900px;
    margin: auto;
}
h1 {
    text-align: center;
    margin-bottom: 10px;
}
.subtitle {
    text-align: center;
    color: #666;
    margin-bottom: 30px;
}
.grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
    gap: 20px;
}
.card {
    background: white;
    border-radius: 12px;
    padding: 24px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.08);
}
.label {
    color: #666;
    font-size: 14px;
    margin-bottom: 8px;
}
.value {
    font-size: 22px;
    font-weight: bold;
    word-break: break-word;
}
.success {
    color: #16833a;
}
.footer {
    margin-top: 30px;
    text-align: center;
    color: #777;
    font-size: 13px;
}
.button-container {
    text-align: center;
}
.button {
    display: inline-block;
    margin-top: 30px;
    padding: 12px 20px;
    background: #222;
    color: white;
    text-decoration: none;
    border-radius: 8px;
}
</style>
</head>
<body>
<div class="container">
    <h1>AWS Traffic Spike Project</h1>
    <div class="subtitle">Highly Available Web Architecture on AWS</div>
    <div class="grid">
        <div class="card">
            <div class="label">RDS Connection</div>
            <div class="value success">CONNECTED</div>
        </div>
        <div class="card">
            <div class="label">Total Visits</div>
            <div class="value"><?php echo $total; ?></div>
        </div>
        <div class="card">
            <div class="label">Backend Instance</div>
            <div class="value"><?php echo htmlspecialchars($backend); ?></div>
        </div>
        <div class="card">
            <div class="label">Last Request</div>
            <div class="value"><?php echo htmlspecialchars($time); ?></div>
        </div>
    </div>
    <div class="button-container">
        <a class="button" href="/db-test.php">Refresh Dashboard</a>
    </div>
    <div class="footer">
        ALB → Auto Scaling EC2 → PHP → Private RDS MySQL
    </div>
</div>
</body>
</html>
PHP

chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html
systemctl restart httpd
```

### Why v4 matters

A manual change to one or two EC2 instances is temporary because Auto Scaling can terminate and replace them at any time. Version 4 moves the final dashboard into the Launch Template so new/replacement instances are reproducible.

```text
ASG launches/replaces EC2
        |
        v
Launch Template v4
        |
        +--> Apache
        +--> PHP
        +--> php-mysqlnd
        +--> Dashboard
        +--> RDS integration
        |
        v
Healthy backend with consistent application state
```

## 13. End-to-End Application Verification

The RDS-backed PHP dashboard was verified through the ALB.

Successive browser requests showed:

- `RDS Connection: CONNECTED`
- Increasing `Total Visits`
- Different EC2 backend hostnames across refreshes
- Shared visit count persisted across different EC2 backends

This proves the full path:

```text
Browser
   |
   v
web-alb
   |
   v
web-tg
   |
   v
Auto Scaling EC2 A or B
   |
   v
Apache + PHP
   |
   v
DB-SG : 3306
   |
   v
Private RDS MySQL
   |
   v
webapp.visits
```

A temporary `404 Not Found` issue exposed configuration drift when only one backend had the new PHP file. The issue disappeared after the application bootstrap was moved into the Launch Template and the fleet was refreshed.

## 14. Monitoring and Alerting

Auto Scaling Target Tracking created CPU alarms used by the scaling policy.

A separate application-health alarm was created:

- Alarm: `web-alb-unhealthy-target-alarm`
- Namespace: `AWS/ApplicationELB`
- Metric: `UnHealthyHostCount`
- Statistic: `Maximum`
- Period: `1 minute`
- Condition: `>= 1`
- Datapoints: `1 out of 1`
- Missing data: Treat as not breaching
- Notification: SNS topic `webapp-monitoring-alerts`
- Delivery: Email subscription confirmed

Operational path:

```text
Unhealthy backend
      |
      v
ALB metric
      |
      v
CloudWatch Alarm
      |
      v
SNS
      |
      v
Email notification
```

## 15. Production Hardening Recommendations

The following improvements were deliberately **documented but not claimed as implemented**.

### HTTPS with ACM

The current lab exposes HTTP/80 through the ALB. A production deployment should use a validated domain and AWS Certificate Manager (ACM) certificate with an HTTPS/443 listener.

Recommended path:

```text
Internet
   |
   v
HTTPS :443
   |
   v
ACM certificate on ALB
   |
   v
Target Group
```

HTTP/80 should normally redirect to HTTPS/443.

This was not implemented because no project domain was available for certificate validation.

### AWS WAF

AWS WAF should be attached in front of the ALB for application-layer filtering, managed rule groups, and rate-based protection where appropriate.

```text
Internet -> AWS WAF -> ALB -> Application
```

WAF was not enabled in this lab to avoid adding a cost-bearing component solely for portfolio demonstration.

### Database credential handling

The lab used password-based database authentication to prove application-to-RDS integration.

A production implementation should **not** keep a long-lived database password in EC2 User Data or application source files.

Preferred approaches include:

- AWS Secrets Manager with an EC2 IAM role and least-privilege secret access
- IAM Database Authentication with `rds-db:connect` where appropriate
- A dedicated least-privilege database application user instead of the RDS master user
- TLS for database connections
- Credential rotation

Recommended direction:

```text
EC2 application
      |
      v
IAM Role
      |
      +--> Secrets Manager
      |       or
      +--> IAM DB Authentication
      |
      v
RDS MySQL
```

### RDS Multi-AZ

The production target is RDS Multi-AZ for database availability and failover. The current implementation remains Single-AZ because the AWS Free Plan limited the available deployment options during this lab.

The repository intentionally does not claim Multi-AZ was implemented.

## 16. Verified Project State

### Implemented and verified

- Custom VPC with 2 public + 2 private subnets across two AZs
- Internet Gateway and NAT-based outbound connectivity
- Layered `ALB-SG -> App-SG -> DB-SG` security design
- Private EC2 application tier
- Application Load Balancer and target group
- Auto Scaling Group across two private subnets
- Target Tracking CPU policy
- Full Auto Scaling cycle: `2 -> 3 -> 4 -> 3 -> 2`
- Systems Manager administration through IAM instance role
- Private RDS MySQL
- EC2-to-RDS TCP reachability
- Authenticated MySQL session
- Database/table creation
- SQL write/read verification
- PHP and MySQL driver installation
- PHP application connected to RDS
- ALB serving the RDS-backed dashboard through multiple backends
- Shared RDS state across different Auto Scaling EC2 instances
- Configuration drift detected and corrected through Launch Template versioning
- CloudWatch unhealthy-target alarm
- SNS email notification subscription
- Launch Template v4 containing the final dashboard bootstrap

### Production hardening documented, not implemented

- HTTPS/443 with ACM
- AWS WAF
- Secrets Manager / IAM DB Authentication
- Dedicated least-privilege DB application user
- RDS Multi-AZ

The project is intentionally explicit about this distinction so the repository reflects what was actually built rather than overstating production features.