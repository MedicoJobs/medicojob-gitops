# medicojob-gitops

This repository is the only repository Argo CD should watch.

It contains:

- Helm charts for all services
- Environment app-of-apps definitions
- Infra charts
- CI/CD flow documentation for updating Helm image tags from service repos

Bootstrap dev:

```bash
kubectl apply -f helm/env/dev/root/infra-root.yaml
kubectl apply -f helm/env/dev/root/apps-root.yaml
```

Bootstrap prod:

```bash
kubectl apply -f helm/env/prod/root/infra-root.yaml
kubectl apply -f helm/env/prod/root/apps-root.yaml
```

## Deployment flow

Each microservice should have its own CI pipeline:

1. Build the Docker image.
2. Push it to GHCR with an immutable tag, the first 7 characters of the merge commit SHA.
3. Commit that new tag into this repo under the correct `values-dev.yaml` file on `feature-branch`.
4. Dev Argo CD watches `feature-branch` and deploys to the `dev` namespace.
5. A reviewer reviews and merges `feature-branch` into `main`.
6. The GitOps promotion workflow copies reviewed dev image tags into `values-prod.yaml` on `main`.
7. Prod Argo CD watches `main` and deploys to the `prod` namespace.

See [helm/CI_PIPELINE_TEMPLATE.md](helm/CI_PIPELINE_TEMPLATE.md) for the GitHub Actions template to copy into each microservice repo.
