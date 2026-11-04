# Helm demo — MongoDB on a managed Kubernetes cluster

Helm installs Kubernetes applications from **charts** (templates + default values). Each installation is a **release**; a values file customizes it. Helm runs through your current Kubernetes context. Helm 3 and 4 do not need Tiller.

Lectures: [Helm basics](https://techworld-with-nana.teachable.com/courses/1108792/lectures/28706437) and [Helm Demo — Managed K8s cluster](https://techworld-with-nana.teachable.com/courses/1108792/lectures/28706435). The demo page was login-locked when checked on November 4, 2026; these instructions are aligned with the committed files, not a verified transcript of the video.

## Your implementation (Git history)

| Commit | What the documentation follows |
| --- | --- |
| `0f1cda4` | MongoDB replica set through Helm; Mongo Express Deployment/Service and Ingress through kubectl. Linode storage, `bitnamisecure/mongodb:latest`, metrics disabled. |
| `1704090` | Ignore local kubeconfig YAML files; other Secret/values YAML files are not covered by those ignore patterns. |
| `8379d67` | Ingress class changed from `nginx-ingress` to `nginx`; host set to `172-104-244-224.ip.linodeusercontent.com`. |

These are confirmed repository choices; differences from the video cannot be verified while it is locked. The `helm-demo` namespace below is a suggested isolated namespace, not evidence of where you previously installed the release. For an existing deployment, find its namespace with `helm list -A` and use that namespace throughout instead of creating a second installation.

## Connect to your managed cluster

Run from this folder. If using the local kubeconfig, point this terminal at it without merging or changing your global configuration:

```bash
export KUBECONFIG="$PWD/test-kubeconfig.yaml"
kubectl config current-context
kubectl get nodes
helm list -A
kubectl get storageclass
kubectl get ingressclass
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

Verify the target cluster before deploying. The committed values use `linode-block-storage`; retain it on LKE if available. A reachable Ingress controller that handles class `nginx` is also required; an Ingress manifest alone does not install one. Unlike the local Minikube workflow, a managed controller's cloud LoadBalancer normally provides external access without `minikube tunnel`.

## Discover and inspect a chart

```bash
helm version
kubectl config current-context
kubectl get storageclass

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo bitnami/mongodb --versions
```

Choose a chart version, then use it consistently:

```bash
CHART_VERSION='<chosen-chart-version>'
helm show chart bitnami/mongodb --version "$CHART_VERSION"
helm show values bitnami/mongodb --version "$CHART_VERSION"
helm show readme bitnami/mongodb --version "$CHART_VERSION"
```

Chart version and application/image version are different. Charts can also be installed directly from OCI registries, e.g. `oci://registry-1.docker.io/bitnamicharts/mongodb`.

## Deploy this demo

Run from this folder. Before installing, review `helm-mongodb.yaml`:

- `architecture: replicaset` and `replicaCount: 3` request three MongoDB members; check the chart's arbiter setting too.
- Replace `linode-block-storage` with a StorageClass available in your cluster (commonly `standard` on Minikube).
- The plaintext password is for learning. For work, use the chart's `auth.existingSecret` and provision credentials separately.
- Check access to `bitnamisecure/mongodb` and select a compatible, pinned image tag/digest. The current `latest` and `allowInsecureImages: true` override are demo settings, not production defaults; the override does not grant image registry access.

```bash
# Preview resources locally; output may contain credentials. Do not share it.
helm template mongodb bitnami/mongodb \
  --version "$CHART_VERSION" -n helm-demo -f helm-mongodb.yaml

# Install, or update the existing release with the same command.
helm upgrade --install mongodb bitnami/mongodb \
  --version "$CHART_VERSION" \
  -n helm-demo --create-namespace \
  -f helm-mongodb.yaml --wait --timeout 10m

helm status mongodb -n helm-demo
kubectl get pods,svc,pvc -n helm-demo

# Mongo Express is a separate kubectl deployment, not part of the Helm release.
kubectl apply -n helm-demo -f helm-mongo-express.yaml
kubectl rollout status deployment/mongo-express -n helm-demo --timeout=5m
kubectl port-forward -n helm-demo svc/mongo-express-service 8081:8081
```

Open [http://localhost:8081](http://localhost:8081) while port-forward runs. The Mongo Express manifest assumes release name `mongodb`, Secret `mongodb` with key `mongodb-root-password`, and hostname `mongodb-0.mongodb-headless`. Verify these against your chart version. For resilient replica-set connections, configure all members and the replica-set name rather than relying on member 0 being primary.

## Access through your Ingress

`helm-ingress.yaml` already uses class `nginx` and host `172-104-244-224.ip.linodeusercontent.com` from commit `8379d67`. Verify that this hostname resolves to your current Ingress controller's external address; update it if the cloud address changes. Keep MongoDB and Mongo Express in the same namespace because the Secret and database hostname references are namespace-local.

```bash
kubectl apply -n helm-demo -f helm-ingress.yaml
kubectl get ingress mongo-express -n helm-demo
kubectl describe ingress mongo-express -n helm-demo
kubectl get endpointslice -n helm-demo \
  -l kubernetes.io/service-name=mongo-express-service
curl -I http://172-104-244-224.ip.linodeusercontent.com

# Isolate DNS problems: substitute the actual controller LoadBalancer IP.
curl -I --resolve 172-104-244-224.ip.linodeusercontent.com:80:<controller-ip> \
  http://172-104-244-224.ip.linodeusercontent.com
```

Open [the demo over HTTP](http://172-104-244-224.ip.linodeusercontent.com). The committed Ingress has no TLS configuration; forced HTTPS will not test the same route. A `401` may indicate Mongo Express web authentication; database root credentials and web login credentials are separate. A `404` suggests a host/class routing mismatch; a `503` commonly indicates no ready backend endpoints. If port-forward works but Ingress fails, check the controller, LoadBalancer address, hostname and firewall. Protect Mongo Express with authentication and TLS before exposing it at work.

## Inspect, troubleshoot, and roll back

```bash
helm list -A
helm get values mongodb -n helm-demo          # Your overrides
helm get values mongodb -n helm-demo --all    # Computed values
helm get manifest mongodb -n helm-demo       # May include Secrets
helm history mongodb -n helm-demo

kubectl describe pod <pod-name> -n helm-demo
kubectl logs <pod-name> -n helm-demo --all-containers --tail=100
kubectl get events -n helm-demo --sort-by=.metadata.creationTimestamp

# Choose a successful revision from helm history.
helm rollback mongodb <revision> -n helm-demo --wait --timeout 10m
```

`Pending` PVCs usually need a working StorageClass/provisioner; `ImagePullBackOff` needs an accessible image and any required registry credentials. Rollback restores release configuration; it does not restore database data or reverse schema changes. Back up MongoDB before upgrades, and follow the chart's upgrade/password-rotation instructions.

## Remove the demo

```bash
kubectl delete -n helm-demo -f helm-ingress.yaml --ignore-not-found
kubectl delete -n helm-demo -f helm-mongo-express.yaml --ignore-not-found
helm uninstall mongodb -n helm-demo
kubectl get pvc -n helm-demo
```

StatefulSet PVCs are commonly retained. Review them before any explicit deletion; removing storage can permanently delete database data.

References: [Helm upgrade](https://helm.sh/docs/helm/helm_upgrade/), [Bitnami MongoDB chart and values](https://github.com/bitnami/charts/tree/main/bitnami/mongodb).
