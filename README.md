# Komga

## Purpose
Komga is a free, open-source comic and manga server that organizes and serves your digital comic collection via a modern web interface. It provides a Netflix-like browsing experience for your comics, supports metadata scraping from multiple sources (AniList, ComicVine, etc.), and offers OPDS support for mobile comic reader apps. Deploy this to centralize your comic library and access it anywhere via Tailscale on desktop or mobile devices.

## Documentation
- [Deployment Guide](docs/deployment.md) - Step-by-step setup
- [Troubleshooting](docs/troubleshooting.md) - Common issues and fixes
- [Monitoring Setup](#monitoring-integration) - Health checks and alerts

## Resource Requirements
- **RAM:** 200-300MB idle, 512MB under load (initial scanning/metadata scraping)
- **CPU:** <5% typical, 20-30% during initial library scan
- **Disk:** User's media collection (93GB initial, can grow to 430GB+) + ~100MB for Komga database
- **Network:** Minimal - serves content over LAN/Tailscale, optional outbound for metadata scraping

## Dependencies
- **External:** Optional API access to AniList, MyAnimeList, or ComicVine for metadata scraping
- **Internal:** None - fully self-contained service
- **Database:** Self-contained SQLite database (stored in Docker volume)

## Quick Start

**Prerequisites:**
- Docker and Docker Compose installed
- Media directory prepared (comics/manga organized into folders)
- Port 8081 available (or adjust in .env)

**Deploy:**
```bash
git clone https://github.com/yourusername/homelab-komga.git
cd homelab-komga
cp .env.example .env
nano .env  # Edit KOMGA_MEDIA_PATH and verify other settings
docker compose up -d
```

**Verify:**
```bash
docker ps | grep komga                       # Container running
docker logs komga                            # Check logs for errors
curl http://localhost:8081/api/v1/actuator/health    # Health check responds 200 OK
```

**Access:**
- Local: `http://homelab.local:8081`
- Tailscale: `http://orangepi:8081`
- First-time setup: Create admin account via web interface

## Monitoring Integration

**Health Check:** Built into docker-compose.yml
- Check interval: 30s
- Timeout: 10s
- Retries: 3
- Start period: 60s (Komga needs time to initialize)

**Uptime Kuma:** Add monitor after deployment
- Monitor Type: HTTP(s)
- Check URL: `http://komga:25600/api/v1/actuator/health`
- Heartbeat: 60s
- See [Deployment Guide](docs/deployment.md#uptime-kuma-integration)

**Prometheus Metrics:** Available
- Metrics endpoint: `http://komga:25600/api/v1/actuator/prometheus`
- Add scrape config to `~/code/homelab/monitoring/prometheus/prometheus.yml`
- See [Deployment Guide](docs/deployment.md#prometheus-integration)

**Grafana Dashboard:** Optional
- Monitor library statistics, active sessions, scanning progress
- Custom dashboard can be created from Prometheus metrics

## Updating

**Check for updates:**
```bash
docker compose pull
```

**Apply updates:**
```bash
docker compose down
docker compose up -d
```

**Rollback if broken:**
```bash
docker compose down
# Edit docker-compose.yml, change image tag to previous version
docker compose up -d
```

**Update strategy:** Check for updates monthly, review changelog before applying

## Troubleshooting

Common issues and quick fixes:

**Container won't start:**
```bash
docker logs komga  # Check error messages
# Common causes: Permission issues, port conflicts, invalid media path
```

**High resource usage:**
```bash
docker stats komga  # Monitor real-time usage
# Expected during initial scan or metadata scraping
```

**Library won't scan:**
```bash
# Verify media path in .env matches actual location
# Check container can see files: docker exec komga ls -la /data
```

**More details:** [docs/troubleshooting.md](docs/troubleshooting.md)

## Links
- **Official Documentation:** https://komga.org/
- **Docker Hub:** https://hub.docker.com/r/gotson/komga
- **GitHub Repository:** https://github.com/gotson/komga
- **Community Support:** https://discord.gg/TdRpkDu (Komga Discord)

## License
Komga is licensed under the MIT License. This deployment configuration is provided as-is for homelab use.
