# GitHub repository setup

Recommended repository name:

`capev2-deployment-runbook`

Recommended visibility: **Private**, because this runbook contains infrastructure addressing and operational details.

Using GitHub CLI after authenticating:

```bash
cd CAPEv2-Deployment-Runbook
git init
git add README.md .gitignore
git commit -m "Add validated CAPEv2 deployment runbook"
gh repo create capev2-deployment-runbook --private --source=. --remote=origin --push
```

If you intentionally want a public repository, review and sanitize internal IPs, usernames, hostnames, infrastructure layout, and any future screenshots/logs first.
