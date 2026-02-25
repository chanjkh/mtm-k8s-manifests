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

## Install Redis Sentinel manually

If you want to install Redis Operator and Redis Sentinel manually.
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
