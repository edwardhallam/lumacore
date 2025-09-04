# Architecture (Draft)

Key components:
- GitLab (SCM/CI), GitHub (public docs), Mirror guardrails for public exposure
- Flux + Kustomize overlays (staging auto-sync, prod manual reconcile)
- Security baselines (Gatekeeper in prod, PSA Restricted, RBAC hardening)
- Default-deny NetworkPolicies with explicit allows
- Observability with Prometheus/Grafana/Loki

This page will grow as Phase 5/8/9 deliverables come online. See PRD §4 and §7.
