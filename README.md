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
2. Push it to GHCR with an immutable tag, usually the Git commit SHA.
3. Commit that new tag into this repo under the correct Helm values file.
4. Argo CD sees the Git change and automatically syncs the Kubernetes Deployment.

See [helm/CI_PIPELINE_TEMPLATE.md](helm/CI_PIPELINE_TEMPLATE.md) for the GitHub Actions template to copy into each microservice repo.
