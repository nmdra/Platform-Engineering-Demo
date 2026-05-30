---
tags: [glossary, reference]
created: 2026-05-29
---

# 📖 Glossary

Quick reference for key terms used throughout this wiki.

---

## A

**Admission Controller**
A piece of code that intercepts requests to the Kubernetes API server before an object is persisted. [[Kyverno]] is a validating admission controller. See [[Policy as Code]].

**ArgoCD**
A GitOps continuous delivery tool for Kubernetes. Watches a Git repo and reconciles the cluster state to match. See [[ArgoCD]], [[GitOps]].

**App-of-Apps**
An ArgoCD pattern where one root Application manages child Application resources. See [[App of Apps Pattern]].

## C

**Claim (XRC)**
A namespaced Kubernetes resource that teams submit to request infrastructure. Creates a Composite Resource (XR) behind the scenes. See [[Claims]].

**CEL (Common Expression Language)**
Google's expression language used in Kubernetes validation rules and [[Kyverno]]'s new `ValidatingPolicy` API.

**Composition**
A Crossplane resource that defines HOW to map a claim to managed resources. Contains the platform team's infrastructure logic. See [[Compositions]].

**Composition Function**
A containerized function that runs in a Crossplane composition pipeline. Examples: `function-patch-and-transform`, `function-auto-ready`. See [[Composition Functions]].

**CompositeResourceDefinition (XRD)**
Defines the API schema (the "form") that teams fill out when creating claims. See [[Composite Resource Definitions]].

**Controller**
A Kubernetes component that watches resources and reconciles desired state with actual state. See [[Reconciliation Loop]].

**CRD (Custom Resource Definition)**
A Kubernetes extension mechanism that lets you define new resource types. See [[Custom Resource Definitions]].

**Crossplane**
An open-source Kubernetes add-on that transforms your cluster into a universal control plane for infrastructure. See [[Crossplane]].

## D

**Deletion Policy**
Controls what happens to the external cloud resource when the Kubernetes managed resource is deleted. `Delete` removes it; `Orphan` leaves it. See [[Deletion Policy]].

**Drift**
When the actual state of infrastructure differs from the desired state declared in code/manifests. See [[Drift Detection]].

## F

**Function (Crossplane)**
See Composition Function.

**function-auto-ready**
A composition function that aggregates readiness from all child managed resources. See [[Composition Functions]].

**function-patch-and-transform**
The primary composition function for mapping claim fields to managed resource fields. See [[Composition Functions]].

## G

**GitOps**
An operational framework where Git is the single source of truth for declarative infrastructure and applications. See [[GitOps]].

**Golden Path**
An opinionated, supported path for teams to follow when building and deploying software. See [[Golden Path]], [[Platform Engineering]].

## H

**Helm**
A package manager for Kubernetes. Uses "charts" to template and deploy K8s resources. See [[Helm]].

**host.docker.internal**
Docker's built-in DNS alias that resolves to the host machine's IP from inside a container. Used by kind nodes to reach [[LocalStack]].

## I

**IaC (Infrastructure as Code)**
Managing infrastructure through code rather than manual processes. See [[Infrastructure as Code]].

**IDP (Internal Developer Platform)**
The product a platform team builds to enable developer self-service. See [[Internal Developer Platform]].

## K

**kind (Kubernetes IN Docker)**
A tool for running local Kubernetes clusters using Docker containers as nodes. See [[kind]].

**Kyverno**
A Kubernetes-native policy engine for validating, mutating, and generating resources. See [[Kyverno]], [[Policy as Code]].

## L

**LocalStack**
A fully functional local AWS cloud stack for development and testing. See [[LocalStack]].

**LimitRange**
A Kubernetes resource that sets default CPU/memory limits for containers in a namespace. See [[Namespace Isolation]].

## M

**Managed Resource (MR)**
A Crossplane resource that represents a single external cloud resource (e.g., an S3 Bucket). See [[Managed Resources]].

**MR** — See Managed Resource.

## N

**Namespace**
A Kubernetes mechanism for dividing cluster resources between multiple users/teams. See [[Namespace Isolation]].

**NetworkPolicy**
A Kubernetes resource that controls network traffic to/from pods. See [[Network Policies]].

## P

**Pipeline Mode**
The only supported composition mode in Crossplane v2. Runs a series of composition functions in sequence. See [[Compositions]], [[Composition Functions]].

**Platform Engineering**
The discipline of designing and building self-service toolchains and internal developer platforms. See [[Platform Engineering]].

**Pod Security Admission (PSA)**
A Kubernetes admission controller that enforces Pod Security Standards at the namespace level. See [[Pod Security Admission]].

**Provider**
A Crossplane package that teaches Crossplane how to manage a specific cloud platform's resources. See [[Providers]].

**ProviderConfig**
A Crossplane resource that tells a provider HOW to authenticate with the cloud API. See [[Providers]], [[LocalStack]].

## R

**RBAC (Role-Based Access Control)**
Kubernetes authorization mechanism using Roles, ClusterRoles, and Bindings. See [[Kubernetes RBAC]].

**Reconciliation**
The process of comparing desired state with actual state and making corrections. See [[Reconciliation Loop]].

**ResourceQuota**
A Kubernetes resource that limits aggregate resource consumption per namespace. See [[Namespace Isolation]].

## S

**Self-Heal**
The ability of a system to automatically recover from drift or failures. ArgoCD and Crossplane both self-heal. See [[Drift Detection]], [[Reconciliation Loop]].

## X

**XR (Composite Resource)**
The cluster-scoped resource created when a team submits a claim. Links the claim to managed resources. See [[Claims]], [[Crossplane]].

**XRC** — See Claim.

**XRD** — See CompositeResourceDefinition.
