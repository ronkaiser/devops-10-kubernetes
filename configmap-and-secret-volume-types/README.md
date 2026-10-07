# ConfigMap and Secret volumes

Use a **ConfigMap** for non-sensitive configuration and a **Secret** for sensitive values. Mounting either as a volume presents each key as a file inside the container.

## Files in this example

- `config-file.yaml` — ConfigMap key `mosquitto.conf`; mounted at `/mosquitto/config/mosquitto.conf`.
- `secret-file.yaml` — Secret key `secret.file`; mounted at `/mosquitto/secret/secret.file`.
- `mosquitto.yaml` — Deployment that mounts both volumes.
- `mosquitto-without-volumes.yaml` — same Deployment before externalized configuration.
- `example-secret-certificate.yaml` — format example only; do not apply its placeholder value.

```text
ConfigMap  -> volume -> /mosquitto/config/mosquitto.conf
Secret     -> volume -> /mosquitto/secret/secret.file
```

## Deploy and verify

Create the referenced configuration before the Pod:

```bash
kubectl apply -f config-file.yaml
kubectl apply -f secret-file.yaml
kubectl apply -f mosquitto.yaml
kubectl rollout status deployment/mosquitto --timeout=5m

kubectl get configmap mosquitto-config-file
kubectl get secret mosquitto-secret-file
kubectl exec deployment/mosquitto -- ls -l /mosquitto/config /mosquitto/secret
kubectl exec deployment/mosquitto -- cat /mosquitto/config/mosquitto.conf
```

Troubleshoot a failed mount with:

```bash
kubectl describe pod -l app=mosquitto
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Important behavior

- ConfigMap and Secret volume changes are projected into existing Pods eventually, but the application must reload the file to use new content. Restart or roll out the Deployment when in doubt.
- `data` values in Secrets are Base64-encoded, **not encrypted**. Never commit real credentials or certificates to Git.
- Mounted Secret volumes should always be `readOnly: true`.
- A ConfigMap or Secret must exist in the same namespace as the Pod that mounts it.

## Production checklist

- Use an external secret manager with workload identity; do not store production secrets in repository YAML.
- Restrict access with RBAC and mount only the exact Secret keys needed (`items`).
- Pin the Mosquitto image to an immutable digest, and add resource requests/limits, probes, and security context.
- Use a separate configuration per environment (for example, Kustomize overlays or Helm values); keep manifests declarative in Git.
- Rotate secrets deliberately and roll out workloads after rotation; verify the application can reload safely.
- Store TLS material in a `kubernetes.io/tls` Secret or have cert-manager manage it, rather than using the placeholder certificate example.

> `example-secret-certificate.yaml` is intentionally illustrative: replace the placeholder and keep `type: Opaque` at the Secret top level, not under `metadata`, if you adapt it.
