# homelab-komga

Komga manga/comic server. Docker Compose only — no custom code.

## Bring Up / Down

```bash
docker compose up -d
```

## Gotchas

- Library directory must be mounted as a volume (`KOMGA_MEDIA_PATH` env var) and populated by the comic scraper
- Web UI on port `${KOMGA_PORT:-8081}` (host) → internal container port 25600
- Health endpoint: `/api/v1/actuator/health`
- Prometheus metrics: `/api/v1/actuator/prometheus`, requires admin auth — scraped via a dedicated `komga-rpi` job in homelab-monitoring's `prometheus.yml` (basic auth, `prometheus` account), not the generic `homelab.metrics=true` label path other services use
- Container needs up to 60s to initialize before health checks pass
