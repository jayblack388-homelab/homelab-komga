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
- Prometheus metrics: deliberately NOT scraped. `/api/v1/actuator/prometheus` requires admin auth, and basic-auth-authenticated requests hit a Komga-side quirk that serves the frontend SPA instead of real API/actuator responses (confirmed on plain endpoints too, not network/credential-related). Deprioritized 2026-08-01 — see homelab-monitoring's `prometheus.yml` for details.
- Container needs up to 60s to initialize before health checks pass
