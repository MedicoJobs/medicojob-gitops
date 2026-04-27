# MedicoJob Microservice CI Pipeline

Use this workflow in each microservice repository, for example:

- `medicojob-user-service`
- `medicojob-job-service`
- `medicojob-api-gateway`
- `medicojob-frontend`

Argo CD should watch this GitOps repo, not every service repo. The service repos only build images and update the matching Helm values file here.

Copy these two template files into every microservice repo:

```text
helm/pipeline-templates/docker_service_pipeline_auto.yml
helm/pipeline-templates/docker_service_pipeline_caller.yml
```

Put them in the service repo like this:

```text
.github/workflows/docker-ci.yml
.github/workflows/docker_service_pipeline.yml
```

`docker-ci.yml` is the reusable workflow. `docker_service_pipeline.yml` is the trigger workflow that runs after a PR is merged with the `build` label.

## Recommended Flow

```text
Developer opens PR
        |
        v
CI runs Sonar, Snyk, and review checks
        |
        v
Reviewer merges PR with label: build
        |
        v
Docker pipeline builds and pushes image
        |
        v
Push image to GHCR with tag = short SHA, 7 chars
        |
        v
Pipeline updates this GitOps repo:
helm/apps/<service>/values-dev.yaml image.tag = <short-sha-7>
        |
        v
Dev Argo CD detects feature-branch Git change
        |
        v
Dev Argo CD syncs Helm chart to dev namespace
        |
        v
Reviewer reviews GitOps feature-branch and merges to main
        |
        v
GitOps promotion workflow updates values-prod.yaml on main
        |
        v
Prod Argo CD syncs Helm chart to prod namespace
```

Do not depend on `latest` for continuous deployment. Kubernetes may keep running the old image, and Git will not show what version is deployed. Use immutable tags such as the first 7 characters of the merge commit SHA.

## Required GitHub Secret

In every microservice repo, create this secret:

```text
GITOPS_TOKEN
```

The token must be allowed to push to `MedicoJobs/medicojob-gitops` or your actual GitOps repo name. A fine-grained GitHub token with `Contents: Read and write` on only the GitOps repo is enough.

## GitHub Actions Template

The caller workflow is also available as `helm/pipeline-templates/docker_service_pipeline_caller.yml`:

```yaml
name: Docker Service Pipeline

on:
  pull_request:
    types:
      - closed
    branches:
      - main

jobs:
  docker:
    if: github.event.pull_request.merged == true && contains(github.event.pull_request.labels.*.name, 'build')
    uses: ./.github/workflows/docker-ci.yml
    with:
      service_name: frontend
      service_path: .
      gitops_branch: feature-branch
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
```

Use `service_path: .` when the Dockerfile is at the repo root. If the Dockerfile is inside a folder, pass that folder path.

The reusable workflow below is the content of `.github/workflows/docker-ci.yml`:

Change these values for each service:

- `SERVICE_NAME`
- `IMAGE_NAME`
- `GITOPS_REPO`
- `VALUES_FILE`

## Service Values

The auto template has these repo-to-GitOps mappings:

| GitHub repo | `SERVICE_NAME` | `IMAGE_NAME` | GitOps `VALUES_FILE` |
| --- | --- | --- | --- |
| `MedicoJobs/medicojob-api-gateway` | `api-gateway` | `ghcr.io/medicojobs/medicojob-api-gateway` | `helm/apps/api-gateway/values-dev.yaml` |
| `MedicoJobs/medicojob-availability-service` | `availability-service` | `ghcr.io/medicojobs/medicojob-availability-service` | `helm/apps/availability-service/values-dev.yaml` |
| `MedicoJobs/medicojob-frontend` | `frontend` | `ghcr.io/medicojobs/medicojob-frontend` | `helm/apps/frontend/values-dev.yaml` |
| `MedicoJobs/medicojob-job-service` | `job-service` | `ghcr.io/medicojobs/medicojob-job-service` | `helm/apps/job-service/values-dev.yaml` |
| `MedicoJobs/medicojob-location-service` | `location-service` | `ghcr.io/medicojobs/medicojob-location-service` | `helm/apps/location-service/values-dev.yaml` |
| `MedicoJobs/medicojob-matching-service` | `matching-service` | `ghcr.io/medicojobs/medicojob-matching-service` | `helm/apps/matching-service/values-dev.yaml` |
| `MedicoJobs/medicojob-reputation-service` | `reputation-service` | `ghcr.io/medicojobs/medicojob-reputation-service` | `helm/apps/reputation-service/values-dev.yaml` |
| `MedicoJobs/medicojob-user-service` | `user-service` | `ghcr.io/medicojobs/medicojob-user-service` | `helm/apps/user-service/values-dev.yaml` |

```yaml
name: Docker image and GitOps deploy

on:
  pull_request:
    types:
      - closed
    branches:
      - main

permissions:
  contents: read
  packages: write
  pull-requests: read

env:
  SERVICE_NAME: user-service
  IMAGE_NAME: ghcr.io/medicojobs/medicojob-user-service
  GITOPS_REPO: MedicoJobs/medicojob-gitops
  GITOPS_BRANCH: feature-branch
  VALUES_FILE: helm/apps/user-service/values-dev.yaml

jobs:
  build-push-update-gitops:
    if: github.event.pull_request.merged == true && contains(github.event.pull_request.labels.*.name, 'build')
    runs-on: ubuntu-latest

    steps:
      - name: Checkout service repo
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.merge_commit_sha }}

      - name: Create short image tag
        run: |
          SHORT_SHA="$(echo "${{ github.event.pull_request.merge_commit_sha }}" | cut -c1-7)"
          echo "SHORT_SHA=$SHORT_SHA" >> "$GITHUB_ENV"

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:${{ env.SHORT_SHA }}

      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          ref: ${{ env.GITOPS_BRANCH }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Install yq
        run: |
          sudo wget -q https://github.com/mikefarah/yq/releases/download/v4.44.3/yq_linux_amd64 -O /usr/local/bin/yq
          sudo chmod +x /usr/local/bin/yq

      - name: Update Helm image tag
        working-directory: gitops
        run: |
          yq -i '.image.repository = strenv(IMAGE_NAME) | .image.tag = strenv(SHORT_SHA) | .image.pullPolicy = "IfNotPresent"' "$VALUES_FILE"

      - name: Commit and push GitOps change
        working-directory: gitops
        run: |
          git config user.name "medicojob-ci"
          git config user.email "medicojob-ci@users.noreply.github.com"
          git add "$VALUES_FILE"
          git commit -m "deploy($SERVICE_NAME): $SHORT_SHA to dev" || exit 0
          git push origin "$GITOPS_BRANCH"
```

## Production Promotion

For production, do not let service repos update prod directly. Production is promoted from the reviewed GitOps branch:

```text
feature-branch -> reviewed PR -> main
```

The workflow `.github/workflows/promote-prod-on-merge.yml` runs in this GitOps repo after the PR from `feature-branch` to `main` is merged. It copies the changed service image tag from:

```text
helm/apps/<service>/values-dev.yaml
```

to:

```text
helm/apps/<service>/values-prod.yaml
```

## Argo CD Settings

Your app manifests already use the important settings:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

That means once CI commits a new image tag into this repo, Argo CD will continuously reconcile the cluster to match Git.

## Optional: Argo CD Image Updater

Another valid approach is Argo CD Image Updater. It watches image registries and writes new tags back to Git. For this project, the CI-commits-to-GitOps approach is simpler and easier to audit because each deployment is a normal Git commit.
