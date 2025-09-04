# LumaCore — Product Requirements Document (PRD)

Last updated: 2025-09-03

## 1) Project Overview

LumaCore is an Infrastructure-as-Code blueprint for building a sophisticated personal homelab that applies modern GitOps CI/CD, security-first development, and observability best practices. Although designed for a solo Owner/Operator, LumaCore is publicly documented to demonstrate technical capability and reproducibility for portfolio reviewers.

Primary personas:
- Owner/Operator: full admin, IaC author, SRE for the homelab
- Portfolio Reviewer: reads public docs; may receive temporary access links for time-bound demos (no standing public endpoints)

Vision statement:
- LumaCore is an Infrastructure as Code blueprint for building a sophisticated homelab ecosystem that applies CI/CD pipelines and modern development best practices to personal infrastructure.

Success criteria (first 90 days, Owner/Operator):
- Deployment success rate: ≥95% first-run within 30 days; ≥99% by day 90
- Median deploy time: <15 minutes end-to-end by day 90 (baseline <30 minutes at launch)
- Post-deploy health checks pass rate: ≥95% without manual intervention within 90 days
- MTTR (node/service): <10 minutes via automated playbooks by day 90

Exposure model:
- Hybrid: public docs (MkDocs) are always available; all live dashboards/apps private by default
- Temporary, Access-protected links can be spun up for specific reviews via Cloudflare Access (expire automatically)
- Public GitHub repo: read-only mirror of infra code and docs for reviewers; canonical write path remains private GitLab; mirror happens post-merge with secret-scanning and path-filter guards.


## Key Decisions (Summary)

- SCM and CI:
  - Canonical code host: GitLab self-hosted (Mac Studio, Docker)
  - Read-only mirrors: Private GitHub (full code), Public GitHub (sanitized infra + docs)
- GitOps engine and repo layout:
  - Flux CD + Weave GitOps UI
  - Single main branch; Pattern C (`apps/<app>/base`, `overlays/staging`, `overlays/prod`; `clusters/lumacore/`)
  - Staging auto-sync; Prod manual reconcile via Weave UI after PR
- Security and secrets:
  - 1Password Secrets Operator + Connect (no secrets in Git)
  - OPA Gatekeeper (enforce in prod, audit in staging)
  - Pod Security Admission (Restricted) everywhere (document exceptions)
  - NetworkPolicies default-deny with explicit allows; hardened RBAC
  - Commit and tag signing on protected branches
- Supply chain and quality:
  - Gitleaks (pre-commit + CI), SAST (Semgrep/CodeQL), SCA (OSV/Trivy), IaC (Checkov/tfsec/KICS), Trivy image scan
  - SBOM (Syft) + image/provenance signing (Cosign), verify at admission
- Observability:
  - kube-prometheus-stack (Prometheus Operator, Alertmanager, Grafana)
  - Loki + Promtail; alerts to Discord
- Ingress/exposure:
  - Traefik in-cluster behind Cloudflare Tunnel + Access (SSO/MFA); no inbound ports
- Lifecycle and backups:
  - Renovate for helm/docker/deps; Flux Image Automation (staging patch auto-bump; prod PR; digest pinning)
  - Version drift exporter + update latency SLOs
  - Offsite backups (R2/B2/S3): GitLab data (restic or built-in), Longhorn PV recurring backups, k3s etcd snapshots; quarterly restore drills

## 2) Core Requirements

- GitOps with single main branch; Kustomize overlays for staging (auto-sync) and prod (manual sync)
- CI/CD using GitLab (self-hosted on Mac Studio via Docker) driving checks and promotions
- Flux source of truth: private GitLab (Option A approved); public GitHub mirror is reviewer-only and not used by Flux
- Flux CD + Weave GitOps UI for reconciliation and manual prod releases
- Multi-stage secret scanning using Gitleaks (pre-commit + CI) and admission policies for secret-safety
- Credential management with 1Password Secrets Operator + Connect (no secrets in Git)
- Monitoring and alerting with kube-prometheus-stack; logs with Loki + Promtail; Alertmanager to Discord
- Security hardening with:
  - Default-deny NetworkPolicies in app namespaces with curated egress/ingress allows
  - RBAC least-privilege for Flux, CI runners, and operators
  - Admission policies (OPA Gatekeeper) enforcing in prod, auditing in staging
  - Pod Security Admission: Restricted baseline across app namespaces (documented exemptions only)
  - Commit and tag signing enforcement: require signed commits/tags for protected branches
  - Kubernetes audit logging (recommended baseline below)
- Ingress and remote access with Traefik in-cluster behind Cloudflare Tunnel + Access (SSO, MFA)
- Public documentation with MkDocs Material; reproducibility guide for the Proxmox+Ubuntu+k3s reference environment
- Reproducible reference architecture: Proxmox, 1x k3s server + 2x agents; Longhorn default StorageClass (replica 2)
- Public, sanitized GitHub mirror of infra code and documentation (MkDocs via GitHub Pages) with enforced secret scanning and path filtering before mirroring
- Security/code-quality assurance: SAST (Semgrep/CodeQL), SCA (OSV-Scanner/Trivy), SBOM + image signing (Syft/Cosign), IaC scanning (Checkov/tfsec/KICS), linting (yamllint, kube-linter, shellcheck, markdownlint), license scanning; enforce as CI gates


## Non-Goals (Out of Scope for MVP)

- Multi-tenant user management and RBAC beyond Owner/Operator needs
- Public, persistent live access to dashboards or internal apps (only time-limited Access links for demos)
- Multi-cluster or geo-distributed deployment (single k3s cluster reference)
- Full-service mesh deployment (Istio/Linkerd); Traefik ingress is sufficient for MVP
- Complex progressive delivery (Flagger/Argo Rollouts); optional later for high-risk apps
- Centralized SIEM platform; Loki + alerts suffice for MVP
- Full disaster recovery automation; quarterly restore drills and documented runbooks cover MVP

## 3) Core Features

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


## 4) Core Components

- Git server and CI:
  - GitLab self-hosted on Mac Studio (Docker)
  - GitLab Runner deployed in-cluster (Kubernetes executor) with minimal RBAC
- GitOps engine and UI:
  - Flux (GitRepository + Kustomizations); Weave GitOps UI read-only for safety
- Secret management:
  - 1Password Secrets Operator + 1Password Connect
- Policy, security, and scanning:
  - Gitleaks (pre-commit + CI)
  - OPA Gatekeeper (admission policies)
  - NetworkPolicy set and hardened RBAC
- Ingress and exposure:
  - Traefik ingress in-cluster
  - Cloudflare Tunnel + Cloudflare Access in front of Traefik
- Observability:
  - kube-prometheus-stack (Prometheus, Alertmanager, Grafana)
  - Loki + Promtail for logs
  - Discord for alerts
- Public SCM mirror: GitHub (read-only) + GitHub Pages (docs)
- Storage:
  - Longhorn as the default StorageClass (replica 2)
- Cluster:
  - k3s: 1x server + 2x agents (3 nodes total)


## 5) App/User Flow

Owner/Operator flow:
1) Build or update app code in separate app repo; push to GitLab; CI builds/pushes image to GitLab Registry
2) Flux Image Automation in infra repo:
   - For staging overlays: automatically bumps to latest patch within selected semver minor line (e.g., 1.4.x), pins by digest
   - For prod overlays: bump occurs via PR; after PR merges, prod is manually reconciled (Resume/Reconcile in Weave UI)
3) Flux reconciles:
   - overlays/staging Kustomization (auto-sync with prune + health checks)
   - overlays/prod Kustomization is suspended by default; production apply requires manual action in Weave UI
4) Post-deploy checks and visibility:
   - Health checks via Flux + Prometheus; dashboards in Grafana; logs in Loki
   - Alerts sent to Discord via Alertmanager
5) Secrets resolved by 1Password Operator at runtime; no secrets stored in Git

Portfolio Reviewer flow:
- Default access is documentation-only (MkDocs)
- Review code in the public GitHub mirror (read-only)
- For demos: time-limited Cloudflare Access links to read-only Grafana dashboards or demo apps; Access controls enforce SSO/MFA and auto-expiry


## 6) Tech Stack

- Platform: Proxmox, Ubuntu VMs, k3s, Longhorn
- Git/CI: GitLab (Docker), GitLab Runner (Kubernetes executor)
- GitOps: Flux, Weave GitOps UI
- Packaging: Kustomize overlays; HelmRelease for kube-prometheus-stack and Loki/Promtail
- Ingress/Exposure: Traefik in-cluster, Cloudflare Tunnel + Access
- Secrets: 1Password Secrets Operator + Connect
- Security/Compliance: OPA Gatekeeper, Pod Security Admission (Restricted), Gitleaks, NetworkPolicies, hardened RBAC, (recommended) Kubernetes audit logging, Renovate Bot (deps/Helm/containers)
- Runtime security (optional): Falco ruleset for syscall anomaly detection in cluster workloads
- Observability: Prometheus Operator, Grafana, Alertmanager, Loki + Promtail, Discord notifications
- Documentation: MkDocs Material, GitHub Pages (or equivalent)
- Public mirror: GitHub (read-only) + minimal GitHub Actions for docs build
- Security/Quality toolchain: Semgrep/CodeQL (SAST), OSV-Scanner/Trivy (SCA), Syft/Cosign (SBOM/signing), Checkov/tfsec/KICS (IaC), ShellCheck/shfmt, markdownlint, pre-commit


## 7) Final Architectural Workflow

1) Developer pushes code to an app repo (GitLab); CI builds and pushes container image to GitLab Registry.
2) Infra repo:
   - Flux ImageRepository scans the registry; ImagePolicy selects latest patch version within minor line for staging; ImageUpdateAutomation commits the updated tag (pin by digest) to overlays/staging.
   - Overlays/prod updates occur via PR after review.
3) Flux source and reconciliation:
   - Single Flux GitRepository (main branch) references the infra repo
   - Two Flux Kustomizations: overlays/staging (auto-sync with interval e.g., 1m; prune and healthChecks enabled) and overlays/prod (spec.suspend: true)
   - Release to prod is a deliberate action in Weave GitOps UI (Resume/Reconcile)
4) Admission and policy:
   - Gatekeeper audits in staging; enforces in prod. Policies include disallowing :latest, privileged pods, inline base64 secrets, and other controls
5) Networking and security:
   - Traffic flows: Cloudflare Edge (Access) → Tunnel → Traefik (in-cluster) → service
   - NetworkPolicies default-deny in app namespaces; explicit egress to DNS, API server, 1Password Connect, GitLab Registry, Cloudflare, Tailscale, NTP; ingress only via Traefik
   - RBAC least-privilege for Flux, GitLab Runner, operators
6) Observability and logs:
   - Prometheus/Grafana provide metrics and dashboards; Alertmanager routes to Discord
   - Loki + Promtail ship logs; Kubernetes audit logs recommended to Loki with moderate policy and 7 days retention
7) Public mirror:
   - On main merges in GitLab, a CI job mirrors sanitized code to the public GitHub repo (read-only)
   - Guardrails: rerun Gitleaks, enforce path whitelist/blacklist, and docs build check before push
   - Flux sources from private GitLab (Option A, approved as standard). Alternative (not used): source from public GitHub with GPG-signed commit verification
8) Supply chain security:
   - App repos produce SBOMs (Syft) and sign images + SBOM/provenance (Cosign/Sigstore)
   - Admission: enforce signature verification via Gatekeeper policy template (deny unsigned/untrusted images)
   - CI gates: SAST (Semgrep/CodeQL), SCA (OSV-Scanner/Trivy), IaC scanning (Checkov/tfsec/KICS), container scan (Trivy), license scan

9) Lifecycle & Maintenance
- Renovate proposes PRs for Helm charts, images, and app deps on a defined cadence
- Staging auto-intakes patch updates via Flux Image Automation; soak period 24–72 hours
- Prod promotion via PR and Weave UI manual reconcile after soak
- Version drift exporter compares desired/deployed vs upstream; alert when drift > policy
- Scheduled rotation pipelines for API-capable providers; 1Password item staleness alerts


## 8) Automated Testing Plan

Unit tests (shift-left):
- App repos: language-specific unit tests and linting (e.g., Go/Python/Node as applicable)
- Policy unit tests: OPA Rego policies validated via conftest or OPA test
- YAML linting: yamllint/kube-linter on K8s manifests and Helm values
- Property-based/fuzz testing (where applicable): Hypothesis/fast-check/Go fuzz tests for critical parsers and business logic
- Shell scripts: shellcheck + shfmt
- Docs: markdownlint for Markdown and link checking
- Pre-commit: enforce hooks (gitleaks protect, lint suites) before committing

Integration tests (CI):
- Secret scanning: Gitleaks (block on known patterns/high severity); allowlist via .gitallowed (with ticket and expiry)
- SAST: Semgrep or CodeQL (language-dependent rulesets incl. OWASP Top 10/CWE Top 25)
- SCA/dependency: OSV-Scanner and/or Trivy filesystem scan; license compliance check
- IaC scanning: Checkov/tfsec for Terraform; KICS for K8s manifests/Helm values
- Manifest validation: kustomize build + kubeconform schemas
- Policy pack: conftest or opa eval against admission policy suite for staging/prod expectations
- Container image: Trivy image scan with critical findings block
- Kubernetes dry-run: kubectl --server-dry-run apply -f on generated manifests using in-cluster GitLab Runner SA with limited RBAC
- Supply chain: SBOM generation (Syft); image signing + attestations (Cosign); verify signatures in CI prior to admission
- Mirror pre-flight: rerun Gitleaks on the mirror candidate, verify path filters, and dry-run docs build to catch broken links

End-to-end tests:
- Staging pipeline:
  - Commit to main triggers Flux to auto-bump staging images (within semver range), reconcile overlays/staging
  - Health verification: pod readiness, service endpoints reachability, Flux Kustomization status=Ready
  - Synthetic check: smoke tests or k6 for key endpoints via Traefik (private or via Access)
- Prod promotion:
  - PR to bump overlays/prod to a specific semver tag; on approval and merge, Weave UI manual reconcile of prod
  - Verify same health metrics and dashboards; compare staging vs prod deltas
- Observability validation:
  - Dashboards present and populated (Grafana)
  - Alerts emulate: temporary rule to fire and verify Discord notification
  - Loki queries return expected logs for selected apps and namespaces
  - Docs publish validation: verify GitHub Pages build succeeds and mirror sync status is healthy
  - DAST baseline (optional): OWASP ZAP baseline scan against staging endpoints via time-limited Access token (report-only by default)


## 9) CI/CD Workflow

GitLab CI (infra repo):
- pre-commit: gitleaks protect (developer machine)
- pipeline stages:
  1) lint: yamllint, kube-linter, Helm lint (for HelmReleases), shellcheck, markdownlint
  2) sast_sca: Semgrep/CodeQL (SAST), OSV-Scanner/Trivy fs scan (SCA/license)
  2.5) fuzz (optional): language-appropriate fuzzers (e.g., go-fuzz, Jazzer for JVM, Node.js fuzzing) on changed modules with crash artifacts collected
  3) validate: gitleaks (block on high severity), kustomize build + kubeconform, conftest policy checks, IaC scanning (Checkov/tfsec/KICS)
  4) dry-run: kubectl --server-dry-run using in-cluster GitLab Runner SA (limited RBAC)
  5) build_sign: generate SBOM (Syft) and sign image + SBOM/provenance (Cosign)
  6) image-automation:
     - staging: Flux ImageUpdateAutomation commits semver patch bumps and pins by digest (auto-merge on green)
     - prod: PR is required; same checks run, plus manual approval (CODEOWNERS) and conversations resolved requirement
  7) mirror_to_github:
     - Runs only on main after all checks pass
     - Uses a GitHub PAT sourced via 1Password and injected into the in-cluster GitLab Runner
     - Performs a path whitelist/blacklist check and reruns Gitleaks; builds docs in dry-run; pushes mirror to GitHub if clean
- protected branches: overlays/prod/** requires PR + review; no force pushes; linear history recommended

GitLab CI (app repos):
- build and push image to GitLab Registry
- SAST (Semgrep/CodeQL) and SCA (OSV-Scanner/Trivy) with block-on-critical
- Generate SBOMs (Syft) and sign images + SBOM/provenance (Cosign); push attestations to registry
- optional Trivy scan (additional policies); optional SBOM export artifact
- trigger Flux Receiver webhook or rely on polling to pick up new image tags

Docs site (GitHub):
- Minimal GitHub Actions workflow to build/publish MkDocs from docs/ to GitHub Pages upon pushes to main


## 10) Implementation Plan

Phase 0 — Foundations:
- Stand up GitLab (Docker on Mac Studio) and create infra/app repos
- Proxmox VMs: Ubuntu for k3s (1x server, 2x agents); install Longhorn; confirm StorageClass default
- Create GitHub public repo and enable GitHub Pages
- Add mirror_to_github GitLab CI job with 1Password-injected PAT; implement path filters and a second Gitleaks pass before mirroring

Phase 1 — GitOps Core:
- Bootstrap Flux (flux-system) and Weave GitOps UI (read-only)
- Implement repo layout (Pattern C: `apps/<app>/base`; `overlays/staging`, `overlays/prod`; `clusters/lumacore/` with GitRepository + Kustomizations)
- Configure staging Kustomization auto-sync (interval 1m; prune + health); prod Kustomization suspended
- Configure Flux ImageRepository/ImagePolicy/ImageUpdateAutomation for semver patch auto-bumps in staging; pin by digest

Phase 2 — Security Baseline:
- Deploy 1Password Secrets Operator + Connect; refactor manifests to use secret references
- Add OPA Gatekeeper; enforce in prod, audit in staging; author core policies (deny :latest, privileged pods, inline secrets, image registry allowlist)
- Apply NetworkPolicy default-deny and explicit allows; verify egress and scrapes
- Harden RBAC for Flux, GitLab Runner, operators
- Enforce image signature verification at admission (Gatekeeper policy template validates Cosign signatures/attestations)
- Recommended: enable Kubernetes audit logging with moderate policy; ship to Loki; retain 7 days

Phase 3 — Observability:
- Install kube-prometheus-stack via HelmRelease (Flux)
- Install Loki + Promtail via HelmRelease (Flux)
- Configure Alertmanager to Discord; import baseline dashboards and alert rules

Phase 4 — Ingress and Exposure:
- Deploy Traefik ingress controller
- Configure Cloudflare Tunnel and Access (SSO/MFA) in front of Traefik; zero inbound ports
- Document hybrid exposure and how to create temporary Access-protected demo links

Phase 5 — CI/CD Hardening:
- GitLab Runner in-cluster (Kubernetes executor) with minimal RBAC
- Add CI stages (lint, sast_sca, validate, dry-run, build_sign, image automation) and branch protection rules
- Implement SAST (Semgrep/CodeQL), SCA (OSV-Scanner/Trivy), IaC scanning (Checkov/tfsec/KICS), image scan (Trivy), SBOM/signing (Syft/Cosign)
- Enforce block-on-critical for SAST/SCA/Trivy and signed image requirement
- CODEOWNERS for overlays/prod PRs

Phase 6 — Documentation and Reproducibility:
- MkDocs Material: publish docs site (architecture, reproducibility guide, operations runbooks, demo playbooks)
- Record topology, namespace conventions, SLOs, and alert/routing policies
- Add screenshots and diagrams (network topology, GitOps flow)
- Validate mirror pipeline on main merges and GitHub Pages build status; add status note/badge to README

Phase 7 — DR and Playbooks (stretch for MVP if needed):
- Node/service failure runbooks targeting MTTR <10 minutes
- Backup/restore procedures for Longhorn and critical components

Phase 8 — Lifecycle & Maintenance Automation (new):
- Objectives:
  - Keep apps and platform stacks current with minimal toil, using safe guardrails
  - Provide clear Owner/Operator maintenance docs and cadences for any recurring manual items
- Deliverables:
  - Renovate enabled across infra + app repos (Helm charts, container images, language dependencies)
  - Flux Image Automation confirmed (staging auto-bumps semver patch; prod by PR; digest pinning)
  - Flux dependsOn and CRD-first ordering for operator stacks (Prometheus Operator, Gatekeeper, Loki, etc.)
  - Version-drift exporter/alerts and update-latency SLOs (e.g., patch ≤7d to staging, ≤30d to prod)
  - Staged rollout policy (patch/minor/major) with soak windows and rollback plan templates
  - Owner/Operator Maintenance Handbook published under docs/ with runbooks
- Acceptance Criteria:
  - Weekly Renovate PRs visible; staging updates auto-merge on green; prod promoted via PR and manual reconcile
  - Drift alerts fire when HelmRelease/chart/image is older than policy threshold
  - Update-latency SLOs measured and reported (Grafana panel + alert)
  - Runbooks exist and have been exercised via a dry-run

Phase 9 — Offsite Backups & DR Validation (new):
- Objectives:
  - Ensure offsite, encrypted backups for code, GitLab data, cluster PVs, and control-plane snapshots
  - Keep operational complexity low with one cloud provider and simple tooling
- Deliverables:
  - Git repositories: push mirrors to private GitHub remotes per repo (read-only)
  - GitLab application data: nightly encrypted backups to S3-compatible storage (restic or GitLab built-in); retention 7 daily / 4 weekly / 6 monthly
  - Longhorn PVs: recurring backups to S3 target (daily + weekly); restore smoke-tests
  - k3s: scheduled etcd snapshots on server; rclone CronJob sync to S3 if snapshots are local-only
  - Credentials: all backup/mirror credentials stored in 1Password; no secrets in backups
  - Single provider standardization (Cloudflare R2 / Backblaze B2 / AWS S3)
  - Restore Playbook published and quarterly DR drill completed
- Acceptance Criteria:
  - Successful dry-run restore of GitLab backup artifacts to a test instance
  - Successful Longhorn volume restore smoke-test into a scratch namespace
  - Presence of recent k3s snapshots in S3; ability to validate/restore snapshot in test flow
  - All backup jobs green for 30 days with retention respected; alerts on failure

Acceptance targets (MVP):
- Staging auto-sync functioning with semver patch auto-bumps
- Production manual reconcile via Weave UI using PR-based promotion
- CI gates blocking unsafe changes reliably (gitleaks, kubeconform, policies, dry-run)
- Secrets fully managed via 1Password Operator (no secrets in Git)
- Observability end-to-end working (dashboards, logs, alerts to Discord)
- NetworkPolicy/RBAC baselines active without breaking core flows
- MkDocs site published with reproducibility guide and architecture explained


## 11) Governance & Change Control

- Branching and protection
  - Single main branch; overlays/staging (auto-sync), overlays/prod (manual)
  - Protected branches: require signed commits/tags; PRs mandatory for overlays/prod/**
  - CODEOWNERS for prod; no force-push; linear history recommended
- Promotion and release policy
  - Patch updates: auto to staging on green; batch PR to prod after soak
  - Minor updates: PR to staging; soak 24–72h; PR to prod with rollback plan
  - Major updates: design review + explicit rollback and maintenance window
- Change windows
  - Monthly: routine prod promotions
  - Quarterly: platform upgrades (k3s, GitLab/Runner), Longhorn breaking changes as needed
- Auditability and traceability
  - CI posts change summaries (drift, SBOM/signature verification results) to PRs
  - Gatekeeper audit logs (staging) and enforcement logs (prod) retained; Loki and dashboards
- Documentation currency
  - Renovate PRs to docs for version references (optional)
  - “Owner/Operator Maintenance Handbook” kept in docs/; dead-link checks in CI

## Appendix A — Namespace Conventions

System namespaces:
- flux-system, weave-gitops, monitoring, ingress-traefik, gatekeeper-system, onepassword, onepassword-connect, longhorn-system, gitlab-runner, logging (if separated)

Per-app namespaces:
- app-freshrss, app-librechat, `app-<service>`

Labeling:
- Recommended labels for env, owner, purpose (e.g., lumacore.env=staging|prod)


## Appendix B — NetworkPolicy Baseline

Default-deny ingress and egress for app namespaces (not kube-system), with explicit allows:
- DNS (TCP/UDP 53) to cluster DNS
- kube-apiserver egress (443) for necessary interactions
- Prometheus scrapes (monitoring namespace to targets)
- Egress to 1Password Connect, GitLab Registry, Cloudflare Tunnel/Access endpoints, Tailscale, NTP
- Ingress permitted only via Traefik (ingress-traefik namespace)


## Appendix C — Audit Logging (Recommended Baseline)

- Enable k3s API audit logging with a moderate policy
- Redact large/sensitive request bodies; include user, verb, resource, namespace, responseStatus
- Ship audit logs to Loki via Promtail; retain 7 days
- Alert on high-severity events to Discord


## Appendix D — Risks, Assumptions, Constraints

Risks:
- Cloudflare Tunnel/Access misconfiguration could overexpose services; mitigated by docs-only default and time-bound Access policies
- Admission policies might block valid changes; mitigated by audit in staging and policy test suite
- Longhorn replica 2 in a 3-node cluster tolerates 1 node failure; further HA may be needed

Assumptions:
- Owner controls a domain and Cloudflare account (or will substitute)
- Mac Studio available to host GitLab; adequate resources for CI and registry operations
- 1Password Connect can be deployed in-cluster and has secure connectivity to 1Password

Constraints:
- Solo-operator throughput; choose automation over manual steps when possible
- Keep public documentation free of secrets and private infrastructure details


## Appendix E — Data/Logging/Telemetry Schema (High-Level)

- Metrics: Prometheus (kube-state-metrics, node-exporter); Grafana dashboards per namespace/service
- Logs: Loki labels include namespace, pod, container, app, env=staging|prod
- Alerts: severity labels (info|warning|critical); route to Discord channel(s) with grouping by namespace/app
- Audit: Loki stream labeled audit=true; dashboards and queries for high-privilege actions

## Appendix F — Mirror Guardrails

- Allowed paths in mirror: clusters/, overlays/, apps/, docs/, .gitallowed, LICENSE, README
- Disallowed paths: state files (e.g., terraform *.tfstate*), caches, local inventories, credentials/secrets, internal-only scripts
- CI mirror job:
  - Use GitHub PAT from 1Password; inject into in-cluster GitLab Runner
  - Whitelist/blacklist check and rerun Gitleaks; dry-run docs build to catch issues
  - Push to public GitHub repo only if all checks pass

## Appendix G — Security Controls Matrix (OWASP Top 10 / CWE Top 25)

- Injection/XSS/Deserialization (OWASP A03/A07): Semgrep/CodeQL rulesets in SAST; ZAP baseline (optional) in DAST
- Authentication/Access (OWASP A01/A07): Cloudflare Access SSO/MFA; Gatekeeper policies; RBAC least-privilege; audit logging
- Sensitive Data Exposure (OWASP A02): 1Password Operator (no secrets in Git); gitleaks; TLS via Cloudflare; internal mTLS optional
- Security Misconfiguration (OWASP A05): IaC scans (Checkov/tfsec/KICS), kube-linter, conftest/OPA policy tests
- Vulnerable/Outdated Components (OWASP A06/CWE Top 25): OSV-Scanner/Trivy SCA; container image scan; Renovate optional for deps
- Identification and AuthN failures (OWASP A07): RBAC reviews; Gatekeeper enforcing restricted capabilities
- Software/Data Integrity (OWASP A08): SBOM (Syft), image/signature verification (Cosign + Gatekeeper), mirror guardrails
- Logging/Monitoring (OWASP A09): Prometheus/Grafana dashboards; Loki logs; audit logs with Discord alerts
- SSRF/CSRF/Request Forgery: Semgrep/CodeQL rules; ZAP baseline (optional)
- General CWE Top 25 coverage: Semgrep/CodeQL standard profiles; policy/unit test suite for Rego; CI gates block criticals

## Appendix H — Hardening Recommendations (Security + Code Quality)

- Commit integrity:
  - Require signed commits and tags on protected branches (GitLab push rules/rulesets)
  - Enforce conventional commits and linear history for clearer auditability
- Dependency and image hygiene:
  - Enable Renovate Bot for app/infra repos (languages, Helm charts, container tags)
  - Enforce registry tag immutability and pin images by digest
- Pod/Container security:
  - Enforce Pod Security Admission: Restricted baseline across namespaces
  - Gatekeeper policies to require: runAsNonRoot, disallow privilege escalation, drop ALL caps with minimal allowlist, readOnlyRootFilesystem, seccompProfile RuntimeDefault, no hostPath/hostNetwork, resource limits/requests
- Supply chain:
  - Generate SBOMs (Syft) and sign images + SBOM/provenance (Cosign); store attestations
  - Admission verification: deny unsigned/untrusted images (policy template or Kyverno verifyImages as alternative)
  - Publish to transparency log (Rekor) when feasible
- Runtime security (optional):
  - Deploy Falco; tune rules for k3s/Longhorn/Flux/GitLab Runner namespaces; alert to Discord
- Fuzzing and property-based tests:
  - Add fuzzer harnesses for parsers/serializers and critical business logic (go-fuzz, Jazzer, fast-check)
  - Run fuzzers as optional CI jobs on PRs and nightly for extended time budgets
- DAST (optional but recommended):
  - ZAP baseline against staging endpoints using short-lived Cloudflare Access token; report-only initially
- Cloudflare/WAF posture:
  - Enable WAF, bot mitigation, rate limiting and security headers at Cloudflare Access per-app policies
- Secrets governance:
  - Rotate 1Password Connect tokens periodically; monitor access with 1Password audit trails
  - Enforce gitleaks on docs/mirror paths and block mirror on detections
- Backups and recovery:
  - Encrypt backups at rest and in transit; test restore as part of DR runbooks
- Audit and compliance:
  - Require 2FA/MFA on GitLab/GitHub accounts; restrict PAT scopes and TTL
  - Retain audit logs for 7–30 days; alert on high-risk verbs (create/update ClusterRole/CRD, etc.)

## Appendix I — Owner/Operator Maintenance Handbook (Cadence & Runbooks)

- Cadence & SLOs
  - Nightly: Renovate discovery; create PRs; staging image auto-bumps
  - Weekly: Merge staging Helm/deps after green CI; soak 1–3 days
  - Monthly: Promote to prod via PR; reconcile in Weave; verify health
  - Quarterly: Platform windows (k3s, GitLab/Runner); backup+restore tests
  - SLOs: Patch update latency (staging ≤ 7d; prod ≤ 30d), Secret age warnings (60d) and critical (90d)

- Manual Task Matrix (What/When/How)
  - k3s minor/major upgrades (quarterly): pre-checks, backups, post-smoke checklist
  - Longhorn upgrades (as needed): read release notes, backup+restore test
  - GitLab self-hosted + Runner (monthly/quarterly): backup, update, validate runners
  - Non-API credential/webhook rotation (as needed): create new, update 1Password item, verify Operator sync, trigger reloader
  - 1Password Connect/Operator token rotation (quarterly): scripted pipeline or step-by-step runbook
  - GitHub PAT rotation for mirror (30–90 days): scripted via API or semi-automated with checklist
  - Backups and restore test (monthly/quarterly): verify restores of key data and state

- How-to Runbooks (to be published under docs/)
  - Promote updates from staging to prod
  - Handle Renovate PRs (charts/images/deps)
  - k3s upgrade window
  - Rotate 1Password Connect token
  - Rotate GitHub PAT for mirror
  - Restore from Longhorn backup (smoke)
  - Respond to version drift alert

- Alerts & Dashboards
  - Grafana panels for Update Latency and Version Drift; Discord alerts for drift > policy, secret age > threshold, and failed post-update health

## Appendix J — Offsite Backups & DR Matrix

- Providers (choose one and standardize): Cloudflare R2 / Backblaze B2 / AWS S3
- Encryption: restic native encryption or server-side encryption for GitLab built-in archives
- Credentials: stored in 1Password; short-lived tokens preferred; no secrets committed

Backup targets:
- Git repositories:
  - Method: GitLab push mirrors to private GitHub remotes (read-only); public sanitized mirror as already defined
  - Frequency: on main merges (push-based)
  - Test: clone mirror, verify commit parity with canonical
- GitLab application data (config, repos, registry, DB):
  - Method A: restic nightly backup of GitLab volumes to S3-compatible storage
  - Method B: GitLab built-in backup rake → push to S3 via s3cmd/rclone
  - Retention: 7 daily, 4 weekly, 6 monthly
  - Test: quarterly restore into a disposable test instance
- Longhorn persistent volumes:
  - Method: Longhorn backupTarget to S3; recurring jobs (daily + weekly)
  - Test: quarterly restore of representative volume to a scratch namespace
- k3s control plane:
  - Method: scheduled etcd snapshots on server; rclone CronJob sync snapshots to S3 if not remote
  - Test: validate snapshot integrity; test restore in a throwaway environment

Monitoring & Alerting:
- Export backup job status and recency metrics to Prometheus
- Alerts to Discord on backup failures or recency violations
- Dashboard panel “Backup Freshness” with per-target status

Restore Playbook (linked from Maintenance Handbook):
- GitLab restore steps (built-in or restic), including registry considerations
- Longhorn volume restore flow and data verification checklist
- k3s snapshot restore outline for test environments
