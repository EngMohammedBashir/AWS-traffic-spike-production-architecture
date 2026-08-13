# RDS and Elasticity Verification

This document records the latest runtime verification results for the AWS Traffic Spike project.

## Auto Scaling elasticity cycle — Verified

The `web-asg` Target Tracking policy uses Average CPU Utilization with a target of `50%`, minimum capacity `2`, desired baseline `2`, and maximum capacity `4`.

A controlled CPU stress test was run on both baseline `t3.micro` instances through AWS Systems Manager. CloudWatch showed group Average CPU Utilization above the target, including a captured value around `71.68%`.

The Target Tracking high alarm triggered automatic scale-out:

```text
2 -> 3 -> 4
```

All four instances eventually became healthy in `web-tg`.

After the synthetic CPU load was stopped, CloudWatch showed CPU utilization falling. The Target Tracking low alarm then triggered automatic scale-in:

```text
4 -> 3 -> 2
```

The full elasticity cycle is therefore verified:

```text
2 -> 3 -> 4 -> 3 -> 2
```

No manual desired-capacity changes were used to produce the scale-in result.

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

## EC2-to-RDS connectivity — Verified

From a private Auto Scaling-managed EC2 instance accessed through Systems Manager, TCP connectivity to the RDS endpoint on port `3306` was tested successfully.

Observed result:

```text
SUCCESS: RDS reachable
```

This verified the network path:

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

## MySQL authentication — Verified

A MariaDB/MySQL-compatible command-line client was installed on the EC2 instance.

The EC2 instance then authenticated successfully to the RDS MySQL endpoint using the configured database credentials.

The server reported MySQL version `8.4.9`, proving that the test moved beyond basic port reachability into an actual authenticated SQL session.

## Database create, write, and read — Verified

A project database was created:

```sql
CREATE DATABASE webapp;
```

A table was then created:

```sql
CREATE TABLE visits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

A test row was inserted:

```sql
INSERT INTO visits (message)
VALUES ('Hello from AWS Traffic Spike Project');
```

The row was read back successfully:

```sql
SELECT * FROM visits;
```

Verified application data:

```text
id: 1
message: Hello from AWS Traffic Spike Project
```

This proves the full database runtime path:

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

## Current verified architecture

```text
Internet
   |
   v
Application Load Balancer
   |
   v
Target Group
   |
   v
Auto Scaling EC2 tier in private subnets
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

## Next milestone

The next milestone is to make the web application itself read from and write to RDS automatically instead of using manual SQL commands from Session Manager.
