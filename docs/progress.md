# Project Progress

## Current Status

**Phase 1 — Networking Baseline (in progress)**

- [x] Repository created
- [x] Project README initialized
- [x] Root account MFA enabled
- [x] AWS monthly cost budget created: `$10`
- [x] Budget alert: Actual cost > 80% (`$8`)
- [x] Budget alert: Actual cost > 100% (`$10`)
- [x] Budget alert: Forecasted cost > 100% (`$10`)
- [x] Budget notifications configured
- [x] IAM administrative user created for routine console work
- [x] IAM administrative group created and AdministratorAccess attached
- [x] Root account signed out; routine work moved to IAM user
- [x] Project Region confirmed: `us-east-1` (N. Virginia)
- [x] Custom VPC created: `main-vpc`
- [x] VPC IPv4 CIDR: `10.0.0.0/16`
- [x] Public subnet A created in `us-east-1a`: `10.0.0.0/21`
- [x] Public subnet B created in `us-east-1b`: `10.0.8.0/21`
- [x] Private subnet A created in `us-east-1a`: `10.0.16.0/21`
- [x] Private subnet B created in `us-east-1b`: `10.0.24.0/21`
- [ ] Route tables and Internet Gateway
- [ ] NAT Gateway and private routing
- [ ] Security Groups and workload IAM roles

## Current Network Layout

```text
main-vpc — 10.0.0.0/16
│
├── us-east-1a
│   ├── public-subnet-a   10.0.0.0/21
│   └── private-subnet-a  10.0.16.0/21
│
└── us-east-1b
    ├── public-subnet-b   10.0.8.0/21
    └── private-subnet-b  10.0.24.0/21
```

> Note: These are the four application/public subnets created so far. Database/cache isolation subnets, if required by the final architecture, will be designed deliberately rather than assumed.

## Rules

- One implementation step at a time.
- Do not move forward until the current step is verified.
- GitHub is the source of truth for project state.
- Sync meaningful completed steps to this repository as soon as they are verified.
- Record important architecture/security/cost decisions and their trade-offs.
- Record actual measurements; never invent performance, recovery, availability, or cost results.
- Track cleanup after every AWS lab session.
- Warn before creating resources that generate ongoing cost.
- Do not commit passwords, access keys, account IDs, secret values, or other credentials to GitHub.

## Account Safety Status

```text
AWS Account
├── Root MFA                 ✅ Enabled
├── Monthly Budget           ✅ $10
│   ├── Actual > $8          ✅ Alert
│   ├── Actual > $10         ✅ Alert
│   └── Forecast > $10       ✅ Alert
├── Routine admin identity   ✅ IAM admin user
├── Region                   ✅ us-east-1
└── Project infrastructure
    ├── VPC                  ✅ Created
    └── Subnets              ✅ 4 created across 2 AZs
```

## Session Log

### Session 0 — Repository Initialization
- Repository connected and initialized.
- README and progress tracker created.
- No AWS infrastructure created.

### Session 1 — AWS Account Guardrails
- Enabled MFA for the AWS root user.
- Created `AWS-Traffic-Spike-Project-Budget` with a monthly budget of `$10`.
- Configured three budget alerts at `$8 actual`, `$10 actual`, and `$10 forecasted`.
- No automatic budget actions attached.

### Session 2 — IAM & Networking Start
- Created an IAM user for daily AWS Console administration.
- Created an admin group and attached the AWS managed `AdministratorAccess` policy.
- Signed out of the root user and continued with the IAM user.
- Switched the project console Region to `us-east-1` (N. Virginia).
- Created `main-vpc` with CIDR `10.0.0.0/16`.
- Created two public and two private subnets across `us-east-1a` and `us-east-1b`.
- Corrected the second public subnet CIDR during creation to avoid CIDR overlap.
- Captured AWS Console screenshots locally as implementation evidence; image files are not yet committed to the repository.

## Next Verified Step

Create and attach the Internet Gateway, then build the public route table. No NAT Gateway will be created until its ongoing cost is explicitly reviewed immediately before creation.
