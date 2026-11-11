# CI/CD security gate samples

Workflows that accompany [Security Gating CI/CD Pipelines](/articles/security-gating-cicd/).

| File | Platform |
| --- | --- |
| [`github-security-gates.yml`](github-security-gates.yml) | GitHub Actions — CodeQL, dependency review, Trivy, gated OIDC deploy |
| [`azure-devops-security-gates.yml`](azure-devops-security-gates.yml) | Azure DevOps — Microsoft Security DevOps, SARIF fail-closed, environment checks |

Copy into your repo, then tighten paths, languages, and severity thresholds for your stack. Pair Azure deploy with short-lived identity (OIDC / federation), not long-lived client secrets.
