---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 6: Day-2 Operations"
date: 2026-10-16
draft: true
summary: "The happy path is done, so what happens when production breaks? Part 6 covers the prehotfix branch that ships a fix without dragging half finished work from dev, cherry-picking it back so it doesn't get lost, and two ways to roll back a bad release: ArgoCD's fast but out of sync rollback, and re-running an old tag pipeline so Git stays the source of truth."
tags: ["DevSecOps", "Hotfix", "Cherry-pick", "Rollback", "ArgoCD", "GitOps", "semantic-release", "GitLab", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

[Part 5](/posts/part-5-branching-releases/) ended on the happy path: a change moves `feature → dev → staging → main` and comes out the other side as a tagged release. That flow is fine for features. It is a problem when production has a bug *right now* and `dev` is full of work nobody has finished testing.

If the only way to production goes through `dev`, a one-line fix has to wait for everything else on `dev`, or it ships with it. Neither is acceptable during an incident.

This last part covers what happens after the first deploy: shipping a hotfix without touching `dev`, getting that fix back into `dev` so it doesn't vanish on the next release, and rolling back when a release turns out to be bad.

## The hotfix path

The idea is to branch from **what is actually running in production**, not from `dev` or `main`. Production runs a tag, so the hotfix branch starts from that tag:

```
v1.1.1-release ──► prehotfix/bug ──► staging ──► main ──► v1.1.2-release
                         │                         │
                         ▼                         ▼
          v1.1.2-hotfix-bug.1-release        cherry-pick ──► dev
           (test image, never deployed)
```

Four steps: build a test image from the hotfix branch, promote it through `staging`, release from `main`, then cherry-pick the fix back to `dev`.

### Step 1: branch from the release tag

The `worker` service was running `v1.1.1-release` in production. In GitLab, the new branch `prehotfix/bug` is created from that tag (not from a branch), and the fix is committed with a normal Conventional Commit message:

```
fix(bug): update hotfix
```

The `prehotfix/*` pattern was already in every job's `only:` list in Part 5, so the branch gets the same two jobs `main` does: `verify_commit_message` and `release`.

![Pipeline on prehotfix/bug running verify_commit_message and release](/images/devsecops-voting-app/prehotfix-release-pipeline.png)

### Step 2: a prerelease tag, not a release

This is where the `.releaserc.json` lines skipped in Part 5 come in:

```json
"branches": [
  "main",
  { "name": "hotfix/+([0-9])?(.{+([0-9]),x}).x", "range": "${name.replace(/^hotfix\\//g, '')}", "type": "maintenance" },
  { "name": "prehotfix/*", "prerelease": "${name.replace(/^prehotfix\\//g, \"hotfix-\")}" }
]
```

`prehotfix/*` is a **prerelease** branch. semantic-release still works out the next version from the commits (a `fix:` on top of `1.1.1` gives `1.1.2`), but it adds a prerelease identifier built from the branch name: `prehotfix/bug` becomes `hotfix-bug`. With the `v${version}-release` tag format, the tag is:

```
v1.1.2-hotfix-bug.1-release
```

The `.1` is the prerelease counter. Push a second fix to the same branch and you get `hotfix-bug.2`. The same bot commit and changelog from Part 5 appear too, just with the prerelease version:

![chore(release): 1.1.2-hotfix-bug.1 commit and tag created by semantic-release-bot](/images/devsecops-voting-app/prehotfix-tag-created.png)

(The `hotfix/1.x`-style **maintenance** branch is a different tool: it patches an older major line while `main` has moved on. This walkthrough doesn't use it.)

### Step 3: a test image that can't reach production

A new tag starts a tag pipeline, and `build-image` runs for *any* tag:

```yaml
build-image:
  before_script:
    - export IMAGE_FULL=$HARBOR_REGISTRY/$REPO_NAME:$STG_VERSION
    - if [[ $CI_COMMIT_TAG != '' ]]; then export IMAGE_FULL=$HARBOR_REGISTRY/$REPO_NAME:$CI_COMMIT_TAG; fi;
  rules:
    - if: '$CI_COMMIT_BRANCH == "staging" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - if: '$CI_COMMIT_TAG != null'
      when: always
```

So the hotfix image lands in Harbor under the prerelease tag, where developers can pull it and reproduce the fix locally:

![Harbor worker repository with the v1.1.2-hotfix-bug.1-release image](/images/devsecops-voting-app/harbor-hotfix-image.png)

The production jobs all have one extra condition:

```yaml
  rules:
    - if: '$CI_COMMIT_TAG =~ /.*-release$/ && $CI_COMMIT_TAG !~ /hotfix-/'
```

`send-approval-notif`, `approval`, and `deploy-prod` all use that rule. A hotfix tag ends in `-release` like any other, but `!~ /hotfix-/` keeps it out of the production stages. The prerelease image is for testing only. To reach production, the fix has to go the normal way.

### Step 4: promote through staging and main

When the fix is confirmed locally, open an MR from `prehotfix/bug` into `staging` and merge it. From here on, nothing is special about it: the merge is a push to `staging`, so the full Part 3 pipeline runs. That means SonarQube, a `stg-<sha>` image build, Trivy, `deploy-stag`, and ZAP against the staging deployment.

The fix skipped `dev`, but it didn't skip the security checks.

When staging looks good, merge `staging` into `main`. The `release` job runs semantic-release again, and this time on `main`, the release branch. The same `fix:` commit that made `1.1.2-hotfix-bug.1` on the prerelease branch now makes a real `1.1.2`:

![v1.1.2-release created on main by semantic-release-bot](/images/devsecops-voting-app/hotfix-release-main.png)

`v1.1.2-release` doesn't contain `hotfix-`, so its tag pipeline goes the full production route from Part 4: build, approval email, manual gate, `deploy-prod`, ArgoCD sync.

## Cherry-picking the fix back to dev

The fix is now on `staging` and `main`, but **not on `dev`**. If nothing is done, the next `dev → staging` merge could put the bug back, or cause a conflict nobody remembers the reason for.

Merging `staging` back into `dev` would break the one-way flow and might pull in other things. A cherry-pick moves only the hotfix commit.

In GitLab, open `staging` → **Commits**, choose the merge commit from the hotfix MR, then **Options → Cherry-pick**:

![Cherry-pick option on the merge commit from prehotfix/bug into staging](/images/devsecops-voting-app/cherry-pick-option.png)

Pick `dev` as the target. GitLab creates a `cherry-pick-<sha>` branch and opens an MR into `dev`. That MR goes through the usual `dev` checks, commit message validation and SAST, and gets merged like any feature branch:

![Cherry-pick MR from cherry-pick-90e3db1a merged into dev](/images/devsecops-voting-app/cherry-pick-mr-dev.png)

Now all three long-lived branches have the fix, and `prehotfix/bug` can be deleted.

## Rolling back

A hotfix assumes you know what's wrong. Sometimes you only know that the latest release is bad and you want the previous one back while you find out why.

Both rollback options below are **manual on purpose**. Rolling back production is a call for the team running the service, not something a pipeline should decide alone. The two options trade speed against keeping Git as the source of truth.

### Option A: ArgoCD rollback (fast, but OutOfSync)

ArgoCD keeps a history of every revision it has synced, so it can redeploy a previous one directly. The `production` branch of the deployment repo has a manual job for it:

```yaml
rollback-production-argocd:
  stage: rollback
  image:
    name: argoproj/argocd:latest
    entrypoint: [""]
  before_script:
    - argocd --insecure --grpc-web login $ARGOCD_SERVER_PROD --username $ARGOCD_USERNAME_PROD --password $ARGOCD_PASSWORD_PROD
  script:
    - echo "Rolling back $APP_PROD to previous version..."
    - argocd --grpc-web app rollback $APP_PROD
    - echo "Rollback completed successfully"
  when: manual
  only:
    refs:
      - production
```

With no history ID, `argocd app rollback` goes back to the previous deployed revision. In this example, production `worker` was on `v1.1.0-release`. Running the job is a single click on the deployment pipeline:

![Manual rollback-production-argocd job in the deployment pipeline](/images/devsecops-voting-app/rollback-argocd-job.png)

The job takes seconds. The `worker` pod is replaced and runs `v1.0.0-release` again:

![Rancher showing the worker pod running v1.0.0-release after rollback](/images/devsecops-voting-app/rancher-rollback-pod.png)

There is a cost. ArgoCD changed the cluster, but nothing changed the `production` branch, which still says `workerTag: v1.1.0-release`. Git and the cluster now disagree, and ArgoCD reports exactly that:

![ArgoCD app Healthy but OutOfSync after rollback](/images/devsecops-voting-app/argocd-rollback-outofsync.png)

**Healthy** but **OutOfSync**. That's the honest state: the app is up, but it isn't running what Git describes. Two things follow from that:

- **This only works because auto-sync is off** (the "Auto sync is not enabled" line in the screenshot). ArgoCD refuses to roll back an app with automated sync, and if it didn't, it would just sync `v1.1.0` straight back on top.
- **The rollback lasts until the next sync.** The next `deploy-sync-prod` run applies whatever Git says, and the bad version comes back. Treat it as a temporary stop while a proper fix or a Git-based rollback is prepared.

### Option B: tag pipeline rollback (slower, stays in sync)

The GitOps way to roll back is to make Git say the old version again. The pipeline already does that for every release, so you just run it for an older tag.

In this example, production `result` was on `v1.0.1-release`. In the `result` application repo, go to **Build → Pipelines → New pipeline** and choose the previous tag, `v1.0.0-release`:

![Run new pipeline for tag v1.0.0-release](/images/devsecops-voting-app/tag-rollback-run-pipeline.png)

That's a normal tag pipeline for a `*-release` tag, so it follows the same production path from Part 4: `build-image`, then the approval email and manual gate, then `deploy-prod`:

![Tag pipeline for v1.0.0-release: build-image, approval, deploy-prod](/images/devsecops-voting-app/tag-rollback-pipeline.png)

`deploy-prod` makes the same one-line change in the `production` branch that it always does, only this time the version goes down:

![values-prod.yaml diff changing resultTag from v1.0.1-release to v1.0.0-release](/images/devsecops-voting-app/tag-rollback-values-diff.png)

ArgoCD syncs it, the `result` pod runs `v1.0.0-release`, and **Git and the cluster agree**. No OutOfSync warning, and nothing waiting to undo the rollback on the next sync. The rollback also leaves a record: a commit in the deployment repo, an approval in the pipeline, and a pipeline run linked to the tag.

It is slower. The image is rebuilt and someone has to approve it, so this takes minutes, not seconds. Also, rebuilding from the same source isn't guaranteed to give an identical image if a base image has changed since. For a planned rollback that's fine. For a 3 a.m. outage, Option A first, then Option B to make Git match, is a reasonable order.

### Which one when

| | ArgoCD rollback | Tag pipeline rollback |
|---|---|---|
| Speed | Seconds | Minutes (build + approval) |
| Git matches cluster | No, app is OutOfSync | Yes |
| Survives next sync | No | Yes |
| Approval gate | Access to the deployment pipeline | Production approver |
| Audit trail | ArgoCD history only | Commit + pipeline + approval |
| Use it for | Stopping an incident now | The lasting rollback |

## Wrapping up the series

Six parts in, this is the whole system:

1. **[Overview](/posts/part-1-overview/)**: three services, two clusters, and why the pipeline is built the way it is.
2. **[Foundation](/posts/part-2-foundation/)**: GitLab variables and runner, SonarQube, DefectDojo, Harbor, ArgoCD, and Vault set up and connected.
3. **[Security](/posts/part-3-security/)**: SAST, image scanning, and DAST on the way to staging, with findings collected in one place.
4. **[GitOps](/posts/part-4-gitops-deployment/)**: the pipeline only writes to Git, ArgoCD deploys, and Vault keeps secrets out of both.
5. **[Releases](/posts/part-5-branching-releases/)**: commit messages decide versions, and no one edits a version file or pushes a tag by hand.
6. **Day-2**: hotfixes that skip `dev` but not the security checks, and rollbacks that can be fast or stay in Git.

The same idea runs through all of it: every change to production is a Git commit that went through the same checks. Even a hotfix and a rollback follow it.

---

📁 **Source code:** [github.com/adelianurlinap/voting-app-devsecops](https://github.com/adelianurlinap/voting-app-devsecops)

**Series navigation:**

1. [Overview & Architecture](/posts/part-1-overview/)
2. [Foundation & Toolchain Setup](/posts/part-2-foundation/)
3. [The Security Pipeline](/posts/part-3-security/)
4. [GitOps Deployment](/posts/part-4-gitops-deployment/)
5. [Branching & Releases](/posts/part-5-branching-releases/)
6. Day-2 Operations *(you are here)*
