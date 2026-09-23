# MongoDB + mongo-express on Kubernetes

Demo architecture:

```text
mongo-express -> mongodb-service:27017 -> MongoDB Pod
       |                                      |
   ConfigMap                            Secret credentials
```

## Files

- `mongo-secret.yaml` — MongoDB username and password.
- `mongo-configmap.yaml` — internal MongoDB service address.
- `mongo.yaml` — MongoDB Deployment and ClusterIP Service.
- `mongo-express.yaml` — mongo-express Deployment.

Labels connect each Service to its Pods. Resource names and Secret/ConfigMap keys must match all references exactly.

## Deploy safely

```bash
# Check the destination before making changes
kubectl config current-context
kubectl config view --minify --output 'jsonpath={..namespace}'; echo

# Validate and review
kubectl apply --dry-run=server -f .
kubectl diff -f .

# Apply dependencies first
kubectl apply -f mongo-secret.yaml -f mongo-configmap.yaml
kubectl apply -f mongo.yaml
kubectl rollout status deployment/mongodb-deployment --timeout=5m
kubectl apply -f mongo-express.yaml
kubectl rollout status deployment/mongo-express --timeout=5m
```

Verify and test without exposing the UI publicly:

```bash
kubectl get deployments,pods,services
kubectl get endpoints mongodb-service
kubectl logs deployment/mongodb-deployment --tail=100
kubectl logs deployment/mongo-express --tail=100
kubectl port-forward deployment/mongo-express 8081:8081
# Open http://localhost:8081
```

Quick troubleshooting:

```bash
kubectl describe pod -l app=mongodb
kubectl describe pod -l app=mongo-express
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Before production

- Replace the committed demo credentials immediately. Base64 is encoding, not encryption; use an external secret manager.
- Use a managed MongoDB service, or a StatefulSet with persistent volumes, backups, and tested restores. This Deployment loses data when its Pod is replaced.
- Pin both images to immutable digests; `mongo-express` currently has no version.
- Add resource requests/limits, health probes, security contexts, and a PodDisruptionBudget.
- Keep MongoDB private and protect mongo-express with authentication and network policy; do not expose it directly to the internet.
- Add an explicit namespace and standard ownership/application labels.
- Check `ME_CONFIG_MONGODB_URL`: Kubernetes variable references use `$(NAME)`, not `${NAME}`. Prefer supported mongo-express variables or construct the value in a controlled entrypoint.
- Deploy through reviewed GitOps/CI/CD changes and monitor application/database health.
