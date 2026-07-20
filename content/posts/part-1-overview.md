---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 1: Overview & Architecture"
date: 2026-07-16
draft: false
summary: "The kickoff of a series where I turn a classic microservices voting app into a full DevSecOps pipeline: GitOps with ArgoCD, security scanning baked into CI, secrets in Vault, and semantic-versioned releases. Part 1 covers the big picture: the app, the toolchain, and how the pieces fit."
tags: ["DevSecOps", "GitOps", "ArgoCD", "Kubernetes", "CI/CD", "Security", "GitLab", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

Most tutorials stop at "here's how to deploy an app". This series goes further: I took the well-known [Docker voting app](https://github.com/dockersamples/example-voting-app), three small microservices and wrapped it in a complete **DevSecOps pipeline**. Security scanning runs inside CI, deployments happen through GitOps, secrets live in Vault, and every release is versioned automatically.

> **A note on this series:** URLs like `harbor.example.com` are placeholders for internal lab services. Full source code (GitLab CI YAML, Helm charts, Kubernetes templates) will be linked to a public GitHub repo as each part goes live.

## What we're building

The app itself is deliberately simple, the point is the *pipeline* around it. It's a voting app: you vote between two options, and the results update in real time. Three services, each in a different language, talking through a queue and a database:

![Application architecture diagram](/images/devsecops-voting-app/application-diagram.png)

- **vote** — a Python/Flask frontend where users cast a vote. It pushes each vote into Redis.
- **worker** — a C#/.NET 7 background service that pulls votes from Redis and writes them to PostgreSQL.
- **result** — a Node.js/Express app that reads from PostgreSQL and shows live results.

So the flow is: `vote → Redis → worker → PostgreSQL → result`. Redis acts as a buffer so the frontend stays fast, and the worker does the durable write. Everything sits behind an Nginx Ingress Controller inside Kubernetes.

Three languages, a cache, a database, and a queue-like handoff, small enough to reason about, but realistic enough to exercise a real pipeline.

## Why DevSecOps, not just DevOps

The interesting part isn't the app, it's that **security is a first-class stage in the pipeline**, not an afterthought. Every code change flows through a series of automated gates before it can reach production:

![Pipeline and security flow](/images/devsecops-voting-app/pipeline-security-diagram.png)

At a high level, each service's CI pipeline does this:

1. **Build & test** the service.
2. **SAST** — static analysis with SonarQube scans the source for code smells and vulnerabilities.
3. **Container scan** — Trivy scans the built image for known CVEs in OS packages and dependencies.
4. **DAST** — OWASP ZAP probes the running app for runtime vulnerabilities.
5. **Aggregate findings** — all results are pushed to DefectDojo, a single dashboard for vulnerability management.
6. **Push image** to Harbor (the container registry), with tag retention policies.
7. **Deploy via GitOps** — ArgoCD syncs the desired state from Git to the Kubernetes clusters, pulling secrets from Vault at deploy time.

The philosophy: shift security *left* (catch issues in CI, early and cheap) but also keep it *continuous* (scan images and running apps, not just source). If a scan fails a gate, the change doesn't move forward.

## GitOps and two environments

Deployment isn't done by pushing to a server. Instead, the desired state of the cluster lives in Git, and **ArgoCD** continuously reconciles the cluster to match it. There are two environments, each with its own ArgoCD instance and its own RKE2 Kubernetes cluster:

- **Staging** — where changes land first, for validation.
- **Production** — promoted to only after staging passes.

Secrets never live in Git. **Vault** holds them, and the clusters authenticate to Vault to pull what they need at runtime. This keeps credentials out of the repo and out of the pipeline logs.

## Versioned, automated releases

Every merge that reaches `main` produces a **semantically versioned release**, driven by [Conventional Commits](https://www.conventionalcommits.org/). The branch flow and versioning look like this:

![Semantic versioning pipeline](/images/devsecops-voting-app/semantic-pipeline-diagram.png)

Code moves `feature → dev → staging → main`, and the commit type decides the version bump: a `fix:` bumps the patch version, a `feat:` bumps the minor, and a `feat:` with a `BREAKING CHANGE:` bumps the major. No one hand-edits version numbers, the pipeline does it. (Part 5 digs into this properly.)

## The full toolchain

Here's everything the pipeline touches, with the versions used in the lab:

| Layer | Tool | Version |
|---|---|---|
| Operating System | Ubuntu | 22.04.2 |
| SCM | GitLab | 17.9.2 |
| CI/CD Agent | GitLab Runner | 18.4.0 |
| Container Tooling | Docker | 28.4.0 |
| Container Registry | Harbor | 2.10.1 |
| SAST | SonarQube | 25.8 Community |
| Container Scanning | Trivy | 0.67.0 |
| DAST | OWASP ZAP | stable |
| Vulnerability Management | DefectDojo | 2.50.4 |
| Secret Management | Vault | 1.20.4 |
| GitOps (Production) | ArgoCD | 3.0.12 |
| GitOps (Staging) | ArgoCD | 3.1.0 |
| Orchestration (Prod) | RKE2 | v1.32.7+rke2r1 |
| Orchestration (Staging) | RKE2 | v1.32.6+rke2r1 |
| Kubernetes Management | Rancher | 2.11.3 |
| Database | PostgreSQL | 15.14 |
| Cache | Redis | 8.2.1 |
| Storage | NFS StorageClass | — |

It's a lot, but each tool has one clear job. Over the series, they'll show up one at a time, in context.

## How the repositories are organized

Before any pipeline exists, the project needs structure. Everything lives under a GitLab group, `voting-app `, split into repositories by responsibility:

![GitLab group structure](/images/devsecops-voting-app/gitlab-group-structure.png)

- **application** (a subgroup) — holds the three service repos: `worker`, `vote`, and `result`.
- **security** — a shared repo holding the reusable security-scanning templates that every service pipeline includes.
- **deployment** — the Helm charts and Kubernetes manifests ArgoCD watches.

Separating `security` and `deployment` from the app code is deliberate: the security template is written once and reused across all three services, and the deployment definitions are what GitOps reconciles, so they get their own home, independent of any single service.

### Quick start: creating the structure

If you're following along, the foundation is just a nested group plus a few blank projects. In GitLab:

1. Create the top-level subgroup, under your parent group, **New subgroup** → name it `voting-app`.
2. Inside it, create an **application** subgroup the same way (**New subgroup**).
3. Under `voting-app/application`, create three blank projects: `worker`, `vote`, `result` (**New project → Create blank project**).
4. Back under `voting-app`, create two more blank projects at the top level: `security` and `deployment`.

That's the skeleton. Nothing runs yet, but every later part hangs off this layout.

## What's next

That's the map. In **Part 2**, we start wiring the foundation: GitLab group variables, registering a GitLab Runner, and standing up the supporting platforms, SonarQube, DefectDojo, Harbor, ArgoCD, and Vault, that the pipelines depend on.

---

**Series navigation:**
- Part 1: Overview & Architecture *(you are here)*
- Part 2: [Foundation & Toolchain Setup](/page/posts/part-2-foundation/)
- Part 3: The Security Pipeline (soon)
- Part 4: GitOps Deployment (soon)
- Part 5: Branching & Releases (soon)
- Part 6: Day-2 Operations (soon)
