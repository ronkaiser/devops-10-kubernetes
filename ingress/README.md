# Kubernetes Ingress quick reference

An Ingress routes HTTP(S) traffic by hostname and path to internal Services. It needs an Ingress controller; the YAML object alone does not accept traffic.

```text
client -> DNS -> Ingress controller -> Service -> Pods
```

## Choose and install a controller

- [Ingress controller options](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) — official Kubernetes list of supported and third-party controllers.
- [NGINX Ingress Controller installation with Helm](https://docs.nginx.com/nginx-ingress-controller/install/helm/open-source/) — installation guide for the NGINX controller used in this lecture.

> Kubernetes recommends Gateway API for new capabilities; the Ingress API remains stable but is frozen.

## This manifest

`dashboard-ingress.yaml` routes `dashboard.com/` to the `kubernetes-dashboard` Service on port `80` in the `kubernetes-dashboard` namespace.

- `ingressClassName: nginx` selects the NGINX Ingress controller.
- `host` selects the DNS name.
- `pathType: Prefix` routes `/` and every path below it.
- The backend Service must exist in the same namespace as the Ingress.

## Apply and verify

```bash
# Confirm the target cluster and controller first
kubectl config current-context
kubectl get ingressclass
kubectl get pods -n ingress-nginx

kubectl apply --dry-run=server -f dashboard-ingress.yaml
kubectl diff -f dashboard-ingress.yaml
kubectl apply -f dashboard-ingress.yaml

kubectl get ingress -n kubernetes-dashboard
kubectl describe ingress dashboard-ingress -n kubernetes-dashboard
kubectl get endpoints kubernetes-dashboard -n kubernetes-dashboard
```

## Minikube test

```bash
minikube addons enable ingress
kubectl apply -f dashboard-ingress.yaml
minikube ip

# Test without changing /etc/hosts
curl --resolve dashboard.com:80:$(minikube ip) http://dashboard.com/
```

For browser access, map `dashboard.com` to the Minikube IP in your local hosts file, then open `http://dashboard.com`.

## Add TLS for `dashboard.com`

TLS certificates are stored in a Kubernetes Secret in the same namespace as the Ingress. Create the Secret from an existing certificate and private key:

```bash
kubectl create secret tls dashboard-tls \
  --cert=dashboard.com.crt \
  --key=dashboard.com.key \
  -n kubernetes-dashboard
```

Add this block under `spec:` in `dashboard-ingress.yaml` (alongside `rules:`):

```yaml
tls:
  - hosts:
      - dashboard.com
    secretName: dashboard-tls
```

Then apply and verify it:

```bash
kubectl apply -f dashboard-ingress.yaml
kubectl describe ingress dashboard-ingress -n kubernetes-dashboard
curl -I --resolve dashboard.com:443:127.0.0.1 https://dashboard.com/
```

For local development, use a certificate trusted by your machine. A self-signed certificate enables HTTPS but produces a browser warning. In production, use cert-manager or your organization’s PKI to issue and renew certificates automatically.

## Production checklist

- Run a supported, highly available Ingress controller; monitor its logs, metrics, and capacity.
- Point public DNS to the controller's load balancer—not directly to application Pods or Services.
- Enforce HTTPS with valid certificates (for example, cert-manager) and redirect HTTP to HTTPS.
- Restrict access with authentication, network policies, WAF/rate limits, and controller security settings.
- Set controller resource requests/limits and use replicas plus disruption protection.
- Keep app backends internal (`ClusterIP`); expose only the controller.
- Review routes for host/path conflicts, and test a rollout before changing shared ingress rules.

> Do not publish the Kubernetes Dashboard to the public internet. Keep it private behind VPN/identity-aware access, or use `kubectl port-forward` for temporary administrative access.
