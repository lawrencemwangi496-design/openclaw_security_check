# OpenClaw Security Monitoring Pipeline

Automated security scanning pipeline that continuously monitors the OpenClaw open-source project for vulnerabilities and automatically creates pull requests for deployment.

## Overview

This repository contains a GitHub Actions workflow that:
- Pulls the latest OpenClaw source code daily
- Runs comprehensive security scans (npm audit, Snyk, Trivy, secret detection)
- Sends Telegram alerts when critical issues are found
- Automatically creates pull requests to self-hosted Gitea for deployment

**Purpose:** Proactive security monitoring of infrastructure dependencies — ensuring the OpenClaw instance running in production is always up-to-date and free of known vulnerabilities.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitHub Actions (openclaw_security_check)          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Schedule: Daily at 3am UTC                                          │
│  Trigger: push, PR, manual dispatch                                 │
│                                                                      │
│  ┌────────────────┐                                                 │
│  │ pull_changes   │ → Clones latest OpenClaw repo                   │
│  │ (job)          │   Uploads as artifact                          │
│  └────────┬───────┘                                                 │
│           │                                                         │
│           ▼                                                         │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│  │ npm_audit      │  │ snyk_scan      │  │ trivy_scan     │        │
│  │ (job)          │  │ (job)          │  │ (job)          │        │
│  └────────┬───────┘  └────────┬───────┘  └────────┬───────┘        │
│           │                   │                   │                 │
│           └───────────────────┼───────────────────┘                 │
│                               │                                     │
│                               ▼                                     │
│  ┌──────────────────────────────────────────────┐                   │
│  │ notification (job)                           │                   │
│  │ - Sends Telegram alert on findings           │                   │
│  │ - Creates PR to self-hosted Gitea            │                   │
│  └──────────────────────────────────────────────┘                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  Self-Hosted Gitea            │
                    │  (openclaw-deployment repo)  │
                    │                               │
                    │  PR Created with security     │
                    │  findings for review          │
                    └───────────────────────────────┘
```

## Security Scan Types

| Scan | Tool | What it Finds | When it Runs |
|------|------|---------------|--------------|
| `pull_changes` | git clone | Fetches latest source | Always |
| `npm_audit` | npm audit | Vulnerable npm dependencies | Daily + manual |
| `snyk_scan` | Snyk | Known vulnerabilities in code and deps | Daily + manual |
| `trivy_scan` | Trivy | Filesystem vulnerabilities | Daily + manual |
| `secret_scan` | TruffleHog | Hardcoded credentials | Daily + manual |
| `image_scan` | Trivy | Container image vulnerabilities | Daily + manual |

## Workflow Triggers

### Scheduled (Automatic)
- Daily at 3am UTC — ensures latest OpenClaw code is always monitored

### Event-Driven
- **Push to main** — scans immediately after code updates
- **Pull request** — scans before merging changes

### Manual (Workflow Dispatch)
- Granular control over which scans to run:
  - `pull_changes` — refresh source code
  - `npm_audit` — dependency scan only
  - `snyk_scan` — Snyk security scan only
  - `trivy_scan` — filesystem vulnerability scan
  - `secret_scan` — credential detection
  - `image_scan` — container image scan
  - `notification` — test Telegram alerts
  - `all` — full security suite

## Telegram Notifications

When critical vulnerabilities are detected, the workflow sends alerts to a configured Telegram chat:

- **Severity thresholds:** HIGH and CRITICAL only (reduces noise)
- **Content:** Summary of findings with links to CVEs
- **Frequency:** Only when new issues are found

## Automated PR Creation

If security scans detect vulnerabilities:
1. The workflow creates a pull request to the self-hosted Gitea deployment repository
2. PR includes security findings and recommended fixes
3. Team can review, approve, and deploy updates

This creates a **closed-loop security system**:
```
Scan → Detect → Notify → Create PR → Review → Deploy → Secure
```

## Security Features

### Implemented
- **Multiple tool coverage** — no single point of failure
- **Scheduled scanning** — continuous monitoring
- **Granular manual controls** — targeted investigations
- **Artifact retention** — scan logs stored for 90 days
- **Telegram alerts** — immediate notification of critical issues
- **Auto-PR creation** — reduces time to remediation

### Secrets Required
Configure these in GitHub repository secrets:

| Secret | Purpose |
|--------|---------|
| `TELEGRAM_BOT_TOKEN` | Bot token for Telegram notifications |
| `TELEGRAM_CHAT_ID` | Chat/group ID to send alerts to |
| `GITEA_API_KEY` | API token for self-hosted Gitea |
| `GITEA_INSTANCE_URL` | URL of self-hosted Gitea (e.g., https://gitea.devsecorpion.sbs) |
| `GITEA_REPO_OWNER` | Owner of deployment repo |
| `GITEA_REPO_NAME` | Name of deployment repo (e.g., openclaw-deployment) |

## Outputs

### Artifacts (stored 90 days)
- `openclaw-source/` — cloned source code
- `npm-audit-logs/` — npm audit results
- `snyk-scan-logs/` — Snyk scan output
- `trivy-scan-logs/` — Trivy vulnerability reports
- `secret-scan-logs/` — TruffleHog findings

### Notifications
- Telegram messages for HIGH/CRITICAL vulnerabilities
- Pull requests to self-hosted Gitea for deployment updates

## Example Notification

```
🔴 Security Alert: OpenClaw Scan

HIGH severity vulnerabilities found:
- CVE-2026-32767 in libexpat (CRITICAL)
- CVE-2026-22184 in zlib (HIGH)

Full report: https://github.com/lawrencemwangi496-design/openclaw_security_check/actions/runs/123456

PR created: https://gitea.devsecorpion.sbs/user/openclaw-deployment/pulls/1
```

## Lessons Learned

### What Worked
- **Multi-tool approach** catches different classes of issues
- **Scheduled scans** ensure continuous coverage without manual intervention
- **Telegram alerts** provide immediate awareness

### What Could Improve
- **False positives** — some tools report issues that aren't exploitable
- **PR automation** — needs review before merging (good safety check)
- **Log storage** — artifacts can grow large; consider periodic cleanup

## Future Improvements

- [ ] Add vulnerability severity scoring (CVSS) to notifications
- [ ] Implement auto-merge for low-risk dependency updates
- [ ] Add Slack/Teams notifications as backup
- [ ] Create dashboard for vulnerability trends over time
- [ ] Integrate with OpenClaw's official security advisory feed

## Related Repositories

- [gitea_pipeline](https://github.com/lawrencemwangi496-design/gitea_pipeline) — Infrastructure for self-hosted Git service
- [github_action_mastery](https://github.com/lawrencemwangi496-design/github_action_mastery) — GitHub Actions learning exercises
- [Note_taking](https://github.com/lawrencemwangi496-design/Note_taking) — DevSecOps learning journey documentation

---

*Built as part of DevSecOps learning journey — continuously monitoring upstream projects to maintain a secure infrastructure.*
