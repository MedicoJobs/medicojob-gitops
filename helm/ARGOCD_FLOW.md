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

## Branch Promotion

Dev and prod are intentionally separated by branch:

```text
dev namespace  -> feature-branch
prod namespace -> main
```

Service Docker pipelines update `values-dev.yaml` on `feature-branch`. Production is not updated directly by service repos.

After a reviewer approves and merges `feature-branch` into `main`, `.github/workflows/promote-prod-on-merge.yml` copies the reviewed image tag from the changed `values-dev.yaml` file into the matching `values-prod.yaml` file on `main`. Only then can prod Argo CD deploy the new image.

## GHCR Images

The app charts use image values from each service values file:

```text
helm/apps/<service>/values-dev.yaml
helm/apps/<service>/values-prod.yaml
```

Examples:

```text
image.repository: ghcr.io/medicojobs/medicojob-user-service
image.tag: <short-sha-7>
```

Each service Docker pipeline should build and push a Docker image with the first 7 characters of the merge commit SHA, then commit that same tag into this GitOps repo. Argo CD watches this repo and syncs the new Deployment automatically.

See `helm/CI_PIPELINE_TEMPLATE.md` and `helm/pipeline-templates/docker_service_pipeline_auto.yml` for the workflow template. Copy that same workflow into every service repo so each repo can update its own Helm values file in this GitOps repo.

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
