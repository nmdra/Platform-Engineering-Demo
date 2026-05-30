---
tags: [index, map-of-content]
created: 2026-05-29
---

# 🗺️ FinEdge Platform Engineering Wiki

> **Your learning knowledge base.** This wiki uses Obsidian-style `[[wikilinks]]` to connect concepts.
> Open this folder in [Obsidian](https://obsidian.md) for the best experience (graph view, backlinks, search).

---

## How to Use This Wiki

- **Browse by topic** using the sections below
- **Follow `[[wikilinks]]`** to explore related concepts
- **Add new pages** as you learn — every issue, concept, or insight deserves a page
- **Use the journal** (`journal/`) for daily learning notes
- **Update troubleshooting** (`troubleshooting/`) when you solve a problem

---

## 🏗️ Core Concepts

### Platform Engineering
- [[Platform Engineering]] — The discipline of building IDPs
- [[Internal Developer Platform]] — The product you're building
- [[Infrastructure as Code]] — Declarative infrastructure management

### Crossplane Deep Dive
- [[Crossplane]] — The control plane framework (overview)
- [[Composite Resource Definitions]] — XRDs: the API schema teams use
- [[Compositions]] — The platform logic that maps claims to resources
- [[Claims]] — What teams submit (namespaced, simple YAML)
- [[Managed Resources]] — 1:1 mapping to cloud resources
- [[Providers]] — AWS provider family and configuration
- [[Composition Functions]] — Pipeline functions (P&T, auto-ready)
- [[Deletion Policy]] — Delete vs Orphan and when to use each

### Kubernetes Fundamentals
- [[Custom Resource Definitions]] — Extending the K8s API
- [[Reconciliation Loop]] — The controller pattern (desired → actual)
- [[Kubernetes RBAC]] — Roles, bindings, service accounts
- [[Namespace Isolation]] — Multi-tenancy via namespaces
- [[Network Policies]] — Ingress/egress control
- [[Pod Security Admission]] — PSA profiles and enforcement

### GitOps & Operations
- [[GitOps]] — Git as the single source of truth
- [[Drift Detection]] — Self-healing infrastructure
- [[Policy as Code]] — Admission control and guardrails

---

## 🔧 Tools

- [[LocalStack]] — Local AWS emulation
- [[ArgoCD]] — GitOps continuous delivery
- [[Kyverno]] — Policy engine for Kubernetes
- [[kind]] — Local Kubernetes clusters
- [[Helm]] — Package manager for K8s
- [[Prometheus and Grafana]] — Monitoring and dashboards

---

## 🧩 Patterns

- [[App of Apps Pattern]] — ArgoCD multi-application management
- [[Composition Pattern]] — Platform abstraction design
- [[Golden Path]] — Opinionated happy paths for developers

---

## 🔥 Troubleshooting

- [[Troubleshooting Index]] — Common issues and solutions
- _Add new pages here as you encounter and solve problems_

---

## 📓 Journal

Daily learning notes go in `journal/`. Name them `YYYY-MM-DD.md`.

Example: `journal/2026-05-29.md`

---

## 📖 Reference

- [[Glossary]] — Key terms and definitions

---

## 🔗 Graph Tips (Obsidian)

Open this wiki in Obsidian and use:
- **Graph View** (`Ctrl/Cmd + G`) — visualize all connections
- **Backlinks** — see which pages link TO the current page
- **Search** (`Ctrl/Cmd + Shift + F`) — full-text search across all pages
- **Tags** — filter pages by tags in YAML frontmatter
