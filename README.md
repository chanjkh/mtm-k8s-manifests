# MTM Kubernetes Manifests

A small repo organizing Kustomize overlays, Argo CD application manifests, and apps deployment manifests used for MTM.

## Repository Structure

- `argocd/` — Project-level Argo CD YAMLs.
- `apps/` — Base Kubernetes manifests and environment overlays (Kustomize).
- `helms/` — Argo CD application manifests and kustomizations (App-of-apps + Kustomize).
- `mtm-apps-applicationset.yaml` — Top-level Argo CD application set YAML for APP (ApplicationSet - all env x all app). 
- `mtm-helms-applicationset.yaml` — Top-level Argo CD application set YAML for HELM (ApplicationSet - all env). 

## Getting Started

```
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get the password for Argo CD login
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("$(kubectl.exe get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}')"))

# Port forward for Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Create the password for Redis
kubectl create namespace mtm-vtg-uat
kubectl create namespace mtm-tgt-uat
kubectl create secret generic mtm2-redis-creds --from-literal=REDIS_PASSWORD='your-password' --namespace mtm-vtg-uat
kubectl create secret generic mtm2-redis-creds --from-literal=REDIS_PASSWORD='your-password' --namespace mtm-tgt-uat

# Install applications to Argo CD
kubectl.exe apply -f argocd\mtm-vtg-uat-project.yaml
kubectl.exe apply -f argocd\mtm-tgt-uat-project.yaml
kubectl.exe apply -f mtm-helm-applicationset.yaml
kubectl.exe apply -f mtm-app-applicationset.yaml

# Uninstall applications from Argo CD
kubectl.exe delete -f mtm-app-applicationset.yaml
kubectl.exe delete -f mtm-helm-applicationset.yaml
```

## How to Deploy a New App

### Helm-based App (via `helms/`)

Use this approach when deploying a third-party Helm chart (e.g., Redis Operator, Airflow) through Argo CD.

1. **Create a base Argo CD Application manifest** in `helms/base/`:
   ```
   helms/base/<app-name>-application.yaml
   ```
   Use placeholders (`PROJECT_PLACEHOLDER`, `SERVER_PLACEHOLDER`, `NAMESPACE_PLACEHOLDER`) for environment-specific values. See `helms/base/redis-operator-application.yaml` for reference.

2. **Register the new resource** in `helms/base/kustomization.yaml`:
   ```yaml
   resources:
     - <app-name>-application.yaml
   ```

3. **Create a patch file for each environment overlay** (e.g., `vtg-uat`, `tgt-uat`):
   ```
   helms/overlays/<env>/<app-name>-application-patch.yaml
   ```
   The patch should replace the placeholders with actual values (project, server, namespace, helm values). See `helms/overlays/vtg-uat/redis-operator-application-patch.yaml` for reference.

4. **Register the patch** in each overlay's `helms/overlays/<env>/kustomization.yaml`:
   ```yaml
   patches:
     - path: <app-name>-application-patch.yaml
       target:
         kind: Application
         name: <app-name>-application
   ```

5. **Commit and push.** The existing `mtm-helm-applicationset.yaml` will automatically pick up the changes for all environments.

### Non-Helm App (via `apps/`)

Use this approach for plain Kubernetes manifests (e.g., Redis Sentinel CRs, Jobs) managed by Kustomize.

1. **Create the base manifests** in a new directory under `apps/`:
   ```
   apps/<app-name>/base/
   ```
   Add your Kubernetes YAML files and a `kustomization.yaml` that lists them as resources.

2. **Create an overlay for each environment** (e.g., `vtg-uat`, `tgt-uat`):
   ```
   apps/<app-name>/overlays/<env>/kustomization.yaml
   ```
   At minimum, set the namespace and reference the base:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   namespace: mtm-<env>
   resources:
     - ../../base
   ```
   Add patches here if you need environment-specific overrides.

3. **Commit and push.** The existing `mtm-app-applicationset.yaml` auto-discovers any new directory under `apps/*` and generates an Argo CD Application for every environment listed in its generator.

## How to Skip Deploying an App to a Specific Environment

### Helm-based App

To skip a helm app (e.g., `airflow`) for a specific environment (e.g., `tgt-uat`), add a delete patch in that environment's overlay to remove the resource inherited from the base.

1. **Create a delete patch file** at `helms/overlays/<env>/<app-name>-application-delete.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <app-name>-application
   $patch: delete
   ```

2. **Register the delete patch** in `helms/overlays/<env>/kustomization.yaml`:
   ```yaml
   patches:
     - path: <app-name>-application-delete.yaml
   ```
   And remove the corresponding regular patch entry for that app if it exists.

### Non-Helm App

The `mtm-app-applicationset.yaml` generates an Application for every combination of `apps/*` directory and environment. To skip an app for a specific environment, create an empty overlay so the generated Application deploys nothing.

1. **Create an empty overlay** at `apps/<app-name>/overlays/<env>/kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   # intentionally empty — skip this app for this environment
   ```
   Do **not** include `../../base` in the resources list. This ensures the Application exists in Argo CD (avoiding sync errors) but deploys no resources.

## Install Redis Sentinel manually

If you need to install the Redis Operator and Redis Sentinel manually, please follow the steps below. This should only be done for troubleshooting purposes.
```
# Install Redis Operator and Redis Sentinel (⚠️ Bitnami Redis is No Longer Free)
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update
helm upgrade redis-operator ot-helm/redis-operator --install --create-namespace --namespace mtm-vtg-uat
kubectl create secret generic mtm2-redis-creds --from-literal=REDIS_PASSWORD='your-password' --namespace mtm-vtg-uat
kubectl apply -f redis-sentinel.yaml -n mtm-vtg-uat

# You can verify the API Version of Redis Operator by running:
kubectl api-versions

# You can verify the Redis config by running:
kubectl exec -it redis-sentinel-sentinel-0 -n mtm-vtg-uat -- redis-cli -p 26379 sentinel masters

# If you need to forward sentinel port to localhost for testing:
kubectl port-forward svc/redis-sentinel-sentinel 26379:26379 -n mtm-vtg-uat

# If you want to test the Redis connection:
kubectl apply -f redis-test-job.yaml -n mtm-vtg-uat
kubectl logs job/redis-test -n mtm-vtg-uat
kubectl delete -f redis-test-job.yaml -n mtm-vtg-uat

# If you need to uninstall everythings:
kubectl delete -f redis-sentinel.yaml -n mtm-vtg-uat
helm uninstall redis-operator -n mtm-vtg-uat
kubectl delete secret redis-secret -n mtm-vtg-uat
kubectl delete namespace mtm-vtg-uat
```

## How to Change Reconciliation Time of Argo CD

Argo CD defaults to a 180-second (3-minute) interval to reconcile the desired Git state with the live cluster state. This polling frequency can be adjusted or disabled (set to 0) by updating the timeout.reconciliation key in the argocd-cm ConfigMap.

1. Edit ConfigMap
```
180s to 60s example
kubectl edit cm argocd-cm -n argocd -o yaml
```
2. Update Value: Add or modify timeout.reconciliation: 60s under the data: field (180s to 60s example)
```
apiVersion: v1
data:
  timeout.reconciliation: 60s
  timeout.reconciliation.jitter: 0s
```
3. Restart Controller
```
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout restart statefulset argocd-application-controller -n argocd
```

## References

* https://artifacthub.io/packages/helm/ot-container-kit/redis-operator
* https://github.com/OT-CONTAINER-KIT/redis-operator
* https://github.com/OT-CONTAINER-KIT/redis-operator/issues/1503
