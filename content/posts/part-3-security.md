---
title: "Building a DevSecOps Pipeline for a Microservices Voting App — Part 3: The Security Pipeline"
date: 2026-07-21
draft: false
summary: "The part where the pipeline stops being plumbing and starts being security. Five reusable templates in one shared repo give every service SAST, container scanning, and DAST — with every finding landing in DefectDojo automatically."
tags: ["DevSecOps", "GitLab CI", "SonarQube", "Trivy", "OWASP ZAP", "DefectDojo", "SAST", "DAST", "Security", "Series"]
categories: ["DevSecOps"]
series: ["DevSecOps Voting App"]
---

[Part 2](/page/posts/part-2-foundation/) left us with a platform that does nothing. SonarQube has empty projects, DefectDojo has empty products, Harbor has an empty registry. Everything is connected but idle.

This part wakes it up. We're writing the templates that scan code, scan images, scan the running app, and funnel every finding into one dashboard, and doing it in a way where **three services share one copy** of that logic.

## The idea: write it once, include it everywhere

The naive approach is to copy the scan jobs into each service's `.gitlab-ci.yml`. That works right up until you need to change something, then you're editing three files and hoping you didn't miss one.

Instead, all the scanning logic lives in its own repo (`security`, from Part 1), and each service pulls it in:

```yaml
include:
  - project: 'voting-app/security'
    ref: main
    file:
      - 'sonarqube-sast.yaml'
      - 'defect-dojo.yaml'
      - 'dast.yaml'
      - 'trivy.yaml'
```

That's the whole integration. A service repo doesn't define a single scan job, it just declares which templates it wants, and GitLab merges them into the pipeline.

Five files live in that repo:

| File | Job |
|---|---|
| `sonarqube-sast.yaml` | SAST for Python and JavaScript |
| `sonarqube-sast-worker.yaml` | SAST for .NET (different scanner) |
| `trivy.yaml` | Container image vulnerability scan |
| `dast.yaml` | Runtime scan with OWASP ZAP |
| `defect-dojo.yaml` | Creates the engagement + shared upload logic |

### Not every service gets every scan

Here's where it gets interesting. The three services don't include the same set:

| Service | SAST | Trivy | DAST |
|---|---|---|---|
| **vote** (Python) | `sonarqube-sast.yaml` | ✅ | ✅ |
| **result** (Node) | `sonarqube-sast.yaml` | ✅ | ✅ |
| **worker** (.NET) | `sonarqube-sast-worker.yaml` | ✅ | ❌ |

Two differences, both deliberate:

- **The worker uses a different SAST template.** SonarQube analyses .NET through `dotnet-sonarscanner`, which has to wrap the build itself.
- **The worker gets no DAST.** DAST works by making HTTP requests against a running app. The worker has no HTTP surface at all, it's a background consumer pulling from Redis. Pointing ZAP at it would scan nothing.

That second one is worth sitting with: **security tooling should match what a service actually is.** Applying all scans to everything looks thorough but produces noise.

## SAST — scanning the source

The generic template runs SonarQube's scanner CLI. The most interesting bit isn't the scan itself, it's the branch handling:

```yaml
sonarqube-check:
  stage: sast
  tags:
    - docker-ap
  image:
    name: sonarsource/sonar-scanner-cli:11
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: '${CI_PROJECT_DIR}/.sonar'
    GIT_DEPTH: '0'   # fetch all branches — the analysis needs them
  cache:
    key: '${CI_JOB_NAME}'
    paths:
      - .sonar/cache
  script:
    - |
      if [ "$CI_COMMIT_BRANCH" == "dev" ]; then
        SONAR_BRANCH="dev"
      elif [ "$CI_COMMIT_BRANCH" == "staging" ]; then
        SONAR_BRANCH="staging"
      else
        SONAR_BRANCH="$CI_COMMIT_BRANCH"
      fi
      sonar-scanner
  allow_failure: true
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev" && $CI_COMMIT_TAG == null'
      when: always
    - if: '$CI_COMMIT_BRANCH == "staging" && $CI_COMMIT_TAG == null'
      when: always
    - if: $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^feature\/.+$/ && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "dev" && $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
```

Three things to notice:

- **`GIT_DEPTH: '0'`** — GitLab shallow-clones by default, but SonarQube needs full history to attribute issues to commits and detect new code.
- **The `.sonar/cache` cache** keeps the scanner from re-downloading its analysers every run.
- **The rules run SAST early and often** — on `dev`, on `staging`, and on merge requests from `feature/*` and `hotfix/*`. That's the "shift left" part: you find out on the MR, not after merging.

Note the scan runs on branch pushes but **not** on tags (`$CI_COMMIT_TAG == null`). Source code doesn't change between merging to `main` and tagging a release, so re-scanning would just burn runner minutes.

### The .NET variant

Same job name, same stage, different mechanics, because `dotnet-sonarscanner` has to bracket the build:

```yaml
sonarqube-check:
  stage: sast
  image:
    name: mcr.microsoft.com/dotnet/sdk:7.0
  before_script:
    - dotnet tool install --global dotnet-sonarscanner
    - export PATH="$PATH:$HOME/.dotnet/tools"
  script:
    - |
      dotnet sonarscanner begin \
        /k:"${SONAR_PROJECT_KEY}" \
        /d:sonar.host.url="${SONAR_HOST_URL}" \
        /d:sonar.login="${SONAR_TOKEN}" \
        /d:sonar.branch.name="${SONAR_BRANCH}"
    - dotnet build
    - |
      dotnet sonarscanner end \
        /d:sonar.login="${SONAR_TOKEN}"
```

`begin` → `build` → `end`. The scanner hooks into the compiler to see the code the way the compiler does, which is why it can't just run standalone like the CLI version.

> 📁 [`sonarqube-sast.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/security/sonarqube-sast.yaml) · [`sonarqube-sast-worker.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/security/sonarqube-sast-worker.yaml)

## Container scanning — Trivy

SAST reads your code. But your image also contains a base OS, system packages, and dependency trees you never wrote. Trivy scans all of that.

```yaml
trivy-scan:
  stage: trivy-scan
  needs:
    - job: build-image
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  before_script:
    - trivy --version
  script:
    - export IMAGE_FULL=$HARBOR_REGISTRY/$REPO_NAME:$STG_VERSION
    - echo "Scanning image $IMAGE_FULL"
    - trivy image --insecure --username $HARBOR_REGISTRY_USERNAME --password $HARBOR_REGISTRY_PASSWORD
        --exit-code 0 --no-progress --format json --output trivy-report.json $IMAGE_FULL
    - trivy image --insecure --username $HARBOR_REGISTRY_USERNAME --password $HARBOR_REGISTRY_PASSWORD
        --exit-code 0 --severity HIGH,CRITICAL $IMAGE_FULL
  artifacts:
    paths:
      - trivy-report.json
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "staging" && $CI_PIPELINE_SOURCE == "push"'
  allow_failure: true
```

It runs Trivy **twice on purpose**, and the reason is nice: the first pass writes JSON for DefectDojo (machine-readable, complete), the second prints only HIGH and CRITICAL to the job log (human-readable, scannable). One output for the tool, one for the person who opens the pipeline.

`needs: [build-image]` means it starts the moment the image exists, without waiting for unrelated jobs.

> 📁 [`trivy.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/security/trivy.yaml)

## DAST — scanning the running app

SAST and Trivy both inspect artefacts. DAST is different: it attacks a **live application** and watches how it responds. Which means it can only run after the app is actually deployed:

```yaml
dast-zap-scan:
  stage: dast-scan
  needs:
    - job: deploy-stag
  image: ghcr.io/zaproxy/zaproxy:stable
  variables:
    ZAP_TARGET: $DAST_TARGET_URL
  before_script:
    - mkdir -p /zap/wrk
  script:
    - zap-baseline.py -I -t $ZAP_TARGET -g gen.conf -x zap-report.xml -z "-config spider.postform=false"
    - cp /zap/wrk/zap-report.xml $CI_PROJECT_DIR
  artifacts:
    paths:
      - zap-report.xml
  rules:
    - if: '$CI_COMMIT_BRANCH == "staging" && $CI_COMMIT_TAG == null'
      when: always
  retry: 2
```

`needs: [deploy-stag]` is the important line, this is the last stage in the pipeline for a reason. It scans the staging deployment, never production.

Two flags worth explaining:

- **`-I`** tells ZAP not to fail the job on warnings. Baseline scans are noisy on first run; failing the pipeline on every informational finding trains people to ignore it.
- **`spider.postform=false`** stops ZAP from submitting forms while crawling. On a voting app, that would mean casting thousands of junk votes into the staging database.

`retry: 2` is there because DAST is the flakiest job in the pipeline, it depends on the app being up, warm, and reachable.

> 📁 [`dast.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/security/dast.yaml)

## DefectDojo — the glue

Three scanners, three formats, three separate UIs. DefectDojo is what turns that into one place to look.

It works in two parts.

### 1. Create an engagement, once per pipeline

An *engagement* in DefectDojo is a container for one round of testing. This job creates one at the very start (`stage: .pre`) and passes its ID to every later job:

```yaml
defectdojo_create_engagement:
  stage: .pre
  image: alpine:3.18
  variables:
    GIT_STRATEGY: none
  before_script:
    - apk add curl jq coreutils
    - TODAY=`date +%Y-%m-%d`
    - ENDDAY=$(date -d "+${DEFECTDOJO_ENGAGEMENT_PERIOD} days" +%Y-%m-%d)
  script:
    - |
      ENGAGEMENTID=`curl --fail --location --request POST "${DEFECTDOJO_URL}/api/v2/engagements/" \
        --header "Authorization: Token ${DEFECTDOJO_TOKEN}" \
        --header 'Content-Type: application/json' \
        --data-raw "{
          \"tags\": [\"GITLAB-CI\"],
          \"name\": \"#${CI_PIPELINE_ID}\",
          \"version\": \"${CI_COMMIT_REF_NAME}\",
          \"engagement_type\": \"CI/CD\",
          \"build_id\": \"${CI_PIPELINE_ID}\",
          \"commit_hash\": \"${CI_COMMIT_SHORT_SHA}\",
          \"product\": \"${DEFECTDOJO_PRODUCTID}\",
          \"source_code_management_uri\": \"${CI_PROJECT_URL}\"
        }" | jq -r '.id'`
    - echo "DEFECTDOJO_ENGAGEMENTID=${ENGAGEMENTID}" >> defectdojo.env
  artifacts:
    reports:
      dotenv: defectdojo.env
  allow_failure: true
```

The `dotenv` artifact is the trick that makes this work. The engagement ID is written to a file, and GitLab turns it into an environment variable available to every downstream job, so the Trivy and ZAP uploads know exactly which engagement to attach to.

Notice what gets recorded: pipeline ID, branch, commit hash, and a link back to the project. When a security reviewer opens a finding weeks later, they can trace it to the exact commit that introduced it.

### 2. A reusable upload job

The second half is a hidden template (the leading `.` means GitLab won't run it directly) that other jobs extend:

```yaml
.defectdojo_upload:
  stage: .post
  image: alpine:3.18
  before_script:
    - apk add curl coreutils
    - TODAY=`date +%Y-%m-%d`
  script:
    - cat ${DEFECTDOJO_SCAN_FILE}
    - |
      curl --fail --location --request POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
        --header "Authorization: Token ${DEFECTDOJO_TOKEN}" \
        --form "scan_date=\"${TODAY}\"" \
        --form "minimum_severity=\"${DEFECTDOJO_SCAN_MINIMUM_SEVERITY}\"" \
        --form "scan_type=\"${DEFECTDOJO_SCAN_TYPE}\"" \
        --form "engagement=\"${DEFECTDOJO_ENGAGEMENTID}\"" \
        --form "file=@${DEFECTDOJO_SCAN_FILE}" \
        --form "close_old_findings=\"${DEFECTDOJO_SCAN_CLOSE_OLD_FINDINGS}\"" \
        --max-time 300
  allow_failure: true
```

Then each scanner just supplies three variables:

```yaml
trivy-defectdojo:
  extends: .defectdojo_upload
  stage: trivy-scan
  needs:
    - job: defectdojo_create_engagement
    - job: trivy-scan
  variables:
    DEFECTDOJO_SCAN_TYPE: "Trivy Scan"
    DEFECTDOJO_SCAN_TEST_TYPE: "Trivy Scan"
    DEFECTDOJO_SCAN_FILE: "./trivy-report.json"

dast-defectdojo:
  extends: .defectdojo_upload
  needs:
    - job: defectdojo_create_engagement
    - job: dast-zap-scan
  variables:
    DEFECTDOJO_SCAN_TYPE: "ZAP Scan"
    DEFECTDOJO_SCAN_TEST_TYPE: "ZAP Scan"
    DEFECTDOJO_SCAN_FILE: "./zap-report.xml"
```

That's the payoff of the whole design. Adding a fourth scanner later means writing one small job with three variables, not another curl command with fifteen form fields.

One config choice worth calling out: `close_old_findings: true`. When a scan no longer reports a vulnerability you fixed, DefectDojo closes it automatically. Without this, the dashboard fills with issues that were resolved months ago and nobody trusts it any more.

> 📁 [`defect-dojo.yaml`](https://github.com/adelianurlinap/voting-app-devsecops/blob/main/security/defect-dojo.yaml)

## Putting it together

With the templates included, a service's own `.gitlab-ci.yml` only has to declare the stages — the jobs slot themselves in:

```yaml
stages:
  - prepare
  - sast          # ← sonarqube-check
  - verify
  - build-image
  - trivy-scan    # ← trivy-scan, trivy-defectdojo
  - update-tag-stg
  - release
  - approval
  - update-tag-prod
  - dast-scan     # ← dast-zap-scan, dast-defectdojo
```

Plus the two implicit stages GitLab provides: `.pre` (where the engagement is created, before everything) and `.post` (where uploads happen, after everything).

## An honest note on gates

Look closely and you'll spot `allow_failure: true` on nearly every security job and `--exit-code 0` on the Trivy scans. That means these scans **report, but don't block**.

That's a deliberate trade-off, and worth being upfront about. Scanning a real application for the first time surfaces hundreds of findings, most of them in dependencies you don't control. Hard-failing on all of them stops delivery on day one, and the usual outcome is that someone disables the scans entirely.

So this setup optimises for *visibility first*: get every finding tracked, triaged, and trending in DefectDojo. Tightening comes after, typically by failing on new HIGH/CRITICAL findings only, so the existing backlog doesn't block work while new problems can't slip in.

The exception is SonarQube. Because each service ships `sonar.qualitygate.wait=true` (from Part 2), the pipeline waits for the quality gate verdict rather than firing and forgetting, the closest thing to a real gate here.

## Verifying it works

After merging into `staging`, the pipeline runs and you can check each layer:

1. **SonarQube** — open the project, switch to the `staging` branch, confirm the analysis and quality gate.

![SonarQube scan results](/images/devsecops-voting-app/sonarqube-scan.png)

![SonarQube scan results 2](/images/devsecops-voting-app/sonarqube-scan-2.png)

2. **Harbor** — the image appears under `voting-app-adel/<service>` tagged `stg-<sha>`.

![Harbor image](/images/devsecops-voting-app/harbor-image.png)

3. **DefectDojo** — go to the product → **Engagements → View Engagements**. There's an engagement named after the pipeline ID, containing a **Trivy Scan** with the image findings.

![DefectDojo engagement showing Trivy scan results](/images/devsecops-voting-app/defectdojo-trivy-scan.png)

4. **DefectDojo again**, after the deploy — the same engagement now also has a **ZAP Scan** with runtime findings.

![DefectDojo engagement showing ZAP scan results](/images/devsecops-voting-app/defectdojo-zap-scan.png)

Seeing both scan types under one engagement, tied to one commit, is the moment the whole thing clicks: three tools, one story about one change.

## Two things I'd add next

If you're adapting this pipeline, there are two stages worth adding early, both of which make the existing jobs work better rather than just adding more of them.

**Split the build out of `build-image`.** Right now the Dockerfile does everything: restore dependencies, compile, and package. That means every image build repeats the full compile inside Docker, where caching is harder to control and the layers get heavier than they need to be. A dedicated `build-app` stage that compiles the application once and publishes the output as a CI artifact changes the shape of the job, `build-image` then only has to copy a ready-made artifact into a slim runtime image. The build itself gets faster, the images get smaller, and you get a clean separation between "did it compile" and "did it package".

**Add a `unit-test` stage that feeds SonarQube.** Tests are the obvious reason to add this, but the more interesting payoff is what it does for the SAST results. On its own, SonarQube reports code smells, bugs, and vulnerabilities, but the coverage figure stays empty, because nothing has told it which lines the tests actually exercise. If the unit-test job publishes a coverage report as an artifact, and the SAST job picks that artifact up, the SonarQube dashboard starts showing coverage alongside everything else. That turns the quality gate into a genuinely useful signal: you can require that *new* code is tested, not just that it's free of obvious defects.

The ordering matters, too. Both stages belong before `build-image`, there's no point building and pushing an image for code that doesn't compile or whose tests fail. Catching it two stages earlier saves a registry push and a Trivy scan on an artefact that was never going anywhere.

## What's next

The pipeline now scans everything and records what it finds. But it still hasn't deployed anything, the `update-tag-stg` job just edits a file in another repo.

**Part 4** covers what happens next: how ArgoCD notices that file change and reconciles it into the cluster, how the Helm chart is structured across staging and production, and how Vault injects secrets at runtime so no credential ever touches Git.

---

📁 **Source code:** [github.com/adelianurlinap/voting-app-devsecops](https://github.com/adelianurlinap/voting-app-devsecops)

**Series navigation:**

1. [Overview & Architecture](/page/posts/part-1-overview/)
2. [Foundation & Toolchain Setup](/page/posts/part-2-foundation/)
3. The Security Pipeline *(you are here)*
4. GitOps Deployment *(soon)*
5. Branching & Releases *(soon)*
6. Day-2 Operations *(soon)*