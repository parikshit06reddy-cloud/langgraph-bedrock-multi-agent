# Security Policy

## Supported Versions

We patch security issues on the latest minor of each line that is currently supported. Older lines receive critical fixes only when feasible.

| Version | Supported          |
| ------- | ------------------ |
| latest `main` | Yes (active development) |
| Tagged `v1.x` | Yes (security fixes) |
| Older releases | No |

## Reporting a Vulnerability

Please do **not** file public GitHub issues for security vulnerabilities.

Report security issues privately via one of:

- GitHub Security Advisories: https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent/security/advisories/new
- Email: open a private advisory above and we will follow up

When reporting, please include:

- A clear description of the issue and the impact
- Steps or proof-of-concept to reproduce
- Affected versions / commit SHA
- Any suggested mitigation

## Response SLAs

| Severity | Acknowledgement | Initial assessment | Patch target |
| -------- | --------------- | ------------------ | ------------ |
| Critical | 24 hours        | 72 hours           | 7 days       |
| High     | 72 hours        | 7 days             | 30 days      |
| Medium   | 7 days          | 14 days            | Next release |
| Low      | 7 days          | 30 days            | Next release |

## Security Tooling

This repository is protected by:

- CodeQL (static analysis) for Python and TypeScript on every PR
- Gitleaks secret scanning on every PR and push
- pip-audit and `npm audit` for dependency vulnerabilities
- Trivy filesystem scan (CRITICAL/HIGH) uploaded to GitHub Security
- Dependabot weekly updates for pip, npm, GitHub Actions, and Docker

GitHub push protection is enabled at the org/repo level to block secrets at push time.

## Operational Hardening

- Aurora PostgreSQL: deletion protection ON in `prod`, KMS-encrypted, access restricted to private subnets
- ALB listeners: HTTPS-only with redirect from HTTP in `prod`; security groups restricted to VPC CIDRs by default
- ECS Fargate: `enableExecuteCommand` is OFF by default in `prod`, ON only in non-prod for break-glass debugging
- Bedrock: invoked over VPC endpoints (PrivateLink) where supported
- Secrets Manager: per-agent IAM scoped reads

## Coordinated Disclosure

We will credit reporters in release notes unless anonymity is requested. Please give us a reasonable window to ship a fix before public disclosure.
