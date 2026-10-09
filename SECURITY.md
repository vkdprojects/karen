# Security Policy

KAREN controls hypervisors and customer VMs. Treat bugs in auth, tenant isolation, agent protocol and network filtering as security issues.

## Reporting
Do **not** open a public issue. Use GitHub private vulnerability reporting (Security → Report a vulnerability).

We aim to acknowledge within 72h and ship a fix or mitigation within 30 days for critical issues.

## Supported versions
Pre-1.0: only the latest release.

## In scope
- Tenant escape (customer accessing another customer's VM, IP, console, backup)
- Auth/RBAC bypass, API token leakage
- Agent ↔ control protocol (mTLS, enrollment tokens)
- Anti-spoofing / network filter bypass
- Command injection into libvirt/QEMU/host
