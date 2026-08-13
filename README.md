# Production-Grade AWS Architecture — Traffic Spike & Reliability

## 🎯 Project Goal

Solve a real-world e-commerce reliability problem: sudden traffic spikes cause a legacy single-server application to suffer from high CPU/memory usage, slow responses, HTTP 5xx errors, and database overload.

## 🏢 Business Problem

The legacy architecture has:
- A single application server (SPOF)
- Database exposure and weak network isolation
- No automatic horizontal scaling
- No controlled handling of traffic/write bursts
- Limited monitoring and incident visibility

## 🏗️ Target Architecture

```text
Internet
   │
Route 53 / ALB DNS
   │
CloudFront + WAF
   │
Application Load Balancer
   │
EC2 Auto Scaling Group
   │
├── AZ-a
├── AZ-b
   │
   ├── ElastiCache Redis
   ├── SQS + Lambda workers
   └── Aurora MySQL
```

> The exact services and architecture are validated during implementation. We will not add a service merely to make the diagram look more "production-grade".

## 📚 Project Phases

1. Architecture & cost planning
2. VPC, subnets, route tables, and Internet Gateway
3. NAT Gateway, Security Groups, and IAM
4. EC2 and Session Manager
5. Aurora/RDS database layer
6. ALB and health checks
7. Auto Scaling
8. Monitoring and alerting
9. Failure testing and measured recovery
10. Documentation, interview preparation, and final cleanup

## 🧪 Engineering Rule

Every important architectural claim must be demonstrated or measured where practical. No invented performance, availability, recovery-time, or cost numbers.

## 💰 Cost Rule

Resources that generate ongoing charges must be tracked. Cost estimates will be compared with actual AWS billing data, and temporary resources will be deleted immediately after testing.

## 📁 Documentation

- `docs/architecture.md` — architecture and traffic flow
- `docs/decisions.md` — architectural decisions and trade-offs
- `docs/cost.md` — estimated vs actual cost
- `docs/failure-tests.md` — controlled failure experiments
- `docs/runbook.md` — incident response procedures
- `docs/postmortem.md` — final incident analysis
- `docs/interview.md` — interview questions and answers
- `docs/progress.md` — project progress and completed steps
- `screenshots/` — evidence from AWS Console

## 👨‍💻 Learning Mode

This project is being built by a beginner Cloud Engineer with theoretical AWS SAA-C03 knowledge. Implementation is intentionally step-by-step: one task at a time, with an explanation of what is being created, why it exists, how to verify it, and what could go wrong.
