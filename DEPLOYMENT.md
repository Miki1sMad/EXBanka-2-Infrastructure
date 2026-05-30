# EXBanka-2 — Lista komponenti i servisa za Kubernetes deployment

**Namespace:** `exbanka-2`
**Domain:** TBD (čekamo)

## Aplikativni servisi (Go)

| Servis                 | Port HTTP | Port gRPC | Slika (GHCR)                                                         |
| ---------------------- | --------- | --------- | -------------------------------------------------------------------- |
| `user-service`         | 8080      | 50051     | `ghcr.io/raf-si-2025/exbanka-2-backend-user-service:latest`         |
| `bank-service`         | 8080      | 50051     | `ghcr.io/raf-si-2025/exbanka-2-backend-bank-service:latest`         |
| `notification-service` | 8083 (metrics) | 50053 | `ghcr.io/raf-si-2025/exbanka-2-backend-notification-service:latest` |

Svi servisi izlažu `/healthz` (liveness/readiness) i `/metrics` (Prometheus).

## Frontend

| Servis     | Port | Slika                                                |
| ---------- | ---- | ---------------------------------------------------- |
| `frontend` | 80   | `ghcr.io/raf-si-2025/exbanka-2-frontend:latest`     |

Nginx servira statički Vite bundle. `docker-entrypoint.sh` generiše
`/config.js` iz ENV-a pri startu pod-a — promena `API_BASE_URL` i
protokola se radi bez rebuilda (samo `kubectl rollout restart`).

## Infrastruktura

| Komponenta | Tehnologija | Replikacija | Skladište |
| ---------- | ----------- | ----------- | --------- |
| Baza       | PostgreSQL 17 (Crunchy Operator) | 3 instance + pgBouncer x2 | 30 GiB + 30 GiB pgBackRest |
| Broker     | RabbitMQ 3 (management plugin)   | 1                          | 2 GiB |
| Cache      | Redis 7 (in-memory, no persist)  | 1                          | — |

### Postgres — jedna baza, dve sheme

| Baza      | Shema           | Servis vlasnik         |
| --------- | --------------- | ---------------------- |
| `bank_db` | `user_service`  | user-service           |
| `bank_db` | `core_banking`  | bank-service           |

Migracije se izvršavaju kao K8s `Job`-ovi (jedan po servisu), sa
`search_path=<schema>` u DSN-u i posebnim `schema_migrations_user` /
`schema_migrations_bank` tabelama za praćenje verzija.

Replikacija je out-of-the-box: Crunchy operator pravi sinhronu replikaciju
preko `instance1.replicas: 3` (1 primary + 2 standby).

## Eksterni servisi (preko interneta)

| Servis              | Svrha                                  | ENV ključ                              |
| ------------------- | -------------------------------------- | -------------------------------------- |
| Gmail SMTP          | Email (activation/reset/OTP)           | `SMTP_USER`, `SMTP_PASS`              |
| ExchangeRate-API    | Live FX kursevi                        | `EXCHANGE_RATE_API_KEY` (opciono)     |
| EODHD               | Real-time stock/forex quotes           | `EODHD_API_KEY` (opciono)             |
| Finnhub             | Backup market data                     | `FINNHUB_API_KEY` (opciono)           |
| AlphaVantage        | Company overview + FX                  | `ALPHAVANTAGE_API_KEY` (opciono)      |
| Firebase Cloud Msg  | Push notifikacije (mobile)             | `FCM_SERVER_KEY` (opciono)            |
| Druga banka (peer)  | Interbank protokol (si-tx-proto)       | `INTERBANK_PEER_BASE_URL`, `INTERBANK_PEER_API_KEY` |

## Ingress

Jedna `Ingress` ruta (`exbanka` u `60-ingress.yaml`). TLS terminacija na
Ingress kontroleru; intra-cluster saobraćaj ide HTTP-om preko ClusterIP
servisa.

**Default routing (Option A):** sve ide na `frontend` servis. Frontend-ov
nginx interno proxy-uje `/api/*` na `user-service:8080` ili
`bank-service:8080`. URL u browseru: `https://DOMEN/...`.

**Alternativa (Option B):** Ingress direktno rutira `/user-service/*` i
`/bank-service/*` na backend servise. URL: `https://DOMEN/user-service/...`.
Manifest je pripremljen, ali zakomentarisan.

## Bez rebuilda — promene adresa

- **Backend**: sve adrese (DB, RabbitMQ, Redis, cross-service, eksterni
  API-ji) i kredencijali čitaju se iz ENV-a u `config.go`. Izmena u
  `20-config.yaml` ili `21-secrets.yaml` + `kubectl rollout restart`.
- **Frontend**: `API_BASE_URL` i protokol čitaju se iz `window.__APP_CONFIG__`,
  koji ulazi iz `/config.js`, koji se generiše iz ENV-a pri startu pod-a.
  Izmena u `20-config.yaml` + `kubectl rollout restart deploy/frontend`.

## Observability (MLA bonus — koef. 0.033)

| Komponenta    | Slika                       | Svrha                                          |
| ------------- | --------------------------- | ---------------------------------------------- |
| Prometheus    | `prom/prometheus:v2.55.1`   | scrape `/metrics` na 3 backend servisa         |
| Alertmanager  | `prom/alertmanager:v0.28.0` | rute alerta na Discord webhook                 |
| Grafana       | `grafana/grafana:11.3.0`    | dashboard `exbanka-red.json` (RED metrike)     |

Alerts implementirani (`70-prometheus.yaml`):
- `ServiceDown` (severity: critical) — target nedostupan >1 min
- `HighGRPCErrorRate` / `HighGRPCLatencyP99` (warning) — 5% error / p99 >1s
- `HighHTTPErrorRate` (warning) — 5% HTTP 5xx

Discord webhook URL = Secret `alertmanager-discord` (NIKAD u Git).

## Šta nije pokriveno (otvorena pitanja)

- Domain (`DOMEN_PLACEHOLDER` u manifestima — zamena posle dodele).
- TLS cert: cert-manager + Let's Encrypt vs. pre-created Secret.
- Da li profesor traži literalno `/user-service/` u URL-u? Ako da: prebaciti
  Ingress na Option B (komentar u `60-ingress.yaml`).
- Observability stack (Prometheus + Grafana) — postoji u `EXBanka-2-Infrastructure/`
  za lokalni docker-compose; treba poseban skup manifesta za K8s ako se traži.
