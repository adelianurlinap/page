---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 2: Foundation & Toolchain Setup"
date: 2026-07-17
draft: false
summary: "Before a single pipeline runs, the platform has to exist. Part 2 wires the foundation: GitLab CI variables across a group hierarchy, a Docker-executor Runner, and the five supporting services the pipelines depend on: SonarQube, DefectDojo, Harbor, ArgoCD, and Vault."
tags: ["DevSecOps", "GitLab", "GitLab Runner", "SonarQube", "DefectDojo", "Harbor", "ArgoCD", "Vault", "CI/CD", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

In [Part 1](/page/posts/part-1-overview/) I mapped out what we're building: three microservices wrapped in a pipeline where security scanning, GitOps deployment, and automated versioning all happen without anyone clicking a deploy button.

None of that works until the platform underneath it exists. This part is the unglamorous but essential half, the setup that every later pipeline quietly depends on. By the end, nothing is deployed yet, but everything is ready.

Here's the order we'll go in, and why:

1. **GitLab CI variables** — so pipelines have credentials without hardcoding them
2. **GitLab Runner** — so pipelines have somewhere to actually execute
3. **SonarQube** — projects to receive SAST results
4. **DefectDojo** — products to collect all scan findings
5. **Harbor** — a registry to push images to, with a robot account
6. **ArgoCD** — connected to the deployment repo, for both environments
7. **Vault** — secrets, policy, and Kubernetes auth

> All internal lab URLs below are genericized (`gitlab.example.com`, `harbor.example.com`, and so on). Tokens and passwords shown are placeholders, never commit real ones.

## 1. GitLab CI/CD variables

Every credential the pipeline needs (Harbor login, Sonar token, DefectDojo API key, ArgoCD access) lives in GitLab CI/CD variables, not in the repo. The pipeline YAML only ever references them as `$VARIABLE_NAME`.

What makes this clean is GitLab's **inheritance**: variables set on a group flow down to every subgroup and project inside it. So you define each variable at the highest level where it's still true, and never repeat yourself.

That gives four levels, matching the repo structure from Part 1:

| Level | Where it's set | What goes here |
|---|---|---|
| **Group** `voting-app-adel` | Settings → CI/CD → Variables | Shared across everything: Harbor registry URL & credentials, DefectDojo URL & token, SonarQube host URL |
| **Repo** `deployment` | Repo settings | ArgoCD server, username, password for both staging and production |
| **Subgroup** `application` | Subgroup settings | Values common to all three services |
| **Repo** `worker` / `vote` / `result` | Each repo's settings | Per-service values: that service's `SONAR_TOKEN`, DefectDojo product ID, image name |

The rule of thumb: **`SONAR_TOKEN` is per-project** (SonarQube issues one token per project), so it belongs on each service repo. The Harbor credentials are the same for everyone, so they belong on the group.

**To set them:** go to the group (or subgroup, or repo) → **Settings → CI/CD** → expand **Variables** → **Add variable**. Mask anything sensitive so it doesn't appear in job logs, and mark it **Protected** if it should only be available on protected branches.

- Group `voting-app-adel`
![GitLab group CI/CD variables](/images/devsecops-voting-app/gitlab-group-variables.png)

- Repo Deployment
![GitLab deployment CI/CD variables](/images/devsecops-voting-app/gitlab-deployment-variables.png)

- Subgroup `application`
![GitLab group CI/CD variables](/images/devsecops-voting-app/gitlab-application-variables.png)

- Repo `worker/result/vote`
![GitLab group CI/CD variables](/images/devsecops-voting-app/gitlab-repo-variables.png)

## 2. GitLab Runner

The Runner is the machine that actually executes jobs. This setup uses a dedicated VM with the **Docker executor**, so every job runs in a clean container.

**Register it.** In GitLab, go to the `voting-app-adel` group → **Build → Runners → New runner**, fill in the details, and GitLab gives you a registration command. On the runner VM:

```bash
gitlab-runner register --url https://gitlab.example.com --token <YOUR_RUNNER_TOKEN>
```

It'll prompt you interactively. The answers that matter:

```
Enter a name for the runner:        voting-app-runner
Enter an executor:                  docker
Enter the default Docker image:     alpine:latest
```

**Then edit the config.** This is the part that trips people up, the defaults won't work for this pipeline. Open `/etc/gitlab-runner/config.toml`:

```toml
[[runners]]
  name = "voting-app-runner"
  url = "https://gitlab.example.com"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true                # needed for Docker-in-Docker image builds
    disable_cache = false
    volumes = ["/cache"]
    extra_hosts = [
      "harbor.example.com:10.0.0.10",
      "sonarqube.example.com:10.0.0.10",
      "defectdojo.example.com:10.0.0.10",
      "argocd-staging.example.com:10.0.0.20",
      "argocd-prod.example.com:10.0.0.30"
    ]
    network_mode = "host"
```

Three settings deserve explanation:

- **`privileged = true`** — the pipeline builds container images with Docker-in-Docker, which needs elevated privileges. (Worth knowing: this is a real security trade-off. In a hardened setup you'd reach for Kaniko or Buildah instead.)
- **`extra_hosts`** — these are internal lab services with no public DNS. Mapping hostname → IP inside the runner lets jobs reach them. If your services resolve publicly, skip this.
- **`network_mode = "host"`** — lets job containers talk to those internal services directly.

Restart and confirm:

```bash
gitlab-runner restart
gitlab-runner list
```

**One extra install.** Semantic versioning (Part 5) relies on Conventional Commits being validated locally, so install `pre-commit` on the runner:

```bash
apt install python3-pip
apt install pre-commit
```

## 3. SonarQube — one project per service

SAST results need somewhere to land. Each of the three services gets its own SonarQube project, because each is a different language and gets scored independently.

For **each** service, in the SonarQube dashboard:

1. **Projects → Create Project**
2. Give it a project key and name (e.g. `Voting-App-Worker`)
3. Choose **With GitLab CI** as the analysis method
4. Select the language — **.NET** for worker, **Python** for vote, **JS** for result
5. **Generate a token**

![Creating a SonarQube project](/images/devsecops-voting-app/sonarqube-create-project.png)

![Select App Lang SonarQube](/images/devsecops-voting-app/select-app-lang.png)

![Create Token SonarQube](/images/devsecops-voting-app/create-token.png)

Save two things from this step:

- **`SONAR_TOKEN`** → paste into that service's GitLab repo variables (from step 1)
- **`sonar-project.properties`** → commit into the service repo:

```properties
sonar.projectKey=Voting-App-Worker
sonar.qualitygate.wait=true
```

That `sonar.qualitygate.wait=true` line matters more than it looks: it makes the pipeline **block and wait** for SonarQube's quality gate verdict instead of firing and forgetting. That's what turns a scan into an actual gate.

The language choice also determines which scan template a service uses later, .NET needs the `dotnet-sonarscanner` tool, so the worker gets its own template (`sonarqube-sast-worker.yaml`) while vote and result share the generic one.

> 📁 [`sonar-project.properties` for each service →](https://github.com/adelianurlinap/voting-app-devsecops/tree/main/application)

## 4. DefectDojo — one product per service

SonarQube, Trivy, and ZAP each produce findings in their own format and their own UI. DefectDojo is where they get aggregated, so a security reviewer has one place to look instead of three.

**First, create a user** for the pipeline to authenticate as: **Users → New user** (e.g. `app_developer`).

**Then create a product per service.** Go to **Products → Add Product** and fill in:

| Field | Value |
|---|---|
| Product Name | `Voting App Worker` |
| Tags | `voting-app-worker` |
| Product Type | Research and Development |
| SLA Configuration | Default |
| Platform | Web |
| Enable full risk acceptance | ✅ checked |

![Creating a DefectDojo product](/images/devsecops-voting-app/defectdojo-create-product.png)

**Then grant access:** go to **Users → `app_developer` → Products this User can access → Add Products**, select the product, and assign the **Owner** role.

![Creating a DefectDojo owner](/images/devsecops-voting-app/add-owner.png)

Repeat for `Voting App Vote` and `Voting App Result`. Each product gets an ID, save those into the per-service GitLab variables, since the pipeline uses them to upload scan reports to the right place.

## 5. Harbor — registry, robot account, retention

Harbor stores the container images the pipeline builds and ArgoCD later pulls.

**Create the project.** Login to Harbor → **New Project** → name it `voting-app-adel`, keep it **private**, optionally set a storage quota.

![Harbor project](/images/devsecops-voting-app/harbor-project.png)

**Create a robot account.** This is the credential the pipeline (and Kubernetes) uses, never a human account:

1. Select the `voting-app-adel` project → **Robot Accounts** tab → **New Robot Account**
2. Name it, and set the expiration
3. Set permissions — **Pull/Push** for the GitLab pipeline, **Pull only** for Kubernetes
4. **Copy the generated username and password immediately**, Harbor shows the secret exactly once

![Harbor robot account](/images/devsecops-voting-app/harbor-robot-account.png)

On expiry: setting *never* keeps a continuously-running pipeline from breaking unexpectedly, but a non-expiring credential is a standing risk. If you do set an expiry, put a calendar reminder to rotate the value in GitLab and Rancher before it lapses.

**Give Kubernetes the pull credential.** In each cluster (staging and production), create the namespace and an image pull secret:

```bash
kubectl create namespace voting-app-adel

kubectl create secret docker-registry vault-harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username='<ROBOT_ACCOUNT_NAME>' \
  --docker-password='<ROBOT_ACCOUNT_SECRET>' \
  -n voting-app-adel
```

![Kube secret](/images/devsecops-voting-app/harbor-secret.png)

That secret name, `vault-harbor-secret`, is what the Helm chart references as `imagePullSecrets` in Part 4.

**Set a tag retention policy.** Without one, a pipeline that builds on every commit will fill the registry. Go to the project → **Policy → Tag Retention → Add Rule**, matching the tags this pipeline produces, `*-release` (production) and `stg-*` (staging), and set the schedule to **Weekly**.

![Retention harbor](/images/devsecops-voting-app/harbor-retention.png)

## 6. ArgoCD — connect the deployment repo

ArgoCD needs read access to the `deployment` repo so it can watch for changes. There are two ArgoCD instances (staging and production), and both connect to the same repo, just different branches.

**First, create a GitLab Group Access Token (GAT).** This is what ArgoCD authenticates with:

1. Go to the `voting-app-adel` group → **Settings → Access tokens**
2. **Add new token**, select the scopes it needs (write repository is enough)
3. **Save the token** — shown only once

![Gitlab Token](/images/devsecops-voting-app/gat.png)

**Then connect the repo in each ArgoCD instance:**

1. Login to ArgoCD → **Settings → Repositories → + Connect Repo**
2. Fill in the repo URL, and use the **GAT as the password**
3. Confirm the connection status shows **Successful**

![ArgoCD repository connection](/images/devsecops-voting-app/argocd-connect-repo.png)

![ArgoCD repository connection](/images/devsecops-voting-app/argocd-connect-repo-2.png)

Repeat on the production ArgoCD instance. If the connection fails, it's almost always the token scope or a network path the ArgoCD pod can't reach, not the URL.

## 7. Vault — secrets, policy, and cluster auth

The database and Redis credentials never touch Git or the pipeline. They live in Vault, and the clusters pull them at runtime through the **Vault Secrets Operator (VSO)**.

**Prerequisites:** Vault installed, and VSO deployed on both Kubernetes clusters.

### Enable a secret engine and store the secrets

```bash
vault secrets enable -path=voting-app kv-v2

vault kv put voting-app/vault-db-secret \
  POSTGRES_USER='postgres' \
  POSTGRES_PASSWORD='<YOUR_DB_PASSWORD>' \
  POSTGRES_DB='postgres' \
  DB_HOST='db'

vault kv put voting-app/vault-redis-secret \
  REDIS_HOST='redis' \
  REDIS_PORT='6379'
```

![Vault Secret](/images/devsecops-voting-app/vault-secret.png)

![Vault DB Secret](/images/devsecops-voting-app/vault-db-secret.png)

![Vault Redis Secret](/images/devsecops-voting-app/vault-redis-secret.png)

### Write a read-only policy

The clusters should be able to *read* these secrets and nothing else:

```bash
vault policy write ap-vault-policy - <<EOF
path "voting-app/data/vault-db-secret" {
  capabilities = ["read"]
}
path "voting-app/metadata/vault-db-secret" {
  capabilities = ["read"]
}
path "voting-app/data/vault-redis-secret" {
  capabilities = ["read"]
}
path "voting-app/metadata/vault-redis-secret" {
  capabilities = ["read"]
}
EOF
```

Note that both `data/` and `metadata/` paths are needed, that's a KV-v2 quirk, and forgetting `metadata/` is a common cause of confusing permission errors.

### Connect each Kubernetes cluster

Each cluster authenticates to Vault using its own service account tokens. **Production:**

```bash
vault auth enable -path=vso-auth kubernetes

vault write auth/vso-auth/config \
  token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA_CERT"

vault write auth/vso-auth/role/vso-role-voting-app-prod \
  bound_service_account_names=default \
  bound_service_account_namespaces=voting-app-adel \
  policies=ap-vault-policy \
  ttl=24h \
  audience="https://kubernetes.default.svc.cluster.local"
```

**Staging** is the same, on its own auth path so the two clusters stay isolated:

```bash
vault auth enable -path=vso-auth-stg kubernetes

vault write auth/vso-auth-stg/config \
  token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA_CERT"

vault write auth/vso-auth-stg/role/vso-role-voting-app-stg \
  bound_service_account_names=default \
  bound_service_account_namespaces=voting-app-adel \
  policies=ap-vault-policy \
  ttl=24h \
  audience="https://kubernetes.default.svc.cluster.local"
```

The `bound_service_account_*` fields are the security boundary: only the `default` service account, only in the `voting-app-adel` namespace, can assume this role. A pod in any other namespace gets nothing.

## Where we are now

Nothing is deployed, but the foundation is complete:

- ✅ Credentials live in GitLab variables, inherited down the group hierarchy
- ✅ A Docker-executor Runner is registered and can reach every internal service
- ✅ SonarQube has three projects, each with a token and quality gate
- ✅ DefectDojo has three products ready to receive findings
- ✅ Harbor has a private project, a robot account, and a retention policy
- ✅ Both ArgoCD instances can read the deployment repo
- ✅ Vault holds the secrets, with a read-only policy scoped per cluster

## What's next

Everything above exists so that **Part 3** can put it to work. Next we write the security templates themselves, the reusable YAML that runs SonarQube SAST, Trivy image scanning, and OWASP ZAP DAST, then ships every finding into DefectDojo. That's where the pipeline stops being plumbing and starts being a gate.

---

📁 **Source code:** [github.com/adelianurlinap/voting-app-devsecops](https://github.com/adelianurlinap/voting-app-devsecops)

**Series navigation:**
- Part 1: [Overview & Architecture](/page/posts/part-1-overview/)
- Part 2: Foundation & Toolchain Setup *(you are here)*
- Part 3: The Security Pipeline (soon)
- Part 4: GitOps Deployment (soon)
- Part 5: Branching & Releases (soon)
- Part 6: Day-2 Operations (soon)