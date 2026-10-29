# Day 18 - ConfigMaps, Secrets & Namespaces

DevOps Winter Arc | Day 18/90 | Week 3: Kubernetes

## Objective

Run the same Nginx workload in two namespaces (`dev` and `prod`) with
different configuration. Use a ConfigMap for the app setting and page
content, a Secret for a credential, and observe how mounted files and
environment variables behave when configuration changes.

## Key concepts

| Resource | Purpose | In this lab |
|---|---|---|
| Namespace | Logical separation of resources inside a cluster | `dev` and `prod` |
| ConfigMap | Non-sensitive configuration, kept apart from the image | `site`: `APP_ENV` and `index.html` |
| Secret | Sensitive values, with separate access control | `db-secret`: `password` (dummy values) |
| Environment variable | Value injected when the container starts | `APP_ENV`, `DB_PASSWORD` |
| Volume mount | Config exposed as files inside the container | ConfigMap mounted at `/usr/share/nginx/html` |

The same `deployment.yaml` is applied to both namespaces. ConfigMap and Secret
names are identical in each, but their values differ, so each Pod reads
the values from its own namespace.

## Files

- `configmap-dev.yaml`: ConfigMap `site` for dev (`APP_ENV: dev`, page "Hello from DEV")
- `configmap-prod.yaml`: ConfigMap `site` for prod (`APP_ENV: prod`, page "Hello from PROD")
- `deployment.yaml`: Deployment `web` (`nginx:1.27`) using the ConfigMap and Secret
- `evidence/`: screenshots

Secrets were created with `kubectl create secret` using dummy values, so no
Secret manifest or real credential is stored in this repository.

## Setup steps

1. Create the namespaces:

   ```bash
   kubectl create namespace dev
   kubectl create namespace prod
   kubectl get namespaces
   ```

2. Create the ConfigMaps:

   ```bash
   kubectl apply -f configmap-dev.yaml -n dev
   kubectl apply -f configmap-prod.yaml -n prod
   kubectl get configmap site -n dev
   kubectl get configmap site -n prod
   ```

3. Create the lab-only Secrets (dummy values):

   ```bash
   kubectl create secret generic db-secret --from-literal=password=<dummy> -n dev
   kubectl create secret generic db-secret --from-literal=password=<dummy> -n prod
   ```

4. Deploy the same workload to both namespaces:

   ```bash
   kubectl apply -f deployment.yaml -n dev
   kubectl apply -f deployment.yaml -n prod
   kubectl get pods -n dev
   kubectl get pods -n prod
   ```

5. Verify each environment:

   ```bash
   kubectl port-forward -n dev deploy/web 8081:80
   kubectl port-forward -n prod deploy/web 8082:80
   curl http://localhost:8081
   curl http://localhost:8082
   kubectl exec -n dev deploy/web -- printenv APP_ENV
   kubectl exec -n prod deploy/web -- printenv APP_ENV
   ```

## Observations

### Namespace separation

- dev page output: **[write what curl returned for 8081]**
- prod page output: **[write what curl returned for 8082]**
- `APP_ENV` in dev: **[value]**, in prod: **[value]**

Evidence: `evidence/[dev-page.png]`, `evidence/[prod-page.png]`, `evidence/[env-values.png]`

The Secret value is injected as `DB_PASSWORD`. Screenshots mask the value, and
it is not recorded in this README.

### Configuration update behavior

I edited the dev ConfigMap with `kubectl edit configmap site -n dev`,
changing `APP_ENV` and the page text.

- Mounted page file: **[describe what you saw and roughly how long it took]**
- `APP_ENV` environment variable before restart: **[value]**
- After `kubectl rollout restart deployment/web -n dev`: **[value]**

```bash
kubectl rollout restart deployment/web -n dev
kubectl rollout status deployment/web -n dev
kubectl exec -n dev deploy/web -- printenv APP_ENV
```

Evidence: `evidence/[config-update.png]`

Note: ConfigMap volumes mounted with `subPath` do not receive automatic
updates. This lab mounts the ConfigMap as a directory, so updates are applied.

## Questions and answers

**1. What is the difference between a ConfigMap and a Secret?**
A ConfigMap stores non-sensitive configuration. A Secret stores sensitive data
such as passwords or tokens and supports tighter access control.

**2. Why must referenced ConfigMaps and Secrets exist in the Pod's namespace?**
A Pod looks up ConfigMaps and Secrets by name within its own namespace only.
If the object is in another namespace, the Pod cannot find it and fails with
`CreateContainerConfigError`.

**3. What is the difference between environment variables and mounted files?**
Environment variables are set once when the container starts. Mounted files
live in a volume that Kubernetes refreshes after a ConfigMap changes (unless
`subPath` is used).

**4. Why does changing a ConfigMap not update an existing container's environment automatically?**
The environment of a running process is fixed at startup. A new Pod is needed
to pick up new values, for example with `kubectl rollout restart`.

**5. Why is base64 not sufficient to protect a Secret?**
Base64 is an encoding, not encryption. Anyone with read access can decode it
with `base64 -d`. Real protection needs RBAC, encryption at rest, and
ideally a dedicated secrets manager.

**6. What can cause CreateContainerConfigError?**
A missing ConfigMap or Secret, an object in the wrong namespace, or a key
name that does not match the reference in the Deployment.

**7. How do namespaces help separate dev and prod resources?**
Resource names are scoped to a namespace, so `site`, `db-secret` and `web` can
exist in both with different values. Each environment can be managed and
deleted on its own. Namespaces are a logical separation, not a hard security
boundary, so RBAC and NetworkPolicy are needed for stronger isolation.

## Troubleshooting / lessons learned

- **[Write one problem you actually hit, what command showed it, and how you fixed it. If none, write what you learned from `kubectl describe pod`.]**
- Both objects must exist in the same namespace as the Pod before it can start.
- Env variables and mounted files react differently to ConfigMap changes.
- Base64 is not encryption.

## Security notes

- Only dummy credentials were used.
- No Secret manifests, kubeconfig files, or tokens are committed.
- Screenshots do not show the Secret value.

## Cleanup

Run only after saving all evidence:

```bash
kubectl delete namespace dev
kubectl delete namespace prod
```

This removes every lab resource inside both namespaces.
