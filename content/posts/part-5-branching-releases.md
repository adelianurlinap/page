---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 5: Branching & Releases"
date: 2026-08-05
draft: false
summary: "Where do version numbers actually come from? Part 5 covers the feature to dev to staging to main branch flow, Conventional Commits as the versioning signal, and semantic-release turning a merge to main into a tagged, changelogged release with no human editing a version file."
tags: ["DevSecOps", "Semantic Versioning", "Conventional Commits", "semantic-release", "CI/CD", "GitLab", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

[Part 4](/posts/part-4-gitops-deployment/) closed with a question I kept dodging: who decides that a change is `v1.0.2` and not `v1.1.0` or `v2.0.0`? Every values file so far just showed a tag already sitting there, `v1.0.2-release`, `v2.0.1-release`, as if it appeared on its own.

It did, in a sense. Nobody on this project hand-edits a version number. This part is about the two things that make that possible: a branch flow that gives every change a predictable path to production, and [Conventional Commits](https://www.conventionalcommits.org/) feeding a tool called `semantic-release` that reads commit messages and decides the version for you.

## The branch flow

Four branches, each mapped to a stage of the pipeline:

![Semantic versioning pipeline](/images/devsecops-voting-app/semantic-pipeline-diagram.png)

- **`feature/*`**: short lived, one per change, branched off `dev`.
- **`dev`**: where feature branches land first. SAST runs here, and so does commit message validation.
- **`staging`**: builds the image, scans it with Trivy, deploys it, and runs DAST against it.
- **`main`**: the only branch that produces a release tag.

The rule is simple to state and easy to follow: work happens on `feature/*`, gets merged to `dev`, then `dev` to `staging`, then `staging` to `main`. Nothing skips a step. `dev`, `staging`, and `main` are all protected branches in GitLab, and tags matching `v*` are protected too, so nobody can push a tag by hand even if they wanted to.

That last point matters more than it sounds. If tags can be created manually, "the tag is the source of truth" stops being true the moment someone runs `git tag` locally to unblock themselves. Protecting the tag pattern is what keeps the version history honest.

## Conventional Commits: the signal the pipeline reads

None of the automation works unless commit messages follow a format the tooling can parse. The convention here is the same one Angular popularized:

- `fix: ...` → patch release (`1.0.0` → `1.0.1`)
- `feat: ...` → minor release (`1.0.1` → `1.1.0`)
- `feat: ...` with a `BREAKING CHANGE:` footer → major release (`1.1.0` → `2.0.0`)

This isn't a convention enforced by a wiki page nobody reads. It's enforced in CI, on every commit that lands on `dev`, `staging`, `main`, or a hotfix branch:

```yaml
.prepare_node_environment:
  image: node:20-alpine
  stage: prepare
  tags:
    - docker-ap
  cache:
    policy: pull-push
    when: on_success
    paths:
      - node_modules
    key:
      files:
        - package.json
      prefix: node_modules
  before_script:
    - node --version
    - npm --version
    - rm -f package.json
    - |
      cat <<EOF > package.json
      {
        "name": "semantic",
        "version": "1.0.0",
        "devDependencies": {
          "@commitlint/cli": "^19.8.0",
          "@commitlint/config-conventional": "^19.8.0",
          "@semantic-release/changelog": "^6.0.3",
          "@semantic-release/git": "^10.0.1",
          "@semantic-release/gitlab": "^13.2.4",
          "@semantic-release/npm": "^12.0.1",
          "semantic-release": "^24.2.3"
        }
      }
      EOF
    - npm install
    - apk add --no-cache git
    - git fetch --prune origin
    - git remote prune origin
  only:
    - dev
    - staging
    - main
    - '/^prehotfix\//'

verify_commit_message:
  stage: verify
  extends: .prepare_node_environment
  script:
    - mkdir -p commitlint
    - cd commitlint
    - echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > .commitlintrc.js
    - echo "${CI_COMMIT_MESSAGE}" | npx commitlint --config .commitlintrc.js
  only:
    - dev
    - staging
    - main
    - '/^prehotfix\//'
```

Two things worth noticing:

- **`package.json` isn't committed to the repo.** It's generated fresh in every job with a heredoc. `commitlint` and `semantic-release` are dev tooling for the pipeline, not something the service itself depends on, so there's no reason to carry them in the actual project.
- **`verify_commit_message` fails the pipeline immediately if the message doesn't parse.** That's the whole point of putting it in the `verify` stage, before anything gets built. A commit like `updated worker logic` never reaches the release job at all, it's rejected right after merge.

Here's what a compliant commit looks like from the GitLab UI, from an actual `fix:` change:

![Commit message following Conventional Commits format](/images/devsecops-voting-app/commit-fix-patch.png)

## semantic-release and .releaserc.json

`commitlint` only checks the format. The tool that actually decides the version and creates the release is `semantic-release`, configured entirely through `.releaserc.json`:

```json
{
  "branches": [
    "main",
    { "name": "hotfix/+([0-9])?(.{+([0-9]),x}).x", "range": "${name.replace(/^hotfix\\//g, '')}", "type": "maintenance" },
    { "name": "prehotfix/*", "prerelease": "${name.replace(/^prehotfix\\//g, \"hotfix-\")}" }
  ],
  "tagFormat": "v${version}-release",
  "repositoryUrl": "https://gitlab.example.com/voting-app/application/vote.git",
  "plugins": [
    [
      "@semantic-release/commit-analyzer",
      {
        "preset": "angular",
        "releaseRules": [
          { "breaking": true, "release": "major" }
        ]
      }
    ],
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/gitlab",
      {
        "assets": [
          { "path": "index.js", "label": "Module" },
          { "path": "README.md", "label": "Documentation" },
          { "path": "LICENSE", "label": "License" }
        ]
      }
    ],
    [
      "@semantic-release/npm",
      { "npmPublish": false }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["package-semantic.json", "CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version}\n\n${nextRelease.notes}"
      }
    ]
  ]
}
```

The plugin list runs top to bottom, and each one has one job:

1. **`commit-analyzer`** reads every commit since the last release and decides what kind of bump it is. It uses the `angular` preset (`fix:` / `feat:` / others), plus one explicit rule of its own: anything marked `breaking: true` is a major release, full stop.
2. **`release-notes-generator`** turns those commits into human readable notes, grouped by type.
3. **`changelog`** writes those notes into `CHANGELOG.md`.
4. **`gitlab`** publishes a GitLab Release with the generated notes attached, plus whatever build assets you list.
5. **`npm`** bumps the version field, with `npmPublish: false` since nothing here is actually published to a registry, this project just reuses `npm`'s versioning logic.
6. **`git`** commits `CHANGELOG.md` (and the version file) straight back to `main`, with the message `chore(release): <version>`.

The `hotfix/*` and `prehotfix/*` branch entries are there for a workflow this part doesn't cover, patching production directly without waiting for the full `dev → staging → main` cycle. That's Part 6 territory.

### Where the tag name comes from

This line answers the question this whole post opened with:

```json
"tagFormat": "v${version}-release"
```

`semantic-release` computes the version (say `1.0.1`), substitutes it into that template, and tags the commit `v1.0.1-release`. That's the entire mechanism. There's no separate script watching for merges to `main` and deciding to tag something, the tag format is a one-line config value, and the version that fills it in comes from the commit history `commit-analyzer` just read.

## The release job: dry run first, then commit

Not every merge to `main` should produce a release. A `docs:` or `chore:` commit, under the default Angular rules, doesn't bump anything. So the job runs `semantic-release` twice, once to check, once for real:

```yaml
release:
  stage: release
  extends: .prepare_node_environment
  script:
    - apk add git curl
    - git config --global user.name "GitLab CI/CD Bot"
    - git config --global user.email "cicd@example.com"
    - |
      DRY_RUN_OUTPUT=$(npx semantic-release --dry-run)
      echo "$DRY_RUN_OUTPUT"
      if echo "$DRY_RUN_OUTPUT" | grep -qi "The next release version is"; then
        NEW_VERSION=$(echo "$DRY_RUN_OUTPUT" | grep -i 'The next release version is' | sed -E 's/.*The next release version is ([0-9]+\.[0-9]+\.[0-9]+).*/\1/')
        echo "New version: $NEW_VERSION"
        echo "$NEW_VERSION" > new_version.txt
        npx semantic-release
      else
        echo "No new version to release"
        echo "No new version" > new_version.txt
      fi
  artifacts:
    paths:
      - package.json
      - new_version.txt
  only:
    - main
    - '/^prehotfix\//'
  retry: 2
```

The `--dry-run` pass prints what `semantic-release` *would* do without touching anything. The job greps that output for "The next release version is", and only if that string shows up does it run `semantic-release` for real, tagging, releasing, and pushing the changelog commit. If it doesn't show up, the job exits quietly with `No new version to release`. Nothing gets tagged, nothing gets pushed. A `docs:` commit passes through main without triggering a phantom `v1.0.2` nobody asked for.

This is what that job actually looks like running after a merge to `main`, both stages green:

![Pipeline running verify_commit_message then release on main](/images/devsecops-voting-app/release-pipeline-main.png)

## CHANGELOG, automatically

The `@semantic-release/git` plugin from the config above is what closes the loop. After a real release runs, it commits `CHANGELOG.md` straight back into `main` as `chore(release): <version>`, authored by the pipeline itself rather than a person:

![Repository showing the auto-generated chore(release) commit and the resulting tag](/images/devsecops-voting-app/tag-changelog-commit.png)

Two things to notice in that screenshot: the commit is attributed to `semantic-release-bot`, not a developer, and the tag dropdown already shows `v1.0.1-release`, created in the same pipeline run that produced the commit. Nobody wrote `CHANGELOG.md` by hand, and nobody ran `git tag` by hand. Both are a side effect of a `fix:` commit reaching `main`.

## Three examples, one mechanism

The cleanest way to see this working is to watch the same flow produce three different outcomes, depending only on the commit message.

| Commit | Branch flow | Result |
|---|---|---|
| `fix(patch): update patch semantic` | `feature/patch-minor` → `dev` → `staging` → `main` | `v1.0.1-release` |
| `feat(minor): ...` | `feature/update-minor` → `dev` → `staging` → `main` | `v1.1.0-release` |
| `feat(major): update major` + `BREAKING CHANGE: ...` | `feature/update-major` → `dev` → `staging` → `main` | `v2.0.0-release` |

Same branches, same pipeline, same `.releaserc.json`. The only thing that changes between a patch and a major version is the words in the commit message. Here's the major example, since it's the most dramatic one to see in the commit UI:

![Commit message with a BREAKING CHANGE footer](/images/devsecops-voting-app/commit-breaking-change.png)

That single `BREAKING CHANGE:` footer is what `commit-analyzer`'s `{ "breaking": true, "release": "major" }` rule is watching for. Everything downstream, the version bump, the tag, the changelog entry, follows from that one line.

## What's next

Everything up to this point assumes the happy path: a change moves cleanly through `feature → dev → staging → main` and comes out the other side as a tagged release. Production doesn't always get that luxury. Sometimes a bug needs a fix *now*, without waiting for whatever is sitting half finished on `dev`.

**Part 6** covers that: the hotfix and cherry-pick workflow hinted at in `.releaserc.json`'s `prehotfix/*` branch config, along with rolling back a bad release through ArgoCD and through the tag pipeline itself.

---

📁 **Source code:** [github.com/adelianurlinap/voting-app-devsecops](https://github.com/adelianurlinap/voting-app-devsecops)

**Series navigation:**

1. [Overview & Architecture](/posts/part-1-overview/)
2. [Foundation & Toolchain Setup](/posts/part-2-foundation/)
3. [The Security Pipeline](/posts/part-3-security/)
4. [GitOps Deployment](/posts/part-4-gitops-deployment/)
5. Branching & Releases *(you are here)*
6. [Day-2 Operations](/posts/part-6-day-2-operations/)
