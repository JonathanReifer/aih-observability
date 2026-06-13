# aih-observability

OTEL collector + Loki + Prometheus + Grafana stack for the aih-security suite. Collects
hook execution telemetry from Claude Code sessions and surfaces it in
[aih-conversation-viewer](../aih-conversation-viewer/) as hook timelines, tool decision
history, and per-session API cost.

---

## Services

| Service | Port | Purpose |
|---------|------|---------|
| OTEL Collector | 4317 (gRPC), 4318 (HTTP) | Receives OTLP telemetry from clients |
| Loki | 3100 | Log aggregation — queried by the conversation viewer |
| Prometheus | 9090 | Metrics storage |
| Grafana | 3001 | Dashboards — http://localhost:3001 (admin / aih) |

---

## Local Mode

Run everything on the same machine as Claude Code.

```bash
cd ~/Projects/aih-observability
docker compose up -d
```

Add to `~/.llm-privacy/.env.sh`:
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export LOKI_URL=http://localhost:3100
```

Start the conversation viewer:
```bash
LOKI_URL=http://localhost:3100 bun ~/Projects/aih-conversation-viewer/src/server.ts
```

---

## Remote Mode

Run the stack on a dedicated server; multiple client machines point at it.

**On the server:**
```bash
cd ~/Projects/aih-observability
docker compose up -d
```

Ensure ports 4317, 4318, 3100 are reachable from client machines (firewall/VPN — these
services have no built-in authentication).

**On each client machine** (add to `~/.llm-privacy/.env.sh`):
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://<server-ip>:4317
export LOKI_URL=http://<server-ip>:3100
```

Start the viewer on any client:
```bash
LOKI_URL=http://<server-ip>:3100 bun ~/Projects/aih-conversation-viewer/src/server.ts
```

---

## Configuration

Copy `.env.example` to `.env` to override defaults:

```bash
cp .env.example .env
```

| Variable | Default | Purpose |
|----------|---------|---------|
| `GRAFANA_ADMIN_PASSWORD` | `aih` | Grafana admin login password |

---

## Security Note

The OTEL collector (4317/4318) and Loki (3100) have **no authentication**. In remote mode,
restrict these ports to trusted networks via firewall rules or a VPN. Grafana (3001) has
password auth via `GRAFANA_ADMIN_PASSWORD`.

---

## Data Flow

```
Claude Code session
  → PAI OTEL hook → OTLP gRPC (4317)
      → otel-collector
          → Loki (traces/logs)
          → Prometheus (metrics)

aih-conversation-viewer
  → Loki /loki/api/v1/query_range (3100)
      → hook timelines, tool decisions, session cost
```

OTEL emission from `llm-privacy-proxy` and `llm-privacy-middleware` is planned for a
future release — the collector is ready to receive it when implemented.
