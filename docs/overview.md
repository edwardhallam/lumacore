# Overview

LumaCore is an Infrastructure-as-Code reference for a personal homelab built with:
- GitOps (Flux) and Weave GitOps UI
- Shift-left security (OPA Gatekeeper, PSA Restricted, NetworkPolicies)
- Observability (kube-prometheus-stack, Loki/Promtail)
- Secrets via 1Password Operator/Connect (no secrets in Git)
- Hybrid exposure (Cloudflare Tunnel + Access)

See the [PRD](PRD-LumaCore.md) for detailed requirements and flows.
