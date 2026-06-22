# Airbnb Clone — Kubernetes Manifests (GitOps)

## The problem

Running a 15-service platform by hand doesn't scale. Manual `kubectl apply` is error-prone and
non-repeatable, live environments **drift** from what's in source control, there's **no audit
trail** of who changed what, and a lost node means manual recovery. How do you deploy and operate
many moving parts **reliably, repeatably, and with a full history** — without babysitting the cluster?

## The approach — GitOps

This repository is the **single source of truth** for the cluster. ArgoCD continuously reconciles
the live cluster to match what's committed here, which turns the problems above into guarantees:

- **Every change is a reviewed git commit** — auditable and revertable, no out-of-band `kubectl`.
- **No drift** — ArgoCD self-heals the cluster back to the declared state automatically.
- **Repeatable & hands-off** — the same commit produces the same cluster, every time.
- **Resilient** — a failed workload is reconciled back without manual intervention.

> **Note:** credentials in this public repo are redacted to `CHANGE_ME`. Real values are
> injected at runtime via **Bitnami Sealed Secrets** (encrypted in-repo) or rotated out of band.

---

## Challenges I faced (and how I solved them)

Running 15+ services on a self-managed k3s cluster meant most of this did **not** work first try.
The real engineering was here — each item below is a problem I actually hit and the fix that
resolved it (traceable in the commit history):

- **Spring Boot pods stuck in a restart loop.** `httpGet` actuator probes fired before the JVM had
  finished booting, so Kubernetes kept killing healthy-but-slow pods. Switched the Java services'
  liveness/readiness to `tcpSocket` probes and tuned the timings — slow startup no longer triggers
  a kill loop.
- **The Gateway API's hard 16-rule cap on a single `HTTPRoute`.** The platform grew past 15 services
  and routing silently stopped accepting new rules; one path (`/api/v1/experiences`) was returning
  **403** because it was unrouted. Split routing into a second `HTTPRoute` that merges on the same
  gateway/host, instead of collapsing path prefixes and losing clarity.
- **Neo4j crashing on config validation.** Kubernetes' default `enableServiceLinks` injects
  `NEO4J_*` env vars for every service, which Neo4j parses as its own (invalid) config. Disabling
  `enableServiceLinks` on the deployment stopped the injected vars from breaking startup.
- **The data-seeder kept getting OOMKilled** on large CSV imports. Raised its memory limit
  (→ 768Mi → 1Gi) and loosened the liveness probe period/threshold so a long seed run isn't
  mistaken for a hang.
- **Sealed secrets that silently didn't match.** Recreating the cluster rotated the sealing key,
  invalidating every existing `SealedSecret`; and new services were sealed against a placeholder
  that didn't match the real JWT issuer, so auth failed at runtime. Fix: reseal all secrets against
  the **current** cluster certificate and share the one real auth JWT secret across services.
- **`ImagePullBackOff` on private images.** `imagePullSecrets` was placed incorrectly across
  deployments and the GHCR token expired. Corrected the secret wiring everywhere and set up token
  refresh.
- **ArgoCD fighting itself.** A `Job`'s immutable `spec.template` blocked re-sync (fixed with
  `Replace=true`), and stray finalizers / `ServerSideApply` produced permanent diffs. Cleaned up the
  Application (directory recurse, removed invalid fields) so sync stays green.

---

## Architecture

```
                              Cloudflare (TLS, WAF, DNS)
                                       │
                              airbb.serghini.me
                                       │
                        ┌──────────────▼───────────────┐
                        │   NGINX Gateway Fabric        │   Gateway API
                        │   (HTTPRoute path routing)    │   /api/v1/* · /ws · /
                        └──────────────┬───────────────┘
                                       │
   ┌───────────────────────────────────────────────────────────────────────────┐
   │  airbnb namespace (k3s)                                                     │
   │                                                                             │
   │  frontend (Next.js 15)                                                      │
   │                                                                             │
   │  Spring Boot microservices ────────────────────────────────────────────────│
   │   auth · listing · booking · payment · chat · review · suggestion           │
   │   notification · wishlist · availability · recently-viewed · guidebook      │
   │   coupon · qa · concierge(AI)                                               │
   │                                                                             │
   │  Datastores:  PostgreSQL · MongoDB · Neo4j · Redis · Elasticsearch           │
   │  Eventing:    RabbitMQ (topic exchange `airbnb.events`)                      │
   └───────────────────────────────────────────────────────────────────────────┘
```

**Database-per-service.** Services talk to each other synchronously over REST and
asynchronously via RabbitMQ events — never by sharing a database.

---

## Services

| Service | Port | Datastore(s) | Responsibility |
|---------|------|--------------|----------------|
| `frontend` | 3000 | — | Next.js 15 App Router UI (SSR) |
| `service-auth` | 8081 | PostgreSQL, Redis | Accounts, JWT issuing |
| `service-listing` | 8082 | MongoDB, Redis, Elasticsearch | Listings + experiences, search |
| `service-booking` | 8083 | PostgreSQL, Redis | Bookings, availability |
| `service-payment` | 8084 | PostgreSQL, Stripe | Payment intents (Stripe test mode) |
| `service-chat` | 8085 | MongoDB, Redis | Messaging (WebSocket `/ws`) |
| `service-review` | 8086 | PostgreSQL, Redis | Reviews + rating stats |
| `service-suggestion` | 8087 | Neo4j, Redis | Graph-based recommendations |
| `service-notification` | 8088 | MongoDB, RabbitMQ | In-app notifications (event consumer) |
| `service-wishlist` | 8089 | MongoDB | Saved listings |
| `service-availability` | 8090 | MongoDB | Host calendar / date blocking |
| `service-recently-viewed` | 8091 | MongoDB | "Continue exploring" history |
| `service-guidebook` | 8092 | MongoDB | Host's local tips |
| `service-coupon` | 8093 | MongoDB | Promo codes at checkout |
| `service-qa` | 8094 | MongoDB | Listing questions & answers |
| `service-concierge` | 8095 | MongoDB, Gemini | AI trip-planning chatbot (function-calling) |
| `data-seeder` | 8086 | — | One-shot demo-data seeder (Flask) |

---

## GitOps flow

```
  per-service repo (develop)                this repo (infra branch)         cluster
  ┌────────────────────────┐  CI updates    ┌───────────────────────┐  sync   ┌─────────┐
  │ push → GitHub Actions  │  image tag in  │ apps/<svc>/            │ ◀────── │ ArgoCD  │
  │ build · test · scan    │ ─────────────▶ │   deployment.yaml ...  │ ──────▶ │ (auto,  │
  │ push image → GHCR      │                │ gateway-api/ · ...     │ prune,  │ selfHeal│
  └────────────────────────┘                └───────────────────────┘ recurse)└─────────┘
```

1. Each service lives in its own repo; CI builds, tests, scans (Semgrep + Trivy), and pushes an image to **GHCR**.
2. CI then bumps the image tag in **this** repo (`apps/<service>/deployment.yaml`).
3. **ArgoCD** (app `airbnb`, auto-sync + prune + self-heal, recursing from the repo root) reconciles the cluster.

---

## Repository layout

```
k8s-manifests/                      (branch: infra — ArgoCD's target revision)
├── namespaces/                     airbnb namespace
├── databases/                      PostgreSQL · MongoDB · Neo4j · Redis · Elasticsearch · RabbitMQ
│   └── mongodb/grant-new-dbs-job.yaml   ArgoCD sync-hook job (per-DB grants)
├── apps/<service>/                 per service: configmap · sealed-secret · deployment · service · hpa · pdb
├── gateway-api/
│   ├── gateway.yml                 NGINX Gateway Fabric Gateway + listeners
│   ├── airbnb-route.yml            primary HTTPRoute (capped at 16 rules)
│   ├── airbnb-route-extra.yml      overflow HTTPRoute for newer services
│   └── policies.yml                ClientSettingsPolicy (body size, etc.)
├── argocd-apps/airbnb-app.yaml     the ArgoCD Application (watches this repo)
└── pub-cert.pem                    Sealed Secrets public cert (for kubeseal)
```

## Key platform decisions

- **Networking** — [NGINX Gateway Fabric](https://docs.nginx.com/nginx-gateway-fabric/) (Gateway API). A single `HTTPRoute` is capped at **16 rules**, so newer services' routes live in `airbnb-route-extra.yml` (rules merge across routes on the same gateway/host).
- **Secrets** — [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets): secrets are sealed with `kubeseal` (`pub-cert.pem`) and committed **encrypted**; the controller decrypts them in-cluster. Configmaps never hold credentials.
- **Scaling / resilience** — CPU-based **HPAs** (min 1, target 60%) and **PodDisruptionBudgets** on the services.
- **Runtime security** — **Falco** for runtime threat detection (installed via Ansible/Helm).
- **Provisioning** — cluster + addons are bootstrapped from a separate `infra` repo (Terraform + Ansible: k3s, ArgoCD, cert-manager, sealed-secrets, gateway, Falco).
