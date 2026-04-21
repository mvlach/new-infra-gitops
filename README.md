# new-infra-gitops

GitOps content for the `new-infra` k3s cluster (Hetzner + ArgoCD). Synced by
ArgoCD; **nothing here should ever be `kubectl apply`-ed manually**.

The bootstrap that registers this repo and creates the initial Applications
lives in the main `individualstartup` working tree at
`new-infra/infrastructure/06-argocd-apps/`.

## Layout

```
new-infra-gitops/
└── logging/
    └── production/
        ├── ns.yaml
        ├── clickhouse-endpoint.yaml    # ConfigMap with the ClickHouse VM IP
        ├── clickhouse-credentials.yaml # Secret with Vector + Grafana-RO passwords
        ├── grafana-admin.yaml          # Secret with Grafana admin login
        ├── vector-rbac.yaml
        ├── vector-config.yaml          # Vector pipeline (sources/transforms/sinks)
        ├── vector-daemonset.yaml
        ├── eventrouter.yaml            # K8s events → stdout → picked up by Vector
        └── grafana.yaml                # Deployment + Service + Ingress + PVC
```

## Preparing for first push

The two Secrets ship with placeholder values. Before the initial commit:

```bash
# Pull the passwords from the ClickHouse bootstrap .envrc
source ../infrastructure/05-clickhouse/.envrc

# Generate a fresh Grafana admin
GRAFANA_ADMIN=$(openssl rand -base64 24)
echo "Grafana admin: $GRAFANA_ADMIN"   # → also write to new-infra/password

# Fill in base64 values
sed -i '' "s|REPLACE_WITH_BASE64_OF_VECTOR_PASSWORD|$(printf '%s' "$TF_VAR_vector_password" | base64)|" \
    logging/production/clickhouse-credentials.yaml
sed -i '' "s|REPLACE_WITH_BASE64_OF_GRAFANA_RO_PASSWORD|$(printf '%s' "$TF_VAR_grafana_ro_password" | base64)|" \
    logging/production/clickhouse-credentials.yaml
sed -i '' "s|REPLACE_WITH_BASE64_OF_GRAFANA_ADMIN_PASSWORD|$(printf '%s' "$GRAFANA_ADMIN" | base64)|" \
    logging/production/grafana-admin.yaml
```

## Pushing

```bash
git init -b main
git remote add origin git@github.com:mvlach/new-infra-gitops.git
git add .
git commit -m "Initial: logging stack (Vector → ClickHouse → Grafana)"
git push -u origin main
```

Then from the main repo:
```bash
cd ../infrastructure/06-argocd-apps
./06-install.sh
```

## Security note

Secrets are committed as **plain** (base64-only) YAML — consistent with the
existing pattern in `mvlach/gitops`. When the `sealed-secrets` controller is
introduced, migrate these to `SealedSecret` CRDs.
