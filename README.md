# infra-gitops

Desired state for the todo app cluster (GitOps). ArgoCD watches this repo and syncs it to k3s.

## Layout

| Path | Purpose |
|---|---|
| `apps/` | ArgoCD Applications (app-of-apps via `root-app.yaml`) |
| `charts/todo-app/` | Helm chart — frontend, backend, Postgres, ingress |
| `platform/` | Namespaces, cert-manager issuers, monitoring & Velero values |
| `jobs/` | CronJobs — DB backup, image cleanup reminder |

## CI

GitHub Actions (`.github/workflows/ci.yaml`) runs on every push/PR to `main`:

| Job | Checks |
|---|---|
| **Helm chart** | `helm lint` (default + prod values), `helm template`, **kubeconform** schema validation, `helm package`, Chart.yaml semver |
| **Manifests** | kubeconform on `jobs/` + `platform/namespace/`, YAML parse of non-templated manifests |

Packaged chart artifact: `todo-app-chart` (`.tgz`).

### Run locally

```bash
helm lint ./charts/todo-app --strict -f ./charts/todo-app/values-prod.yaml
helm template todo-app ./charts/todo-app -f ./charts/todo-app/values-prod.yaml --namespace todo
helm package ./charts/todo-app --destination dist
```

## Bootstrap order

1. Install ArgoCD on the cluster  
2. `kubectl apply -f apps/root-app.yaml`  
3. Root app syncs child apps (namespaces → cert-manager → todo-app → monitoring → velero → jobs)

## Deploy flow

1. `app-todo` CI builds and pushes images to Docker Hub  
2. Image tag updated in `charts/todo-app/values-prod.yaml`  
3. ArgoCD syncs → cluster runs the new version  

## Notes

- Single env: **prod** on one EC2 / k3s node (`t3.micro` is tight — monitoring/Velero may need to stay paused)  
- Replace `changeme` secrets and `admin@example.com` before real TLS  
- Velero needs a `cloud-credentials` secret in the `velero` namespace  
- DB backup CronJob uploads to `s3://todo-app-tfstate-victor/db-backups/` (EC2 instance role)
