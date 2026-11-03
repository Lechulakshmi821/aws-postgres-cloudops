# AWS PostgreSQL CloudOps

A small project to run PostgreSQL on AWS in a way that is secured, monitored, backed up and restorable, with the whole setup written as code (CloudFormation) and documented.

**Status:** work in progress, started 24 Oct 2026. Updated step by step.

## Goal

- Build the infrastructure with CloudFormation (no manual clicking)
- Run PostgreSQL on Amazon RDS, not reachable from the internet
- Watch it with CloudWatch metrics and alarms
- Back it up and prove the restore works (restore test with notes and timing)
- Keep costs low and document how to clean up

## Architecture

```
Me (AWS CLI) --CloudFormation--> VPC 10.0.0.0/16 (eu-central-1, Frankfurt)

  Public subnet      : test client (EC2, reached with Session Manager, no SSH keys)
                              |
                port 5432, only from the client's security group
                              v
  Private subnets A+B: RDS PostgreSQL 17 (no public access, encrypted, SSL required)

  Secrets Manager    : generated database password
  CloudWatch + SNS   : alarms and email alerts
```

## Repository layout

```
cloudformation/
  network.yaml      VPC, subnets, route table, internet gateway, security groups
  client.yaml       small Linux test client + IAM role (Session Manager, read one secret)
  database.yaml     RDS PostgreSQL, parameter group, generated password in Secrets Manager
  monitoring.yaml   SNS email alerts, 4 CloudWatch alarms, dashboard
docs/               notes, screenshots and test results
```

## Progress

| Step | What | Status |
|------|------|--------|
| 1 | Network: VPC, public and private subnets, security groups | done |
| 2 | Test client with Session Manager (no open SSH) | done |
| 3 | RDS PostgreSQL 17 in private subnets | done |
| 4 | Connection test over TLS, SSL-enforcement test | done |
| 5 | CloudWatch alarms, dashboard, email alerts (SNS) | done |
| 6 | Backups and snapshots | planned |
| 7 | Restore test with timing | planned |
| 8 | Least-privilege database user, CloudTrail | planned |

## Security decisions

- Database is in private subnets, **not publicly accessible**.
- Security group allows port 5432 **only from the client's security group**.
- Storage is **encrypted at rest**; the parameter group sets `rds.force_ssl=1`.
- Master password is **generated** and kept in **Secrets Manager**, never in the code.
- The client's IAM role can read **only that one secret**.
- Access to the client uses **Session Manager**, so no SSH keys are needed.
- Connections, disconnections and slow queries (over 1 s) are logged and sent to CloudWatch.

## Tests done

- Connected from the client with `psql` over TLS 1.3; `SHOW ssl;` returned `on`.
- Connection with `sslmode=disable` was **refused** by the server ("no pg_hba.conf entry ... no encryption"), which shows SSL is enforced.
- Created a table `customers` with 3 rows, to be used later in the restore test.
- Forced the CPU alarm into the ALARM state with `aws cloudwatch set-alarm-state`; the alert email from SNS arrived, so the whole chain works (CloudWatch alarm, SNS, email).

## Monitoring

`monitoring.yaml` creates an SNS topic with an email subscription, four CloudWatch alarms and a dashboard for the RDS instance:

| Alarm | Fires when |
|-------|-----------|
| `pg-cloudops-cpu-high` | CPU above 80 % for 10 minutes |
| `pg-cloudops-storage-low` | less than 5 GB free storage |
| `pg-cloudops-connections-high` | more than 50 connections |
| `pg-cloudops-memory-low` | less than 100 MB freeable memory |

The dashboard `pg-cloudops-postgres` shows CPU, connections, free storage and freeable memory.

Screenshots are in `docs/`: `monitoring-dashboard.png`, `monitoring-alarms.png`, `monitoring-alert-email.png`, `cloudformation-stacks.png`.

```bash
aws cloudformation deploy --region eu-central-1 --template-file cloudformation/monitoring.yaml \
  --stack-name cloudops-monitoring --parameter-overrides AlertEmail=<your-email>
# confirm the subscription email from AWS, then test:
aws cloudwatch set-alarm-state --region eu-central-1 --alarm-name pg-cloudops-cpu-high \
  --state-value ALARM --state-reason "Test"
```

## Deploy

Prerequisites: AWS CLI configured for an IAM user (not root), region `eu-central-1`.

```bash
aws cloudformation deploy --region eu-central-1 --template-file cloudformation/network.yaml \
  --stack-name cloudops-network --parameter-overrides MyIp=<your-public-ip>/32

aws cloudformation deploy --region eu-central-1 --template-file cloudformation/client.yaml \
  --stack-name cloudops-client --capabilities CAPABILITY_IAM

aws cloudformation deploy --region eu-central-1 --template-file cloudformation/database.yaml \
  --stack-name cloudops-db
```

Connect to the client, then to the database:

```bash
aws ssm start-session --region eu-central-1 --target <instance-id>
# on the client:
export AWS_DEFAULT_REGION=eu-central-1
export PGPASSWORD=$(aws secretsmanager get-secret-value --secret-id pg-cloudops/db-master \
  --query SecretString --output text | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
psql "host=<db-endpoint> dbname=appdb user=dbadmin sslmode=require"
```

## Clean up (important, to avoid costs)

Delete in this order and wait for each to finish:

```bash
aws cloudformation delete-stack --region eu-central-1 --stack-name cloudops-monitoring
aws cloudformation delete-stack --region eu-central-1 --stack-name cloudops-db
aws cloudformation delete-stack --region eu-central-1 --stack-name cloudops-client
aws cloudformation delete-stack --region eu-central-1 --stack-name cloudops-network
aws secretsmanager delete-secret --region eu-central-1 --secret-id pg-cloudops/db-master --force-delete-without-recovery
```

## Cost control

- Smallest sizes (`t3.micro`, `db.t3.micro`, 20 GB), no NAT gateway.
- Budget alert set before building anything.
- Stacks are deleted after each working session.

## Lessons learned

- The first network template had no export names, so the other stacks could not find the VPC; I added exports and checked them with `list-exports`.
- Templates are checked with `cfn-lint` before deploying.
- A deleted secret keeps its name reserved, so I force-delete it during cleanup.

## Next

Take a snapshot, simulate data loss, restore into a new instance and record the time, add a read-only database user and CloudTrail, then add screenshots to `docs/`.
