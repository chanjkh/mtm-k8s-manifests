```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get password of Argo CD
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("$(kubectl.exe get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}')"))

# Port forward for testing
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Create password for Redis
kubectl create namespace mtm-vtg-uat
kubectl create secret generic mtm2-redis-creds --from-literal=REDIS_PASSWORD='your-password' --namespace mtm-vtg-uat

kubectl.exe apply -f argocd\mtm-vtg-uat-project.yaml
kubectl.exe apply -f mtm-vtg-uat-application.yaml

kubectl.exe delete -f mtm-vtg-uat-application.yaml
```