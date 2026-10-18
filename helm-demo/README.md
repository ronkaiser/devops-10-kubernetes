# Helm command reference

Helm installs Kubernetes applications from **charts** (templates + default values). Each installation is a **release**; a values file customizes it. Helm runs through your current Kubernetes context. Helm 3 and 4 do not need Tiller.

Source: [Nana's Helm lecture](https://techworld-with-nana.teachable.com/courses/1108792/lectures/28706437). The lecture required login when checked; these notes use the local demo files and official documentation.

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

Optional Ingress: replace `YOUR_HOST_DNS_NAME` and verify `ingressClassName` using `kubectl get ingressclass`, then run `kubectl apply -n helm-demo -f helm-ingress.yaml`. Configure DNS/hosts and controller reachability as described in the neighboring Ingress lesson. Protect Mongo Express with authentication and TLS before exposing it.

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
