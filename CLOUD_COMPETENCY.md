# Cloud competency mapping

This document supports a junior cloud and IT job search. The main case study is in the [README](README.md).

Everything I practised building and operating LinguaLink maps onto core cloud engineering disciplines. This table is the bridge.

| What I did on LinguaLink | Cloud discipline | AWS equivalent |
| --- | --- | --- |
| Row Level Security, column-level grants, role-gated API routes | Least-privilege access control | IAM policies, resource policies |
| Sentry with proof-of-receipt verification (found and fixed a silently dead client DSN) | Observability and alerting | CloudWatch, X-Ray |
| GitHub Actions CI gating every deploy behind 658 automated tests | CI/CD pipelines | CodePipeline, CodeBuild |
| Scheduled backups plus a documented restore path | Backup and disaster recovery | AWS Backup, RDS snapshots |
| Cron jobs for reminders and calendar sync, with failure monitoring | Scheduled and event-driven workloads | EventBridge, Lambda |
| Idempotent atomic operations on the booking money-path | Reliable distributed operations | SQS idempotency patterns, DynamoDB conditional writes |
| CSRF origin gate, rate limiting, session revocation, audit cycle | Defence in depth | WAF, security groups, GuardDuty mindset |
| Secrets kept in environment config, never in code, secret scanning enabled | Secrets management | Secrets Manager, Parameter Store |

The concepts transferred here are platform-independent: the same least-privilege thinking, the same "monitoring needs monitoring" lesson, the same recovery-path discipline.

## Certifications

- CompTIA A+
- CompTIA Network+
- AWS Solutions Architect Associate (SAA-C03), currently studying
