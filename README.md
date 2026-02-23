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
