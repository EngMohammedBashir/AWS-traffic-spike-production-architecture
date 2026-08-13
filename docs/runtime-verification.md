# Runtime Verification Log

This document records the runtime evidence collected after the initial infrastructure configuration. It intentionally separates **Configured** from **Verified** behavior.

## 1. Auto Scaling baseline verified

The manually created legacy `web-server` target was deregistered from `web-tg` so the load-balancing pool contained only instances managed by `web-asg`.

Verified baseline:

- Auto Scaling Group: `web-asg`
- Desired capacity: `2`
- Minimum capacity: `2`
- Maximum capacity: `4`
- Two application targets registered in `web-tg`
- One target in `us-east-1a`
- One target in `us-east-1b`
- Both targets reported `Healthy`
- Repeated requests through `web-alb` returned different backend hostnames, confirming load distribution across targets

## 2. Launch Template v2 and Systems Manager

The first Auto Scaling instances were not visible in Systems Manager Fleet Manager because they did not have an IAM instance profile for SSM.

A new instance role/profile was created:

- `WebServer-SSM-Role`
- AWS managed policy: `AmazonSSMManagedInstanceCore`

An unnecessary CloudWatch Agent policy was removed because the project did not require it for this step.

A new Launch Template version was then created:

- Launch Template: `web-launch-template`
- Version: `2`
- Description: `Add SSM access for private EC2 instances`
- Version `2` was set as the default

This allows private EC2 instances to be administered through AWS Systems Manager without assigning public IP addresses or exposing SSH directly to the Internet.

## 3. Rolling Instance Refresh verified

An Auto Scaling Instance Refresh was used to replace the existing fleet with instances using Launch Template version 2.

Refresh settings included:

- Replacement type: Replace instances
- Strategy: Rolling
- Minimum healthy percentage: `100%`
- Maximum healthy percentage: `150%`
- Instance warmup: `300 seconds`
- Skip matching: Enabled
- Desired Launch Template configuration: default version `2`

The refresh completed successfully:

- Status: `Successful`
- Percentage completed: `100%`
- Instances to update: `0`

The Activity history showed the expected rolling sequence of successful launches and terminations.

## 4. SSM connectivity and Apache verified

After the Instance Refresh, Systems Manager Fleet Manager showed two managed EC2 nodes with status `Online`.

A Session Manager shell was opened to a private Auto Scaling-managed EC2 instance. Runtime checks confirmed:

- Private hostname resolution
- Apache running successfully
- The project page available locally
- User Data still functioning after adding the SSM IAM role

This verified the management path:

```text
Administrator
    |
    v
AWS Systems Manager
    |
    v
SSM Agent + WebServer-SSM-Role
    |
    v
Private EC2
```

## 5. Controlled CPU load test

The first HTTP request-load experiment increased CPU only modestly because Apache served the static page efficiently.

The verification method was therefore changed to a controlled CPU stress test on the two baseline `t3.micro` instances through SSM. Both vCPUs on each baseline instance were intentionally kept busy during the experiment.

Inside the instances, `top` showed approximately zero idle CPU while the stress was active.

CloudWatch then showed the Auto Scaling group's **Average CPU Utilization** rise above the configured Target Tracking value of `50%`.

A captured data point reached approximately:

- Average CPU Utilization: `71.68%`

## 6. Scale-out verified

The Auto Scaling Activity history recorded the actual scaling trigger:

```text
CloudWatch Target Tracking alarm entered ALARM
        |
        v
cpu-target-tracking executed
        |
        v
Desired capacity changed 2 -> 3
        |
        v
New EC2 instance launched
```

Because CPU pressure continued, the Auto Scaling Group scaled again until it reached the configured maximum capacity of `4`.

Final scale-out verification in `web-tg`:

- Total targets: `4`
- Healthy: `4`
- Unhealthy: `0`
- Initial: `0`
- Draining: `0`

One newly launched target briefly appeared unhealthy during initialization. Direct SSM inspection confirmed Apache was active and the local application endpoint returned HTTP 200. After the target-group health-check cycle caught up, the target became healthy.

The verified scale-out chain is therefore:

```text
CPU load
   |
   v
Average CPU > 50%
   |
   v
Target Tracking alarm
   |
   v
web-asg: 2 -> 3 -> 4
   |
   v
New instances bootstrap through User Data
   |
   v
web-tg health checks pass
   |
   v
4 healthy application targets
```

## 7. Scale-in verification in progress

After scale-out was proven, the synthetic CPU workload was stopped on both stressed baseline instances.

CloudWatch then showed Average CPU Utilization falling from the elevated test range. The graph visibly declined from roughly the mid-80% range toward the mid-40% range.

Expected next behavior:

```text
CPU load removed
      |
      v
Average CPU falls
      |
      v
Target Tracking evaluates lower demand
      |
      v
Expected: 4 -> 3 -> 2
```

**Scale-in is not marked Verified yet.** It will only be marked verified after Auto Scaling Activity confirms the automatic scale-in actions and the group returns to its configured minimum capacity of two healthy targets.

## Current verification status

### Verified

- Auto Scaling baseline with two healthy targets across two AZs
- ALB distributing requests across Auto Scaling-managed backends
- Launch Template v2 with SSM access
- Rolling Instance Refresh completed 100%
- Two private EC2 nodes online in Systems Manager
- Session Manager access without public SSH exposure
- Apache and User Data verified after the refresh
- Controlled CPU stress successfully raised ASG Average CPU above 50%
- Target Tracking policy triggered automatically
- Desired capacity increased from 2 to 3
- Continued scale-out reached maximum capacity 4
- All four targets became healthy in `web-tg`
- Synthetic load removed
- CPU metric observed decreasing after load removal

### Pending

- Automatic scale-in from 4 to 3
- Automatic scale-in from 3 to 2
- Final confirmation of two healthy targets after scale-in

The full elasticity cycle will only be marked complete after the following sequence is observed automatically:

```text
2 -> 3 -> 4 -> 3 -> 2
```
