---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 4: GitOps Deployment"
date: 2026-07-24
draft: false
summary: "How a one-line change to a file in Git becomes a running deployment. Part 4 covers the Helm chart that packages all five services, ArgoCD reconciling two environments, and Vault injecting secrets at runtime so no credential ever touches the repo."
tags: ["DevSecOps", "GitOps", "ArgoCD", "Helm", "Vault", "Kubernetes", "RKE2", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

At the end of [Part 3](/posts/part-3-security/), the pipeline scanned everything, recorded the findings, and then did something that looked almost anticlimactic: the `update-tag-stg` job cloned a *different* repo and changed one line in a YAML file.

That one line is the whole deployment. This part explains why.

## The GitOps idea

In a traditional pipeline, the CI job reaches into the cluster and runs `kubectl apply` or `helm upgrade`. The pipeline needs cluster credentials, and the cluster's real state lives wherever the last job left it.

GitOps flips that around. **Git is the single source of truth for what should be running**, and an agent inside the cluster continuously pulls from Git and makes reality match. The pipeline never touches the cluster, it just commits a change to a repo. [ArgoCD](https://argo-cd.readthedocs.io/) is that agent.

So the deployment flow has a clean split:

- The **service pipeline** (Part 3) builds an image, pushes it to Harbor, and updates an image tag in the `deployment` repo.
- **ArgoCD** notices the `deployment` repo changed, and syncs the new state into the cluster.

The pipeline writes intent; ArgoCD makes it real. Neither knows much about the other.

## The deployment repo

Everything ArgoCD watches lives in the `deployment` repo, structured as a Helm chart with per-environment values:

```
deployment/
├── Chart.yaml
├── environments/
│   └── values-<env>.yaml     # the only file the pipeline edits
└── templates/
    ├── db.yaml
    ├── redis.yaml
    ├── vote.yaml
    ├── result.yaml
    ├── worker.yaml
    ├── ingress.yaml
    └── vault-secret.yaml
```

Staging and production are **separate branches** of this repo (`staging` and `production`), each with its own `values-<env>.yaml`. The templates are identical across environments; only the values differ. That's the core Helm idea: one set of templates, many environments and it's what keeps staging and production from drifting apart structurally.

### The values file is the interface

Everything environment-specific lives in `values-<env>.yaml`, and it's worth seeing what changes between the two:

| | Staging | Production |
|---|---|---|
| Ingress hosts | `stg-vote.ap.com` | `vote.ap.com` |
| Image tags | `stg-<sha>` | `v2.0.2-release` |
| Vault auth mount | `vso-auth-stg` | `vso-auth` |
| Vault role | `vso-role-voting-app-stg` | `vso-role-voting-app-prod` |
| StorageClass | `nfs-stg-adel` | `nfs-adel` |

Everything else like replica counts, resource limits, service ports, container images for Postgres and Redis is the same. And crucially, the image tag lines are the only ones the pipeline ever rewrites:

```yaml
image:
  resultRepository: harbor.example.com/voting-app-adel/result
  resultTag: v1.0.2-release      # ← the pipeline sed-replaces this line
  voteRepository: harbor.example.com/voting-app-adel/vote
  voteTag: v2.0.1-release
  workerRepository: harbor.example.com/voting-app-adel/worker
  workerTag: v1.1.3-release
```

Notice the three services carry **independent version numbers** (`v1.0.2`, `v2.0.1`, `v1.1.3`). Each service releases on its own cadence, a fix to `vote` doesn't force a version bump on `worker`. Part 5 covers how those numbers get assigned.

> 📁 [The full deployment chart →](https://github.com/adelianurlinap/voting-app-devsecops/tree/main/deployment)

## How the pipeline updates the tag

Back to that anticlimactic job. Here's what `deploy-stag` actually does:

```yaml
deploy-stag:
  stage: update-tag-stg
  image:
    name: alpine/git:latest
  needs:
    - build-image
  script:
    - git clone --branch staging $DEPLOYMENT_REPOSITORY .manifest
    - cd .manifest/environments
    - result_tag=$(awk '/resultTag:/ {print $2}' values-stag.yaml)
    - sed -i "s/resultTag: $result_tag/resultTag: $STG_VERSION/g" values-stag.yaml
    - git commit -am "Update Staging Result Image Version to $STG_VERSION" && git push origin staging
```

It clones the `staging` branch, reads the current tag, `sed`-replaces it with the new one, and pushes the commit. That's it. The production version (`deploy-prod`) is identical except it targets the `production` branch and uses the release tag instead of `stg-<sha>`.

This is the seam between the two halves of the system: the pipeline's last act is a `git push` to the deployment repo, and ArgoCD takes it from there.

## ArgoCD reconciles it

ArgoCD has an *Application* pointing at the `deployment` repo one per environment. Each Application says, in effect: *"watch this branch, render this Helm chart with this values file, and keep this namespace matching the result."*

When the pipeline pushes the tag change, ArgoCD detects the new commit, re-renders the chart, sees that the `result` Deployment now references a new image tag, and rolls it out. If someone manually changes something in the cluster, ArgoCD notices the drift and corrects it back to what Git says. Git always wins.

The **staging** and **production** clusters each run their own ArgoCD instance, connected to the matching branch (from the setup in Part 2). Nothing crosses over: staging's ArgoCD can't touch production, and vice versa.

![ArgoCD application synced and healthy](/images/devsecops-voting-app/argocd-app-synced.png)

## Secrets never live in Git

There's an obvious problem with putting your whole deployment in Git: the database needs a password, and you can't commit a password to a repo, especially a public-ish one.

This is where **Vault** and the **Vault Secrets Operator (VSO)** come in. The secrets sit in Vault (set up in Part 2). The chart ships a `vault-secret.yaml` template that tells VSO how to fetch them and materialise them as native Kubernetes Secrets inside the cluster at runtime, never passing through Git or the pipeline.

The template has two pieces. First, a `VaultAuth` that says how the cluster proves its identity to Vault:

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: {{ .Values.vault.auth.name }}
spec:
  vaultConnectionRef: vault-secret/vault-connection
  method: kubernetes
  mount: {{ .Values.vault.auth.mount }}      # vso-auth (prod) / vso-auth-stg
  kubernetes:
    role: {{ .Values.vault.auth.role }}
    serviceAccount: default
```

Then a `VaultStaticSecret` per secret, which pulls a value from Vault and creates a Kubernetes Secret from it:

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: {{ .Values.vault.staticSecretDB.name }}
spec:
  vaultAuthRef: {{ .Values.vault.auth.name }}
  mount: {{ .Values.vault.staticSecretDB.mount }}   # voting-app
  type: kv-v2
  path: {{ .Values.vault.staticSecretDB.path }}     # vault-db-secret
  refreshAfter: 10s
  destination:
    create: true
    name: {{ .Values.secret.db }}                    # vault-db-secret
```

The `destination.create: true` is the key line, VSO creates a real Kubernetes Secret named `vault-db-secret`. From there, the database Deployment consumes it like any ordinary secret:

```yaml
envFrom:
  - secretRef:
      name: {{ .Values.secret.db }}
```

So the Postgres container gets `POSTGRES_PASSWORD` as an environment variable, but that value's journey was Vault → VSO → Kubernetes Secret → pod. It never appears in the chart, the values file, the pipeline logs, or Git.

`refreshAfter: 10s` means VSO re-checks Vault periodically rotate a secret in Vault, and the cluster picks it up without a redeploy.

> 📁 [`vault-secret.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/deployment/production/templates/vault-secret.yaml)

## The full picture

Putting Parts 3 and 4 together, here's the complete path from a commit to a running production release:

1. Merge to `main` triggers a release tag (Part 5).
2. The tag pipeline builds the image, tags it `*-release`, pushes to Harbor.
3. An email goes to the approver; the pipeline waits at the manual gate.
4. On approval, `deploy-prod` rewrites the image tag in the `production` branch of the deployment repo.
5. Production ArgoCD sees the commit and syncs the chart.
6. As pods start, VSO pulls DB and Redis credentials from Vault and injects them.
7. The new version is live and nobody ran `kubectl` against production.

That last point is the whole reward of GitOps. Every change to production is a Git commit: reviewable, revertable, and auditable. Which, as it happens, is exactly what makes the rollback story in Part 6 so clean.

## What's next

The deployment machinery is complete. But one thing has been hand-waved through all four parts: where do those version numbers come from? Who decides that a change is `v1.0.2` versus `v1.1.0` versus `v2.0.0`?

**Part 5** is about the branching model and semantic versioning: how `feature → dev → staging → main` works, and how Conventional Commits let the pipeline assign version numbers with no human editing a file.

---

📁 **Source code:** [github.com/adelianurlinap/voting-app-devsecops](https://github.com/adelianurlinap/voting-app-devsecops)

**Series navigation:**

1. [Overview & Architecture](/posts/part-1-overview/)
2. [Foundation & Toolchain Setup](/posts/part-2-foundation/)
3. [The Security Pipeline](/posts/part-3-security/)
4. GitOps Deployment *(you are here)*
5. Branching & Releases *(soon)*
6. Day-2 Operations *(soon)*
