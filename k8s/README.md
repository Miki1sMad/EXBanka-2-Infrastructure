# EXBanka-2 — Kubernetes deploy

Manifests target a single namespace (`exbanka-2`). All images pulled from
GHCR (built by the per-repo CI/CD).

## Prerequisites on the cluster

1. **Crunchy Postgres Operator** installed (cluster-wide or in `exbanka-2`).
2. **Nginx Ingress Controller** (or adjust `ingressClassName` in `60-ingress.yaml`).
3. **cert-manager** (or pre-created `exbanka-tls` Secret) for HTTPS.
4. **GHCR pull secret** in `exbanka-2`:
   ```sh
   kubectl create secret docker-registry ghcr-pull -n exbanka-2 \
     --docker-server=ghcr.io \
     --docker-username=<your-gh-user> \
     --docker-password=<gh-pat-read:packages>
   ```

## Apply order

```sh
# 1) Namespace
kubectl apply -f 00-namespace.yaml

# 2) Postgres cluster (operator must already be installed)
kubectl apply -f 10-postgres-cluster.yaml
# Wait until the operator creates `db-pguser-bank_admin` Secret and a
# `db-primary` Service before proceeding.

# 3) Infrastructure
kubectl apply -f 11-rabbitmq.yaml -f 12-redis.yaml

# 4) Config + secrets (copy 21-secrets.example.yaml → 21-secrets.yaml first)
kubectl apply -f 20-config.yaml -f 21-secrets.yaml

# 5) Migration ConfigMaps (run from EXBanka-2-Backend repo root)
kubectl create configmap user-migrations -n exbanka-2 \
  --from-file=services/user-service/internal/database/migrations/
kubectl create configmap bank-migrations -n exbanka-2 \
  --from-file=services/bank-service/internal/database/migrations/

# 6) Migration Jobs
kubectl apply -f 30-migrations-user.yaml -f 31-migrations-bank.yaml
kubectl wait --for=condition=complete -n exbanka-2 \
  job/migrate-user-service job/migrate-bank-service --timeout=5m

# 7) Backend + frontend
kubectl apply -f 40-user-service.yaml -f 41-bank-service.yaml \
              -f 42-notification-service.yaml -f 50-frontend.yaml

# 8) Ingress
sed -i 's/DOMEN_PLACEHOLDER/your.real.domain/g' 60-ingress.yaml 20-config.yaml
kubectl apply -f 60-ingress.yaml
kubectl rollout restart deploy/frontend -n exbanka-2   # picks up new API_BASE_URL if changed

# 9) Observability (MLA bonus) — Prometheus + Alertmanager + Grafana
kubectl create secret generic alertmanager-discord -n exbanka-2 \
  --from-literal=webhook_url="$(cat ../alertmanager/discord_webhook_url)"
kubectl create secret generic grafana-admin -n exbanka-2 \
  --from-literal=user=admin --from-literal=password='CHANGE_ME'
kubectl create configmap grafana-dashboards -n exbanka-2 \
  --from-file=../grafana/dashboards/exbanka-red.json
kubectl apply -f 70-prometheus.yaml -f 71-alertmanager.yaml -f 72-grafana.yaml
```

## Change backend addresses without rebuild

All backend services read DB, RabbitMQ, Redis, cross-service, and external
API addresses from ENV (`config.go` in each service). Edit `20-config.yaml`
or `21-secrets.yaml`, re-apply, then:

```sh
kubectl rollout restart deploy/user-service deploy/bank-service \
  deploy/notification-service -n exbanka-2
```

## Change frontend API URL / protocol without rebuild

Edit `API_BASE_URL` in `20-config.yaml` and roll the frontend:

```sh
kubectl rollout restart deploy/frontend -n exbanka-2
```

`docker-entrypoint.sh` regenerates `/usr/share/nginx/html/config.js` on every
pod start; browsers fetch the new value with the next page load.

## Ingress routing modes

See header comment in `60-ingress.yaml`. Default is Option A (frontend nginx
proxies). Switch to Option B if the professor requires literal
`/user-service/...` and `/bank-service/...` in the URL.
