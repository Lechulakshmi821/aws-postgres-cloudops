# aws-postgres-cloudops


# AWS PostgreSQL CloudOps

A small project to run PostgreSQL on AWS in a way that is **monitored, backed up and restorable**, with the whole setup written as code and documented.

## Goal
- Build the infrastructure with CloudFormation (no manual clicking)
- Run PostgreSQL on Amazon RDS
- Watch it with CloudWatch metrics and alarms
- Back it up and **prove** the restore works (restore test with notes and timing)
- Keep costs low and document how to clean up

## Architecture (planned)
```
My laptop --SSH (my IP only)--> small test server (public subnet)
                                        |
                        port 5432 (only from the test server)
                                        v
                          PostgreSQL / RDS (private subnets)


## Plan
- [ ] 1. VPC, subnets, security groups (CloudFormation)
- [ ] 2. RDS PostgreSQL instance
- [ ] 3. CloudWatch metrics and alarms (CPU, storage, connections)
- [ ] 4. Backups and snapshots
- [ ] 5. Restore test (steps and result documented)
- [ ] 6. Cost and cleanup notes

## Repository layout
```
cloudformation/   templates for the infrastructure
docs/             notes, diagrams and test results
scripts/          helper scripts (deploy, backup check)
```

## Tools
AWS (VPC, RDS, CloudWatch, SNS), CloudFormation, AWS CLI, PostgreSQL

## Status
Started 24 Oct 2026. Work in progress. Updated step by step.

## What I want to show
How a database is operated in the cloud: building it repeatably, watching it, and recovering it after a failure.

## Notes on cost
Uses the smallest instance size. Resources are deleted after each test session.
