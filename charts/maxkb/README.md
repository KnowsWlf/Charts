# MaxKB Helm Chart

Production-ready Helm chart for [MaxKB](https://github.com/1Panel-dev/MaxKB) (Max Knowledge Brain) on Kubernetes — an open-source, enterprise-grade agent platform integrating RAG pipelines, agentic workflows, and MCP tool-use.

MaxKB v2 ships as an **all-in-one image** (`1panel/maxkb`) that bundles the application, **PostgreSQL 17 + pgvector**, and **Redis** in a single container. This chart deploys it as a **StatefulSet** with **one replica** backed by a single persistent volume at `/opt/maxkb`. You can optionally point it at an **external PostgreSQL and/or Redis** for production.

## Architecture

```
 Browser
    │
 ┌──┴────────────────────────────────────────────────┐
 │ Kubernetes Cluster                                 │
 │  Ingress (nginx) / Traefik IngressRoute (optional) │
 │  maxkb.example.com → MaxKB :8080                    │
 │                                                    │
 │  ┌──────────────────────────────────────────────┐ │
 │  │            MaxKB (StatefulSet, 1 pod)         │ │
 │  │  ┌────────────┐ ┌────────────┐ ┌───────────┐ │ │
 │  │  │  MaxKB app │ │ PostgreSQL │ │   Redis   │ │ │
 │  │  │  Django    │◀▶│ + pgvector │ │           │ │ │──▶ LLM APIs
 │  │  │  :8080     │ │ :5432      │ │ :6379     │ │ │
 │  │  └────────────┘ └────────────┘ └───────────┘ │ │
 │  │  ┌────────────────────────────────────────┐  │ │
 │  │  │  Persistent Volume  /opt/maxkb          │  │ │
 │  │  │  (PG data, Redis data, logs, model)     │  │ │
 │  │  └────────────────────────────────────────┘  │ │
 │  └──────────────────────────────────────────────┘ │
 └────────────────────────────────────────────────────┘
```

> **Why a StatefulSet, not a Deployment?** The embedded PostgreSQL/Redis own a
> single ReadWriteOnce data directory. A StatefulSet fully terminates the old
> pod (releasing the volume) before starting the replacement during upgrades,
> avoiding the Multi-Attach deadlock a Deployment's rolling update would hit.
> It also encodes that the workload **cannot be scaled horizontally**.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.8+
- A persistent storage provisioner (for the `/opt/maxkb` volume)

## Quick Start

```bash
helm repo add knowswlf https://knowswlf.github.io/Charts/
helm repo update

# All-in-one (embedded PostgreSQL + Redis), access via port-forward
helm install maxkb knowswlf/maxkb
kubectl port-forward svc/maxkb 8080:8080
```

Open <http://localhost:8080> and sign in:

- **Username:** `admin`
- **Password:** `MaxKB@123..`  (change immediately after first login)

## Configuration modes

### 1. All-in-one (default)

Everything embedded in the pod. No external dependencies:

```bash
helm install maxkb knowswlf/maxkb \
  --set persistence.size=20Gi
```

### 2. External PostgreSQL and/or Redis

Point at managed services. The embedded PostgreSQL/Redis are automatically
skipped (the image only starts them when the host is `127.0.0.1`). The external
PostgreSQL **must** have the `vector` (pgvector) extension available.

```yaml
# my-values.yaml
database:
  external:
    enabled: true
    host: postgres.db.svc.cluster.local
    port: 5432
    name: maxkb
    user: maxkb
    password: "change-me"        # or use secrets.existingSecret

redis:
  external:
    enabled: true
    host: redis.cache.svc.cluster.local
    port: 6379
    db: 0
    password: "change-me"        # or use secrets.existingSecret

persistence:
  size: 20Gi
```

```bash
helm install maxkb knowswlf/maxkb -f my-values.yaml
```

To supply passwords from an existing Secret instead of plaintext values, create
a Secret with keys `MAXKB_DB_PASSWORD` and/or `MAXKB_REDIS_PASSWORD` and set
`secrets.existingSecret=<name>`.

## Ingress

The chart supports **both** a standard Kubernetes Ingress (nginx and others) and
a native **Traefik IngressRoute**. Enable only one at a time.

### NGINX (standard Ingress)

MaxKB uploads documents (large request bodies) and streams LLM output (long-lived
connections), so tune the nginx annotations accordingly:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
  tls:
    - secretName: maxkb-tls
      hosts:
        - maxkb.example.com
  hosts:
    - host: maxkb.example.com
      paths:
        - path: /
          pathType: Prefix
```

### Traefik (IngressRoute CRD)

Creates a `Middleware` (HTTP→HTTPS redirect) plus `web`/`websecure` IngressRoutes:

```bash
helm install maxkb knowswlf/maxkb \
  --set traefik.enabled=true \
  --set traefik.host="maxkb.example.com" \
  --set traefik.tlsSecretName="maxkb-tls"
```

## Values

| Key | Description | Default |
|-----|-------------|---------|
| `image.repository` | Image repository | `1panel/maxkb` |
| `image.tag` | Image tag (empty → chart `appVersion`) | `""` |
| `replicaCount` | Replicas (**must be 1**) | `1` |
| `updateStrategy.type` | StatefulSet update strategy | `RollingUpdate` |
| `env` | Extra environment variables (map) | `{}` |
| `resources` | Resource requests/limits | `1Gi`/`500m` → `4Gi`/`2` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `8080` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Volume size | `10Gi` |
| `persistence.storageClass` | StorageClass (empty → default) | `""` |
| `persistence.existingClaim` | Use an existing PVC | `""` |
| `persistence.mountPath` | Data mount path | `/opt/maxkb` |
| `database.external.enabled` | Use external PostgreSQL | `false` |
| `database.external.host/port/name/user/password` | External PG connection | — |
| `database.external.maxOverflow` | Max overflow connections | `80` |
| `redis.external.enabled` | Use external Redis | `false` |
| `redis.external.host/port/db/password` | External Redis connection | — |
| `secrets.create` | Create a Secret for external passwords | `true` |
| `secrets.existingSecret` | Use an existing Secret | `""` |
| `ingress.enabled` | Standard Ingress (nginx/…) | `false` |
| `ingress.className` | Ingress class (`nginx`, `traefik`, …) | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `traefik.enabled` | Traefik IngressRoute | `false` |
| `traefik.host` | Traefik host | `maxkb.example.com` |
| `traefik.tlsSecretName` | TLS secret for Traefik | `""` |

See [`values.yaml`](values.yaml) for the full, commented list.

## Persistence & upgrades

- The PVC is created with `helm.sh/resource-policy: keep`, so it **survives
  `helm uninstall`**. Delete it manually if you want to wipe the data.
- Do not run more than one replica against the same volume — the embedded
  database would be corrupted.
- Upgrading from MaxKB v1 to v2 is **not supported** by the image; this chart
  targets v2 only.

## Uninstall

```bash
helm uninstall maxkb
# The data PVC is retained; remove it explicitly if desired:
kubectl delete pvc maxkb-data
```

## License

MIT (chart). MaxKB itself is licensed under GPL-3.0 — see the [upstream repository](https://github.com/1Panel-dev/MaxKB).
