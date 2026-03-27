# CLAUDE.md — DevOps Dashboard Backend (ops.jcrlabs.net)

> Extiende: `SHARED-CLAUDE.md` (sección Go + Hexagonal)

## Project Overview

Agregador de observabilidad: K8s API + Prometheus + GitHub Actions. Push en tiempo real via SSE. Histórico de métricas en PostgreSQL. Cache en Redis.

## Tech Stack

- **Language**: Go 1.26
- **K8s**: `k8s.io/client-go` (in-cluster + fallback kubeconfig)
- **DB**: PostgreSQL 17 con `github.com/jackc/pgx/v5` (pool)
- **Cache**: Redis `github.com/redis/go-redis/v9`
- **GitHub**: `github.com/google/go-github/v68` (REST v3 — más simple que GraphQL para Actions)
- **SSE**: stdlib `net/http` con `text/event-stream`
- **Logging**: `log/slog`
- **Metrics**: `github.com/prometheus/client_golang`

## Architecture (Hexagonal)

```
devops-dashboard-back/
├── cmd/server/main.go
├── internal/
│   ├── domain/
│   │   ├── cluster.go               # Node, Pod, Deployment — pure structs
│   │   ├── pipeline.go              # WorkflowRun, Job — pure structs
│   │   ├── metric.go                # MetricSnapshot — pure struct
│   │   └── alert.go                 # Alert + Severity enum
│   │
│   ├── app/
│   │   ├── cluster_service.go       # Lógica cluster state
│   │   │   // port: ClusterReader interface (List nodes, pods, deployments)
│   │   ├── pipeline_service.go      # Lógica pipelines
│   │   │   // port: PipelineReader interface (List workflow runs)
│   │   ├── alert_service.go         # Evalúa reglas → genera alertas
│   │   │   // port: AlertNotifier interface (opcional: futuro webhook)
│   │   └── metric_service.go        # Persiste snapshots
│   │       // port: MetricStore interface (Save, Query by timerange)
│   │
│   ├── adapter/
│   │   ├── inbound/
│   │   │   ├── http/
│   │   │   │   ├── server.go
│   │   │   │   ├── cluster_handler.go
│   │   │   │   ├── pipeline_handler.go
│   │   │   │   ├── alert_handler.go
│   │   │   │   └── sse_handler.go       # SSE stream: register/heartbeat/push
│   │   │   └── collector/               # Goroutines que alimentan el sistema
│   │   │       ├── k8s_collector.go     # Ticker 15s → ClusterReader → broadcast
│   │   │       ├── github_collector.go  # Ticker 60s → PipelineReader → broadcast
│   │   │       └── manager.go          # Arranca/para collectors con context
│   │   │
│   │   └── outbound/
│   │       ├── kubernetes/
│   │       │   └── client.go            # Implements ClusterReader via client-go
│   │       ├── github/
│   │       │   └── client.go            # Implements PipelineReader via go-github
│   │       ├── postgres/
│   │       │   └── metric_repo.go       # Implements MetricStore via pgx
│   │       └── redis/
│   │           └── cache.go             # Cache wrapper con TTL
│   │
│   ├── sse/
│   │   └── hub.go                       # Broker: register/unregister/broadcast channels
│   │
│   ├── middleware/                       # Standard chain (ver shared principles)
│   └── config/
│
├── migrations/                          # SQL up/down
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml                 # read-only: pods, nodes, deployments, ingresses
│   └── clusterrolebinding.yaml
├── .golangci.yml
├── Makefile
└── Dockerfile
```

## Collectors pattern

```go
// internal/adapter/inbound/collector/manager.go
type Manager struct {
    collectors []Collector
    hub        *sse.Hub
    logger     *slog.Logger
}

type Collector interface {
    Run(ctx context.Context) error  // blocks until ctx.Done
    Name() string
}

func (m *Manager) Start(ctx context.Context) {
    for _, c := range m.collectors {
        go func(c Collector) {
            if err := c.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
                m.logger.Error("collector stopped", "name", c.Name(), "error", err)
            }
        }(c)
    }
}
```

## SSE Hub pattern

```go
// internal/sse/hub.go
type Hub struct {
    clients    map[chan []byte]struct{}
    register   chan chan []byte
    unregister chan chan []byte
    broadcast  chan []byte
}

func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case client := <-h.register:
            h.clients[client] = struct{}{}
        case client := <-h.unregister:
            delete(h.clients, client)
            close(client)
        case msg := <-h.broadcast:
            for client := range h.clients {
                select {
                case client <- msg:
                default:
                    delete(h.clients, client)
                    close(client)
                }
            }
        }
    }
}
```

## RBAC K8s (ClusterRole)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: devops-dashboard-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "services", "namespaces", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates"]
    verbs: ["get", "list"]
```

## Deploy

- **Dominio**: `ops.jcrlabs.net`
- **Namespace**: `monitoring`
- **ServiceAccount** con ClusterRoleBinding read-only
- **Redis**: sidecar o servicio compartido
- **PostgreSQL**: PVC 5Gi (métricas históricas)

## CI local

Ejecutar **antes de cada commit** para evitar que lleguen errores a GitHub Actions:

```bash
gofmt -l .                      # no debe mostrar ficheros
go vet ./...
golangci-lint run --timeout=5m
go test -race ./...
```
## Git

- Ramas: `feature/`, `bugfix/`, `hotfix/`, `release/` — sin prefijos adicionales
- Commits: convencional (`feat:`, `fix:`, `chore:`, etc.) — sin mencionar herramientas externas ni agentes en el mensaje
- PRs: título y descripción propios del cambio — sin mencionar herramientas externas ni agentes
- Comentarios y documentación: redactar en primera persona del equipo — sin atribuir autoría a herramientas
