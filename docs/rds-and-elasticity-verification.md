# RDS Verification

This document records the implementation and runtime verification of the private RDS database tier for the AWS Traffic Spike project.

## Private RDS network design

A dedicated RDS DB subnet group was created:

- Name: `web-db-subnet-group`
- `private-subnet-a` — `us-east-1a` — `10.0.16.0/21`
- `private-subnet-b` — `us-east-1b` — `10.0.24.0/21`

A dedicated database security group was created:

- Name: `DB-SG`
- Inbound protocol: TCP
- Port: `3306`
- Source: `App-SG`

This restricts MySQL access to the application tier instead of exposing the database directly to the Internet.

```text
Internet -> ALB -> App-SG -> Private EC2 -> DB-SG -> RDS MySQL
```

## RDS MySQL instance — Verified

The database instance was created with:

- DB identifier: `web-db`
- Engine: MySQL Community
- Engine version: `8.4.9`
- Instance class: `db.t4g.micro`
- VPC: `main-vpc`
- DB subnet group: `web-db-subnet-group`
- Public access: Disabled
- VPC security group: `DB-SG`
- Runtime status: `Available`

### Availability limitation

The AWS Free Plan exposed only the Single-AZ deployment option. Therefore the implemented RDS instance is **Single-AZ**.

The production architecture target remains **Multi-AZ**, but this repository does not claim Multi-AZ was implemented when the account plan did not make that deployment option available.

---

## EC2-to-RDS connectivity — Verified

A private Auto Scaling-managed EC2 instance was accessed through AWS Systems Manager Session Manager.

Before installing a MySQL client, TCP reachability was tested directly with Bash:

```bash
timeout 5 bash -c '</dev/tcp/web-db.<redacted>.us-east-1.rds.amazonaws.com/3306' \
  && echo "SUCCESS: RDS reachable" \
  || echo "FAILED"
```

Observed result:

```text
SUCCESS: RDS reachable
```

This verified the network path before testing database authentication:

```text
Private EC2
   |
   v
App-SG
   |
   v
TCP/3306
   |
   v
DB-SG
   |
   v
Private RDS MySQL
```

The first `nc` test could not run because Netcat was not installed. Using Bash `/dev/tcp` provided a lightweight alternative without changing the network configuration.

---

## MySQL client and authentication — Verified

A MySQL-compatible client was installed on Amazon Linux 2023:

```bash
sudo dnf install -y mariadb105
```

Verification:

```bash
mysql --version
```

The client then connected to the private RDS endpoint:

```bash
mysql -h web-db.<redacted>.us-east-1.rds.amazonaws.com \
      -P 3306 \
      -u admin \
      -p
```

The server reported MySQL `8.4.9`, proving that the test moved beyond basic port reachability into an authenticated SQL session.

> Database passwords are intentionally excluded from this repository and from screenshots.

---

## Database create, write, and read — Verified

A project database was created:

```sql
CREATE DATABASE webapp;
```

The active database was selected:

```sql
USE webapp;
```

A test table was created:

```sql
CREATE TABLE visits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

A row was written:

```sql
INSERT INTO visits (message)
VALUES ('Hello from AWS Traffic Spike Project');
```

The data was read back:

```sql
SELECT * FROM visits;
```

Verified application data included:

```text
id: 1
message: Hello from AWS Traffic Spike Project
```

The runtime chain was therefore verified:

```text
EC2
 |
 v
Reach RDS on 3306
 |
 v
Authenticate to MySQL
 |
 v
Create database
 |
 v
Create table
 |
 v
Write data
 |
 v
Read data
```

---

## PHP application integration — Verified on a private EC2 instance

The original Auto Scaling bootstrap installed Apache and served static HTML. To let the application tier communicate with MySQL, PHP and the MySQL driver were added:

```bash
sudo dnf install -y php php-mysqlnd
```

PHP runtime verification:

```bash
php -v
```

Installed MySQL extensions were confirmed with:

```bash
php -m | grep -E 'mysqli|pdo_mysql'
```

Observed modules:

```text
mysqli
pdo_mysql
```

This means the application server had both the PHP runtime and the database connector required to communicate with RDS.

### First PHP-to-RDS read test

A temporary application endpoint was created at:

```text
/var/www/html/db-test.php
```

The script used PHP `mysqli` to connect to `webapp`, execute a `SELECT`, and render the database result as HTML.

Local validation from the EC2 instance:

```bash
sudo systemctl restart httpd
curl -i http://localhost/db-test.php
```

The response included:

```text
HTTP/1.1 200 OK
X-Powered-By: PHP/8.5.9
RDS Connection: SUCCESS
Hello from AWS Traffic Spike Project
```

This verified the application path:

```text
Apache -> PHP -> mysqli -> RDS MySQL -> SELECT -> HTML response
```

### Application-driven write + read test

The PHP endpoint was then changed so that every request:

1. inserts a new row into `webapp.visits`,
2. counts the total rows,
3. returns the current record count,
4. displays the backend EC2 hostname.

The important application logic is intentionally shown without credentials:

```php
$conn = new mysqli($host, $user, $password, $database);

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
```

Local application verification:

```bash
curl http://localhost/db-test.php
```

Observed output included:

```text
RDS Connection: SUCCESS
Total database records: 2
Backend: ip-10-0-19-196.ec2.internal
```

Repeating the request increased the database record count, proving that the **web application itself** was performing both writes and reads against the shared RDS database.

---

## ALB integration exposed configuration drift

The PHP endpoint was then tested through the Application Load Balancer.

One request succeeded and increased the RDS record count, while another refresh returned:

```text
404 Not Found
```

The target group still showed two healthy EC2 targets in separate Availability Zones.

This was an important finding: the ALB health check only proved that the normal application root (`/`) was healthy. It did **not** prove that both instances had the newly created `/db-test.php` file.

Because the PHP application had initially been configured manually on only one EC2 instance, the fleet had configuration drift:

```text
                 ALB
                /   \
               v     v
        EC2-A           EC2-B
        PHP app ✅      static app only ❌
             \           /
              \         /
               v       v
                 RDS
```

This is exactly the kind of inconsistency that immutable infrastructure and Auto Scaling bootstrap automation are intended to prevent.

---

## Launch Template v3 — Application bootstrap configured

To remove the fleet inconsistency, a new Launch Template version was created from **Version 2**, preserving the existing SSM instance profile.

Launch Template:

- Name: `web-launch-template`
- Version: `3`
- Source version: `2`
- Description: `Add PHP and RDS web application bootstrap`
- IAM instance profile: `WebServer-SSM-Role`
- Security group: `App-SG`
- Subnet: intentionally omitted so `web-asg` continues to control placement across both private subnets
- Version `3` set as default

The updated User Data installs:

```bash
dnf install -y httpd php php-mysqlnd
```

It then creates both:

```text
/var/www/html/index.html
/var/www/html/db-test.php
```

The purpose is to make every Auto Scaling-created instance reproducible:

```text
Launch Template v3
      |
      v
New EC2
      |
      +--> Apache
      +--> PHP
      +--> MySQL driver
      +--> application files
      |
      v
Same application on every backend
```

### Credential handling note

For this lab, database password authentication is being used to complete the application integration path. Credentials must never be committed to GitHub or exposed in screenshots.

A stronger production implementation would move database credentials out of User Data/application source and into a secrets-management mechanism such as AWS Secrets Manager or another controlled secret-delivery workflow. IAM database authentication was also considered as a future hardening option, but it is not claimed as implemented in this lab.

---

## Instance Refresh for Launch Template v3 — In progress / pending verification

The Auto Scaling Group will use an Instance Refresh to replace Version 2 instances with Version 3 instances.

Planned refresh configuration:

- Replace instances
- Minimum healthy percentage: `100%`
- Maximum healthy percentage: `150%`
- Instance warmup: `300 seconds`
- Skip matching: Enabled
- Desired configuration: `web-launch-template` Version `3`

The intended rollout is:

```text
Existing v2 instance
        |
        v
Launch v3 replacement
        |
        v
Run User Data
        |
        v
Apache + PHP + db-test.php ready
        |
        v
Target becomes Healthy
        |
        v
Terminate old v2 instance
```

**Version 3 application rollout is configured but not yet marked Verified.** It will be marked verified only after both Auto Scaling backends are replaced and repeated requests through the ALB succeed consistently while sharing the same RDS state.

---

## Current verified database/application architecture

```text
Internet
   |
   v
Application Load Balancer
   |
   v
Auto Scaling EC2 tier in private subnets
   |
   +--> Apache
   +--> PHP
   +--> mysqli / pdo_mysql
   |
   v
App-SG
   |
   v
DB-SG : TCP/3306
   |
   v
Private RDS MySQL
   |
   v
webapp.visits
```

## Verification status

### Verified

- Private RDS networking
- DB subnet group across two private subnets
- `DB-SG` allows MySQL only from `App-SG`
- RDS public access disabled
- EC2-to-RDS TCP/3306 reachability
- MySQL authentication
- `webapp` database creation
- `visits` table creation
- SQL INSERT and SELECT
- PHP 8.5 runtime installed
- `mysqli` and `pdo_mysql` available
- PHP application can read from RDS
- PHP application can insert into RDS
- Application can display the shared database record count
- ALB test exposed backend configuration drift
- Launch Template Version 3 created to automate a consistent application bootstrap

### Pending verification

- Complete Instance Refresh to Version 3
- Both v3 targets healthy
- `/db-test.php` available on both backends
- Repeated ALB requests succeed consistently
- Backend hostname changes while database counter remains shared

The next milestone is to complete the Version 3 Instance Refresh and verify the full path repeatedly:

```text
Internet -> ALB -> any Auto Scaling EC2 -> PHP -> RDS -> shared state
```
