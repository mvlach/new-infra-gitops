# harbor/production

Harbor container registry (+ Trivy scanner) for the `new-infra` cluster.
Applied by ArgoCD (`Application: harbor`, project `infra`).

## Before first commit

```bash
# Generate admin password and record it in new-infra/password
HARBOR_ADMIN=$(openssl rand -base64 24)
echo "Harbor admin: $HARBOR_ADMIN"

# Substitute into values.yaml
sed -i '' "s|REPLACE_WITH_HARBOR_ADMIN_PASSWORD|$HARBOR_ADMIN|" values.yaml
```

## Rotating the admin password

1. Edit `harborAdminPassword` in `values.yaml`.
2. `git commit -m "harbor: rotate admin password"; git push`.
3. ArgoCD syncs, Harbor chart rewrites the `harbor-core` Secret.

Update `new-infra/password` to match.

## Using the registry

```bash
# login — prompted for the password from new-infra/password
docker login harbor.individualstartup.cz -u admin

# publish an image into the default "library" project
docker pull alpine:3.20
docker tag alpine:3.20 harbor.individualstartup.cz/library/alpine:3.20
docker push harbor.individualstartup.cz/library/alpine:3.20
```

Vulnerability scan: open Harbor UI → Projects → library → alpine → Scan.

## Pulling into the cluster

Create an `imagePullSecret` in the target namespace:

```bash
kubectl -n <ns> create secret docker-registry harbor \
  --docker-server=harbor.individualstartup.cz \
  --docker-username=<robot-or-admin> \
  --docker-password=<token-or-password>
```

Reference it from the Pod/Deployment:

```yaml
spec:
  imagePullSecrets:
    - name: harbor
```

Use **robot accounts** (Harbor UI → project → Robot Accounts) rather than the
admin password for CI/CD — they're scoped, revocable, and have long tokens.
