# MedicoJob Argo CD Flow

This folder uses an app-of-apps layout.

## Bootstrap

Install Argo CD in the cluster first, then apply one environment root:

```bash
kubectl apply -f helm/env/dev/root/infra-root.yaml
kubectl apply -f helm/env/dev/root/apps-root.yaml
```

For production:

```bash
kubectl apply -f helm/env/prod/root/infra-root.yaml
kubectl apply -f helm/env/prod/root/apps-root.yaml
```

## Sync Order

1. `infra-root` creates infrastructure Applications.
2. `app-namespace` creates the `dev` or `prod` namespace, `medicojob-config`, and `medicojob-secrets`.
3. `gateway` creates the Gateway API `Gateway`.
4. `apps-root` creates one Argo CD Application per microservice.
5. Each app chart deploys its Deployment, Service, HTTPRoute, ServiceAccount, and optional Job.

## GHCR Images

The app charts use image values from each service values file:

```text
helm/apps/<service>/values-dev.yaml
helm/apps/<service>/values-prod.yaml
```

Examples:

```text
image.repository: ghcr.io/medicojobs/medicojob-user-service
image.tag: <git-commit-sha>
```

Each service CI pipeline should build and push a Docker image, then commit the new immutable tag into this GitOps repo. Argo CD watches this repo and syncs the new Deployment automatically.

See `helm/CI_PIPELINE_TEMPLATE.md` for the workflow template.

If GHCR packages are private, create a Kubernetes pull secret and set `imagePullSecrets` in each service values file.

## Secrets

Secrets are not hardcoded in Git. The `app-namespace` chart creates `medicojob-config` from normal values, but `medicojob-secrets` is disabled by default:

```yaml
secret:
  create: false
  name: medicojob-secrets
  data: {}
```

Create the secret outside Git, or pass secret values through a private Argo CD values source, Sealed Secrets, External Secrets, or Argo CD parameters. The shape is documented in:

```text
helm/infra/app-namespace/values-secret-example.yaml
```
