# MedicoJob Microservice CI Pipeline

Use this workflow in each microservice repository, for example:

- `medicojob-user-service`
- `medicojob-job-service`
- `medicojob-api-gateway`
- `medicojob-frontend`

Argo CD should watch this GitOps repo, not every service repo. The service repos only build images and update the matching Helm values file here.

## Recommended Flow

```text
Developer pushes code
        |
        v
Service repo GitHub Actions builds Docker image
        |
        v
Push image to GHCR with tag = commit SHA
        |
        v
Pipeline updates this GitOps repo:
helm/apps/<service>/values-dev.yaml image.tag = <commit SHA>
        |
        v
Argo CD detects Git change
        |
        v
Argo CD syncs Helm chart to Kubernetes
```

Do not depend on `latest` for continuous deployment. Kubernetes may keep running the old image, and Git will not show what version is deployed. Use immutable tags such as `${{ github.sha }}`.

## Required GitHub Secret

In every microservice repo, create this secret:

```text
GITOPS_TOKEN
```

The token must be allowed to push to `MedicoJobs/medicojob-gitops` or your actual GitOps repo name. A fine-grained GitHub token with `Contents: Read and write` on only the GitOps repo is enough.

## GitHub Actions Template

Save this as `.github/workflows/docker-gitops-dev.yml` in each microservice repo.

Change these values for each service:

- `SERVICE_NAME`
- `IMAGE_NAME`
- `GITOPS_REPO`
- `VALUES_FILE`

## Service Values

Use these values in each microservice repo:

| Repo/service | `SERVICE_NAME` | `IMAGE_NAME` | `VALUES_FILE` |
| --- | --- | --- | --- |
| API gateway | `api-gateway` | `ghcr.io/medicojobs/medicojob-api-gateway` | `helm/apps/api-gateway/values-dev.yaml` |
| Availability service | `availability-service` | `ghcr.io/medicojobs/medicojob-availability-service` | `helm/apps/availability-service/values-dev.yaml` |
| Frontend | `frontend` | `ghcr.io/medicojobs/medicojob-frontend` | `helm/apps/frontend/values-dev.yaml` |
| Job service | `job-service` | `ghcr.io/medicojobs/medicojob-job-service` | `helm/apps/job-service/values-dev.yaml` |
| Location service | `location-service` | `ghcr.io/medicojobs/medicojob-location-service` | `helm/apps/location-service/values-dev.yaml` |
| Matching service | `matching-service` | `ghcr.io/medicojobs/medicojob-matching-service` | `helm/apps/matching-service/values-dev.yaml` |
| Reputation service | `reputation-service` | `ghcr.io/medicojobs/medicojob-reputation-service` | `helm/apps/reputation-service/values-dev.yaml` |
| User service | `user-service` | `ghcr.io/medicojobs/medicojob-user-service` | `helm/apps/user-service/values-dev.yaml` |

```yaml
name: Build image and update GitOps dev

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

env:
  SERVICE_NAME: user-service
  IMAGE_NAME: ghcr.io/medicojobs/medicojob-user-service
  GITOPS_REPO: MedicoJobs/medicojob-gitops
  GITOPS_BRANCH: main
  VALUES_FILE: helm/apps/user-service/values-dev.yaml

jobs:
  build-and-update-gitops:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout service repo
        uses: actions/checkout@v4

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
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

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
          yq -i '.image.repository = strenv(IMAGE_NAME) | .image.tag = strenv(GITHUB_SHA) | .image.pullPolicy = "IfNotPresent"' "$VALUES_FILE"

      - name: Commit and push GitOps change
        working-directory: gitops
        run: |
          git config user.name "medicojob-ci"
          git config user.email "medicojob-ci@users.noreply.github.com"
          git add "$VALUES_FILE"
          git commit -m "deploy($SERVICE_NAME): $GITHUB_SHA to dev" || exit 0
          git push
```

## Production Promotion

For production, use a separate workflow with manual approval and update:

```text
helm/apps/<service>/values-prod.yaml
```

Recommended policy:

- `main` branch deploys to `dev` automatically.
- Production deploys only from a manual `workflow_dispatch` or a GitHub Release.
- The prod workflow copies a tested image tag from `values-dev.yaml` to `values-prod.yaml`.

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
