# LumaCore

[![gh-pages](https://github.com/edwardhallam/lumacore/actions/workflows/gh-pages.yml/badge.svg)](https://github.com/edwardhallam/lumacore/actions/workflows/gh-pages.yml)
[![link-check](https://github.com/edwardhallam/lumacore/actions/workflows/link-check.yml/badge.svg)](https://github.com/edwardhallam/lumacore/actions/workflows/link-check.yml)
[![Phase](https://img.shields.io/badge/phase-Phase%201%20%E2%80%94%20Docs%20First-blue)](https://edwardhallam.github.io/lumacore/roadmap/)
![Last Updated](https://img.shields.io/github/last-commit/edwardhallam/lumacore/main?label=last%20updated)

LumaCore is an Infrastructure-as-Code blueprint for building a sophisticated personal homelab that applies modern GitOps CI/CD, security-first development, and observability best practices. Although designed for a solo Owner/Operator, LumaCore is publicly documented to demonstrate technical capability and reproducibility for portfolio reviewers.

Purpose
- Share a reproducible reference for standing up a small, secure Kubernetes platform.
- Document decisions, trade-offs, and guardrails used throughout the stack (GitOps, security baselines, observability, exposure model).
- Provide a public, sanitized entry point; implementation details and private infra live elsewhere.

Start here
- Roadmap: https://edwardhallam.github.io/lumacore/roadmap/
- PRD (Product Requirements Document): https://edwardhallam.github.io/lumacore/PRD-LumaCore/

Core Features

- GitOps workflow:
  - Single repo main branch for infra; overlays/staging auto-sync; overlays/prod manual reconcile
  - Image automation: staging auto-bumps semver patch lines; prod promoted via PR; pin by digest; no :latest
  - Promotion by PR bump in overlays/prod; prod is reconciled manually via Weave UI
- CI enforcement gates:
  - Required checks for overlays/staging and overlays/prod: secret-scan (Gitleaks), kustomize build + kubeconform, OPA policy checks (conftest or policy test step), kubectl --server-dry-run
  - Optional checks: kube-linter, helm lint
  - Additional security quality gates:
    - SAST (Semgrep/CodeQL) for app repos (language-dependent)
    - SCA/dependency vulnerability scan (OSV-Scanner/Trivy) across app and infra deps
    - IaC scanning (Checkov/tfsec/KICS) for Terraform and Kubernetes manifests
    - Container image scan (Trivy) and license scan
    - SBOM generation (Syft) and image signing (Cosign), with verification policy at admission
  - Staging: required checks, allow auto-merge on green; no required review (velocity)
  - Prod: same required checks + manual review (CODEOWNERS) and all conversations resolved; protect branch; no force pushes
- Security:
  - 1Password Secrets Operator + Connect: in-cluster secret retrieval; Git stores references only
  - Gatekeeper: enforce in prod, audit in staging; policies for prohibited patterns (e.g., base64 secrets inline, :latest tags, privileged pods) and workload hardening (require runAsNonRoot, drop ALL capabilities with minimal allowlist, readOnlyRootFilesystem, seccompProfile RuntimeDefault, disallow hostPath/hostNetwork/privileged; enforce Pod Security Admission Restricted)
  - NetworkPolicies: default-deny in app namespaces with explicit allows (DNS, API server, Prometheus, 1Password Connect, GitLab Registry, Cloudflare Tunnel/Access, Tailscale, NTP, and ingress via Traefik only)
  - RBAC: hardened least-privilege (namespace-scoped Flux Kustomizations, GitLab Runner constrained roles, operators limited to their namespaces with CRD-scope as needed)
- Observability:
  - kube-prometheus-stack via HelmRelease (Prometheus Operator, Alertmanager, Grafana, node/kube exporters)
  - Loki + Promtail for cluster logs; dashboards integrated in Grafana
  - Alertmanager routes to Discord webhook; recommended alert baselines enabled
- Ingress/TLS:
  - Cloudflare Tunnel terminates TLS at the edge; Cloudflare Access policies per app for SSO/MFA
  - Traefik provides in-cluster routing and middlewares
  - Optional internal TLS (cert-manager) for mTLS to Traefik or origin certs if needed
- Documentation:
  - MkDocs Material site hosted via GitHub Pages (or equivalent); includes reproducibility guide and architecture docs
  - Hybrid exposure: temporary, Access-protected demo links (72h) for specific reviewers; otherwise docs-only public
- Public mirror pipeline: On merges to main in GitLab, a CI job mirrors the infra repo to GitHub (read-only). Guardrails: Gitleaks block, .gitallowed with expirations, and path filters. Flux remains pointed at the private GitLab repo (approved).
- Lifecycle Automation & Maintenance:
  - Renovate-driven PRs (helm/docker/deps) with CI gates and auto-merge of patch updates to staging on green
  - Flux Image Automation: staging auto patch; prod PR and manual reconcile; digest pinning
  - Version drift exporter + update-latency SLO alerts (Grafana + Discord)
  - Owner/Operator Maintenance Handbook with runbooks for updates, rotations, and maintenance windows
