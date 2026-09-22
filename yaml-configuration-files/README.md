# Kubernetes YAML quick reference

Use declarative YAML in Git; let Kubernetes reconcile the cluster to the declared state.

## Files here

- `nginx-deployment.yaml` — runs two NGINX Pods.
- `nginx-service.yaml` — exposes Pods whose label is `app: nginx`.
- `nginx-deployment-result.yaml` — exported live state for inspection only; do not apply it.

Every manifest starts with:

```yaml
apiVersion: apps/v1       # Kubernetes API version
kind: Deployment          # Resource type
metadata:
  name: nginx-deployment  # Resource name
spec:                     # Desired state
```

For Deployments, these labels must match exactly:

```yaml
spec.selector.matchLabels == spec.template.metadata.labels
```

The Service `spec.selector` must also match the Pod labels. `port` is the Service port; `targetPort` is the port on the container.

## Safe deployment workflow

```bash
# Confirm the target cluster and namespace first
kubectl config current-context
kubectl config view --minify --output 'jsonpath={..namespace}'; echo

# Validate locally, then ask the API server to validate
kubectl apply --dry-run=client -f .
kubectl apply --dry-run=server -f .

# Review the change before applying it
kubectl diff -f .
kubectl apply -f .

# Verify rollout and connectivity
kubectl rollout status deployment/nginx-deployment --timeout=5m
kubectl get deployments,pods,services -o wide
kubectl get endpoints nginx-service
```

Inspect and troubleshoot:

```bash
kubectl describe deployment nginx-deployment
kubectl describe service nginx-service
kubectl logs deployment/nginx-deployment --all-pods --tail=100
kubectl get events --sort-by=.metadata.creationTimestamp
```

Rollback:

```bash
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment
```

## Production checklist

- Use an explicit namespace (`-n <namespace>` or `metadata.namespace`).
- Pin images to an immutable digest; avoid `latest` and broad mutable tags.
- Set CPU/memory requests and limits.
- Add readiness, liveness, and startup probes.
- Define rolling-update settings, a PodDisruptionBudget, and suitable replicas.
- Run as non-root with a read-only filesystem and dropped Linux capabilities.
- Store secrets in a secret manager; never commit plaintext credentials.
- Add standard ownership/application labels and monitoring.
- Deploy through reviewed GitOps/CI/CD changes; restrict direct production access.

> Note: standard NGINX listens on port `80`. The manifests here use container/target port `8080`; change both to `80` unless the image is configured to listen on `8080`.
