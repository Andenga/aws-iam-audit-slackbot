# AWS IAM Audit SlackBot

A Slack bot that continuously audits AWS accounts for security drift. It flags stale or unused IAM access keys, over-permissioned IAM roles, and unused OIDC trust policies, then posts findings directly to a Slack channel on a schedule.

## Why this project

Setting up secure infrastructure once isn't enough. Permissions drift over time: keys stop rotating, roles accumulate unused access, trust relationships get created and forgotten. Most teams only discover these problems during an incident or an audit, months after the drift happened.

This bot treats IAM hygiene as something to monitor continuously, not check once. It builds directly on [aws-github-actions-oidc](#), where I set up federated authentication for CI/CD, by checking that setup (and the rest of the account) actually stays secure over time.

## What it checks for

- **Stale IAM access keys** — keys older than a configurable threshold (default 90 days, matching the AWS CIS Benchmark recommendation) that haven't been rotated
- **Unused IAM users and roles** — no activity in CloudTrail for a configurable period, flagged as candidates for removal
- **Over-permissioned roles** — roles with `AdministratorAccess`, wildcard (`*:*`) policies, or broad permissions attached to CI/CD or automation roles
- **Unused OIDC trust policies** — OIDC providers or IAM roles configured for federation (e.g. GitHub Actions) that have never actually been assumed, based on `AssumeRoleWithWebIdentity` activity in CloudTrail
- **IAM users without MFA** — console-access users that don't have multi-factor authentication enabled

## Architecture

```
EventBridge (daily schedule)
        |
        v
Scanner Lambda (Python, boto3)
        |
        |--- Queries IAM (users, roles, access keys, policies)
        |--- Queries CloudTrail (activity history, AssumeRoleWithWebIdentity events)
        |
        v
Findings stored in DynamoDB
        |
        | (diffed against previous run to avoid repeat alerts)
        v
Slack Bot Lambda (Bolt SDK)
        |
        v
Daily digest posted to Slack channel
```

**Components:**
- **Scanner Lambda** — runs on a schedule, pulls IAM and CloudTrail data via boto3, evaluates it against each check
- **DynamoDB** — stores findings so the bot can diff "what's new since yesterday" instead of re-alerting on the same issue every day
- **Slack Bot (Bolt SDK)** — posts a daily digest to a channel; supports slash commands for on-demand checks
- **Terraform** — provisions the Lambda functions, EventBridge schedule, DynamoDB table, and the IAM role the bot itself runs under (scoped to read-only IAM/CloudTrail access)

## Repository structure

```
.
├── scanner/
│   ├── checks/
│   │   ├── stale_keys.py
│   │   ├── overpermissioned_roles.py
│   │   ├── unused_oidc_trust.py
│   │   └── mfa_check.py
│   ├── handler.py
│   └── requirements.txt
├── slack_bot/
│   ├── handler.py
│   └── requirements.txt
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── README.md
```

## Setup

### Prerequisites
- An AWS account with CloudTrail enabled
- A Slack workspace where you can create an app and bot token
- Terraform installed locally

### Steps

1. Create a Slack app, enable the Bolt SDK bot token scopes needed (`chat:write`), and note the bot token.
2. Update `terraform/variables.tf` with your Slack token (stored in AWS Secrets Manager, not committed) and the AWS account(s) to audit.
3. Provision the infrastructure:
   ```bash
   cd terraform
   terraform init
   terraform apply
   ```
4. The scanner runs automatically on the configured schedule. Findings post to the Slack channel you specify.

## Key design decisions

- **Read-only by default** — the bot's own IAM role has only read/list/describe permissions on IAM and CloudTrail. It flags issues; it does not auto-remediate, to avoid the bot itself becoming a high-privilege target.
- **Diffing findings, not re-alerting** — findings are stored and compared run-over-run so the daily digest shows only new or changed issues, avoiding alert fatigue.
- **Configurable thresholds** — "stale" and "unused" are defined by configurable time windows rather than hardcoded, since acceptable rotation periods vary by team.
- **Scoped to one account initially, designed to extend to multiple** — the checks are written per-account so the same Lambda can later be pointed at multiple accounts via cross-account roles.

## What I'd add next

- Slash commands for on-demand checks (`/audit key-age <role-arn>`)
- Auto-remediation behind an explicit approval step (e.g. a Slack button to disable a stale key)
- Support for auditing multiple AWS accounts via cross-account IAM roles
- A severity score per finding, so the digest can be sorted by risk rather than just listed

## References

- [AWS: IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [AWS CIS Benchmark: IAM access key rotation](https://docs.aws.amazon.com/securityhub/latest/userguide/iam-controls.html)
- [Slack Bolt for Python](https://slack.dev/bolt-python/concepts)
