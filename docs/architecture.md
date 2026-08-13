# Implemented AWS Architecture Walkthrough

This document describes the final architecture that was actually implemented and verified before cleanup. It intentionally separates **Implemented / Verified** features from **Production Hardening Recommendations**.

## 1. Final Verified Architecture

```text
Internet
   |
   v
Application Load Balancer
(public-subnet-a + public-subnet-b / ALB-SG)
   |
   v
web-tg (HTTP/80)
   |
   v
web-asg
   |
   +--> Private EC2 / us-east-1a (App-SG)
   +--> Private EC2 / us-east-1b (App-SG)
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

Systems Manager -> private EC2 administration
CloudWatch -> scaling metrics + unhealthy-target alarm
SNS -> email notification
```

The ALB was the only public application entry point. EC2 and RDS remained private.

## 2. Network Design

- VPC: `main-vpc`
- CIDR: `10.0.0.0/16`
- Region: `us-east-1`

| Subnet | CIDR | AZ | Role |
|---|---|---|---|
| `public-subnet-a` | `10.0.0.0/21` | `us-east-1a` | ALB / NAT-facing resources |
| `public-subnet-b` | `10.0.8.0/21` | `us-east-1b` | ALB resources |
| `private-subnet-a` | `10.0.16.0/21` | `us-east-1a` | Application / RDS placement |
| `private-subnet-b` | `10.0.24.0/21` | `us-east-1b` | Application / RDS placement |

Public routing used an Internet Gateway. Private EC2 used a NAT Gateway for outbound package installation and updates without receiving public IPv4 addresses.

## 3. Security Boundaries

```text
Internet
   |
   v
ALB-SG
   |
   v
App-SG
   |
   | TCP/3306 only
   v
DB-SG
   |
   v
Private RDS
```

- `ALB-SG` protected the public load balancer.
- `App-SG` allowed application traffic from the ALB rather than directly from the Internet.
- `DB-SG` allowed MySQL/TCP 3306 only from `App-SG`.
- RDS public access was disabled.

## 4. Load Balancing and Auto Scaling

### Application Load Balancer

- Internet-facing
- Deployed across both public subnets
- Listener: HTTP/80
- Forwarded to `web-tg`

### Auto Scaling Group

- Desired: `2`
- Minimum: `2`
- Maximum: `4`
- Private subnet placement across two Availability Zones
- Health checks: EC2 + ELB
- Health check grace period: `300 seconds`

Target Tracking policy:

- Metric: Average CPU Utilization
- Target: `50%`
- Instance warmup: `300 seconds`
- Scale-in enabled

### Verified elasticity

Controlled CPU load produced the complete cycle:

```text
2 -> 3 -> 4 -> 3 -> 2
```

This proved both automatic scale-out and automatic scale-in.

## 5. Private EC2 Administration

AWS Systems Manager Session Manager was used instead of public SSH access.

```text
Administrator
   |
   v
Systems Manager
   |
   v
IAM instance role + SSM Agent
   |
   v
Private EC2
```

The project IAM role used `AmazonSSMManagedInstanceCore` during the lab and was deleted during cleanup.

## 6. Private RDS MySQL

Implemented configuration:

- DB identifier: `web-db`
- Engine: MySQL Community `8.4.9`
- Instance class: `db.t4g.micro`
- Public access: Disabled
- DB subnet group: `web-db-subnet-group`
- Security group: `DB-SG`

The DB subnet group included both private subnets. The actual DB instance used Single-AZ because the account Free Plan restricted the available deployment options.

## 7. EC2-to-RDS Verification

TCP reachability was verified from private EC2:

```bash
timeout 5 bash -c '</dev/tcp/<RDS-ENDPOINT>/3306' \
  && echo "SUCCESS: RDS reachable" \
  || echo "FAILED"
```

Observed result:

```text
SUCCESS: RDS reachable
```

A MySQL-compatible client was then used to authenticate:

```bash
mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p
```

## 8. Database Verification

```sql
CREATE DATABASE webapp;
USE webapp;

CREATE TABLE visits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO visits (message)
VALUES ('Hello from AWS Traffic Spike Project');

SELECT * FROM visits;
```

INSERT and SELECT were both verified successfully.

## 9. Reproducible Application Bootstrap

The final Launch Template bootstrap installed Apache, PHP, and the MySQL PHP extension, then created the RDS-backed dashboard automatically.

The important engineering point is not the internal Launch Template version number. The final state was **reproducible infrastructure**: any replacement or scaled EC2 instance could recreate the same application automatically.

> Security note: the real database password is not committed to this repository. `YOUR_DB_PASSWORD` below is a placeholder. Production credential handling is documented later.

```bash
#!/bin/bash

dnf update -y
dnf install -y httpd php php-mysqlnd

systemctl enable httpd
systemctl start httpd

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

$stmt = $conn->prepare("INSERT INTO visits (message) VALUES (?)");
$message = "Website visit through ALB";
$stmt->bind_param("s", $message);
$stmt->execute();

$result = $conn->query("SELECT COUNT(*) AS total FROM visits");
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
body { font-family: Arial, sans-serif; background: #f4f6f8; margin: 0; padding: 40px; }
.container { max-width: 900px; margin: auto; }
h1 { text-align: center; margin-bottom: 10px; }
.subtitle { text-align: center; color: #666; margin-bottom: 30px; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 20px; }
.card { background: white; border-radius: 12px; padding: 24px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); }
.label { color: #666; font-size: 14px; margin-bottom: 8px; }
.value { font-size: 22px; font-weight: bold; word-break: break-word; }
.success { color: #16833a; }
.footer { margin-top: 30px; text-align: center; color: #777; font-size: 13px; }
.button-container { text-align: center; }
.button { display: inline-block; margin-top: 30px; padding: 12px 20px; background: #222; color: white; text-decoration: none; border-radius: 8px; }
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
    <div class="footer">ALB → Auto Scaling EC2 → PHP → Private RDS MySQL</div>
</div>
</body>
</html>
PHP

chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html
systemctl restart httpd
```

## 10. Configuration Drift Incident

A temporary failure occurred after the PHP application was manually added to only one backend.

Observed behavior:

```text
Request -> Backend A -> Success
Request -> Backend B -> 404 Not Found
Request -> Backend A -> Success
```

Both targets were still reported healthy because the health check only verified `/`.

Root cause: application configuration drift between Auto Scaling instances.

Final fix:

```text
Application bootstrap
      |
      v
Launch Template
      |
      v
Rolling Instance Refresh
      |
      v
Consistent EC2 fleet
```

After the fleet refresh, repeated ALB requests no longer produced intermittent 404 responses.

## 11. End-to-End Application Verification

The final dashboard was tested after the last rolling refresh.

Successive requests showed:

- RDS connection status: connected
- Increasing shared visit count
- Different backend EC2 hostnames
- No intermittent 404 responses

Verified path:

```text
Browser
   -> ALB
   -> Auto Scaling EC2 A or B
   -> Apache/PHP
   -> Private RDS MySQL
   -> webapp.visits
```

This demonstrated load balancing plus shared persistent database state.

## 12. Monitoring and Alerting

A separate unhealthy-target alarm was configured:

- Metric: `AWS/ApplicationELB / UnHealthyHostCount`
- Statistic: `Maximum`
- Period: `1 minute`
- Condition: `>= 1`
- Datapoints: `1 out of 1`
- Missing data: treated as not breaching
- Action: SNS email notification

```text
Unhealthy backend -> CloudWatch Alarm -> SNS -> Email
```

Target Tracking CloudWatch alarms were also used during Auto Scaling.

## 13. Production Hardening Recommendations

These improvements are **recommended, not claimed as implemented**.

### HTTPS + ACM

Production should terminate TLS on the ALB using an ACM certificate and redirect HTTP/80 to HTTPS/443.

Not implemented because no owned project domain was available for DNS validation.

### AWS WAF

Recommended in front of the ALB for managed application-layer protections and rate-based controls where justified.

```text
Internet -> AWS WAF -> ALB -> Application
```

Not enabled in the lab to avoid adding unnecessary cost solely for portfolio demonstration.

### Database credentials

The lab used password-based DB authentication to prove the integration path.

Production should avoid long-lived credentials in User Data or application source files. Preferred controls include:

- AWS Secrets Manager
- IAM Database Authentication where appropriate
- Least-privilege IAM permissions
- Dedicated application DB user instead of the master user
- TLS for database connections
- Credential rotation

### RDS Multi-AZ

Recommended for production database availability and failover. The lab remained Single-AZ because the Free Plan restricted available deployment options.

## 14. Cleanup and Cost Control

After final verification, temporary resources were removed.

Cleanup verification included:

- Auto Scaling capacity reduced to zero and ASG deleted
- Remaining project EC2 terminated
- ALB deleted
- NAT Gateway deleted
- RDS deleted without retaining a final snapshot or automated backups
- Target group deleted
- Launch Template deleted
- Project security groups deleted
- Project IAM role deleted
- CloudWatch alarms removed
- SNS project topic removed
- Elastic IP addresses: `0`
- EBS volumes: `0`
- RDS manual snapshots: `0`

AWS Resource Explorer briefly showed stale deleted resources while its index was still updating, so live service consoles were used as the source of truth.

At the final billing check, AWS Bills showed an estimated grand total of `USD 0.00` under the account Free Plan.

## 15. Final Outcome

The project demonstrated the full engineering lifecycle:

```text
Design
  -> Build
  -> Secure
  -> Scale
  -> Monitor
  -> Integrate Data
  -> Diagnose Drift
  -> Verify
  -> Clean Up
```

The implementation is complete and the temporary AWS environment has been cleaned up.
