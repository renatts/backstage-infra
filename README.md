# backstage-infra

ArgoCD-managed Kubernetes infrastructure for the Backstage developer portal.
Uses an **ApplicationSet** that deploys environments via **Kustomize overlays** over
the [official Backstage Helm chart](https://github.com/backstage/charts).

Out of the box this repo targets a local **kind** cluster. Add a `prd` list entry
to the ApplicationSet (and a prd overlay) to extend to real clusters.

## Structure

```
backstage-infra/
├── argocd/
│   ├── namespace.yaml              # argocd namespace
│   └── applicationset.yaml         # ApplicationSet: generates backstage-dev
├── kind/
│   └── cluster.yaml                # kind cluster config (ingress port mappings)
└── kustomize/
    ├── base/
    │   ├── values.yaml             # Shared Helm overrides (all envs)
    │   └── namespace/              # Namespace, ResourceQuota, LimitRange, SA, RBAC
    └── overlays/
        ├── dev/                    # kind-targeted overlay
        └── prd/                    # real cluster overlay (Postgres, TLS, PDB)
```

## How it works

```
ApplicationSet (list generator)
  └── backstage-dev → kustomize/overlays/dev/ → base/values.yaml + dev/values.yaml
```

Each overlay uses Kustomize's `helmCharts` to render the `backstage/backstage`
chart (v2.6.3) with layered values — base first, then env-specific overrides.

## Environment differences

| Feature            | dev (kind)                          | prd                        |
|--------------------|-------------------------------------|----------------------------|
| Replicas           | 1                                   | 2                          |
| Image              | `library/backstage:dev` (kind load) | `<registry>/backstage:1.0` |
| Database           | SQLite in-memory                    | External PostgreSQL        |
| Auth               | guest (dev mode)                    | production providers       |
| PDB                | no                                  | yes (minAvailable: 1)      |
| TLS                | no                                  | yes (cert-manager)         |
| Ingress host       | `backstage.127.0.0.1.nip.io`        | `backstage.example.com`    |
| Namespace          | `backstage-dev`                     | `backstage-prd`            |

---

## Local bootstrap on kind

### 1. Create the kind cluster

```bash
kind create cluster --config kind/cluster.yaml
```

### 2. Install ingress-nginx (kind flavour)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### 3. Install Argo CD

```bash
kubectl apply -f argocd/namespace.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --namespace argocd --for=condition=available deployment --all --timeout=180s
```

Enable Kustomize's Helm support (required for `helmCharts:`):

```bash
kubectl -n argocd patch configmap argocd-cm \
  --type merge \
  -p '{"data":{"kustomize.buildOptions":"--enable-helm"}}'
kubectl -n argocd rollout restart deployment argocd-repo-server
```

### 4. Build the Backstage image and load it into kind

From the sibling `backstage-app/` repo:

```bash
cd ../backstage-app
yarn install --immutable
yarn tsc
yarn build:backend
docker build . -f packages/backend/Dockerfile --tag backstage:dev
kind load docker-image backstage:dev --name backstage
```

`kind load` copies the image into the node's containerd; no push to a registry.
Dev overlay pins `registry: docker.io` + `repository: library/backstage` because
that's the canonical name containerd resolves `backstage:dev` to.

### 5. Point the ApplicationSet at your fork and apply

Edit `argocd/applicationset.yaml` → set `spec.template.spec.source.repoURL` to your
git remote. Push, then:

```bash
kubectl apply -f argocd/applicationset.yaml
```

Argo CD picks it up, renders `kustomize/overlays/dev`, and deploys to
`backstage-dev`.

### 6. Reach the portal

```bash
open http://backstage.127.0.0.1.nip.io
```

`127.0.0.1.nip.io` resolves to `127.0.0.1`, which maps through kind's port
mapping (80/443) into the ingress-nginx controller.

### 7. Watch sync status

```bash
kubectl -n argocd get applications
kubectl -n backstage-dev get pods,svc,ingress
```

Argo CD UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Preview locally (no cluster)

```bash
kustomize build --enable-helm kustomize/overlays/dev
kustomize build --enable-helm kustomize/overlays/prd
```

## Rebuilding after backstage-app changes

```bash
cd ../backstage-app
yarn build:backend
docker build . -f packages/backend/Dockerfile --tag backstage:dev
kind load docker-image backstage:dev --name backstage
kubectl -n backstage-dev rollout restart deployment/backstage
```

## Extending to prd

1. Add a Kubernetes cluster secret to Argo CD for the prd cluster.
2. Append a second element to the `list` generator in
   `argocd/applicationset.yaml`:
   ```yaml
   - env: prd
     namespace: backstage-prd
     server: https://<prd-cluster-api-url>
   ```
3. Replace placeholder values in `kustomize/overlays/prd/values.yaml`
   (image repository, ingress host, Postgres secret name).
4. Create the `backstage-postgres-secret` in the prd namespace via
   External Secrets Operator or Sealed Secrets (keys: `host`, `port`,
   `username`, `password`).
