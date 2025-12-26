# Deployment Guide

## Prerequisites

**System Requirements:**
- [ ] Docker Engine 24.0+ installed
- [ ] Docker Compose v2.20+ installed
- [ ] Tailscale configured (for remote access)
- [ ] Sufficient resources available (check with `free -h` and `df -h`)

**Service-Specific Requirements:**
- [ ] Media directory created and accessible (e.g., `~/media/comics`)
- [ ] Comics/manga collection organized (see media organization section below)
- [ ] Port 8081 available (or alternate port chosen)

**Resource Availability Check:**
```bash
# Check available RAM (need at least 512MB free)
free -h

# Check available disk space (need enough for your media + ~100MB for Komga)
df -h /

# Check CPU load (should be <4.0 on 8-core system)
uptime
```

## Pre-Deployment Steps

### 1. Clone Repository
```bash
cd ~/code
git clone https://github.com/yourusername/homelab-komga.git
cd homelab-komga
```

### 2. Create Media Directory
```bash
# Create directory for your comics/manga
mkdir -p ~/media/comics

# Set proper permissions
chmod 755 ~/media/comics
```

### 3. Configure Environment
```bash
cp .env.example .env
nano .env
```

**Required variables to set:**
- `KOMGA_MEDIA_PATH` - Absolute path to your comics directory (e.g., `/home/jayblack388/media/comics`)
- `KOMGA_PORT` - Port for web access (default: 8081)
- `PUID` - Your user ID (run `id -u` to find it, typically 1000)
- `PGID` - Your group ID (run `id -g` to find it, typically 1000)

**Optional variables:**
- `TZ` - Your timezone (default: America/New_York)

**Example .env file:**
```bash
KOMGA_PORT=8081
KOMGA_MEDIA_PATH=/home/jayblack388/media/comics
PUID=1000
PGID=1000
TZ=America/New_York
```

### 4. Verify User/Group IDs
```bash
# Check your user ID
id -u
# Should match PUID in .env

# Check your group ID
id -g
# Should match PGID in .env
```

### 5. Pre-Flight Check
```bash
# Validate docker-compose.yml syntax
docker compose config

# Check for port conflicts
sudo netstat -tuln | grep 8081

# Verify media directory exists and is readable
ls -la ~/media/comics
```

## Media Organization (Before First Scan)

### Recommended Directory Structure

**Option 1: One Series Per Folder (Recommended)**
```
~/media/comics/
├── Berserk/
│   ├── Berserk v01.cbz
│   ├── Berserk v02.cbz
│   └── ...
├── Naruto/
│   ├── Naruto v001.cbz
│   ├── Naruto v002.cbz
│   └── ...
└── One Piece/
    ├── One Piece v001.cbz
    └── ...
```

**Option 2: Keep Image Folders**
```
~/media/comics/
├── Bleach/
│   ├── Volume 01/
│   │   ├── 001.jpg
│   │   ├── 002.jpg
│   │   └── ...
│   ├── Volume 02/
│   └── ...
```

### File Naming Best Practices

**Good naming examples:**
- `Berserk v01.cbz`
- `Naruto c001.cbz`
- `One Piece 001.cbz`

**Avoid:**
- `berserk.zip` (no volume number)
- `Naruto_Chapter_1.cbz` (inconsistent format)
- `OP_v1.cbz` (unclear abbreviation)

**Komga parses these patterns:**
- `v01`, `v001`, `vol 01`, `volume 01` - All work
- `c01`, `c001`, `ch 01`, `chapter 01` - All work
- Consistency matters more than specific format

## Deployment

### 1. Start Service
```bash
docker compose up -d
```

Expected output:
```
[+] Running 3/3
 ✔ Network komga-net       Created
 ✔ Volume komga-config     Created
 ✔ Container komga         Started
```

### 2. Initial Verification
```bash
# Container is running
docker ps | grep komga
# Should show: Up X seconds (healthy) after ~60 seconds

# Logs show successful start
docker logs komga
# Should show: "Started KomgaApplication in X seconds"

# Health check is passing
docker inspect komga | grep -A5 Health
# Should show: "Status": "healthy" after start_period
```

### 3. Service Accessibility
```bash
# Health endpoint responds
curl http://localhost:8081/api/v1/actuator/health
# Should return: {"status":"UP"}

# Main UI accessible
curl -I http://localhost:8081/
# Should return: 200 OK
```

### 4. Access from Browser
- **Local:** `http://homelab.local:8081`
- **Tailscale:** `http://orangepi:8081`

**Initial Setup Wizard:**
1. Open web interface in browser
2. Create admin account:
   - Username: (your choice)
   - Password: (strong password - save in password manager)
   - Email: optional
3. Click "Create Account"

## Initial Configuration

### 1. Add Library
1. Click "Add Library" or go to Settings → Libraries
2. Configure library:
   - **Name:** "Manga" (or "Comics")
   - **Root folder:** `/data` (this maps to your KOMGA_MEDIA_PATH)
   - **Scan interval:** Manual (trigger scans when you add content)
   - **Import CBR/CBZ:** Enabled
   - **Import EPUB:** Enable if needed
3. Click "Add"

### 2. Initial Library Scan
1. Click "Scan Library Now" in library settings
2. Monitor progress:
   - Top-right corner shows scanning indicator
   - Check Tasks page for detailed progress
   - Initial scan takes 5-10 minutes for 50GB+
3. Wait for scan to complete

### 3. Verify Library Contents
1. Navigate to "Libraries" in web UI
2. Browse your series - are they detected correctly?
3. Check for issues:
   - Missing series (files not detected)
   - Volumes not grouped correctly
   - See troubleshooting guide for fixes

## Integration Steps

### Uptime Kuma Integration

1. Access Uptime Kuma: `http://orangepi:3002`

2. Add New Monitor:
   - Click "Add New Monitor"
   - **Monitor Type:** HTTP(s)
   - **Friendly Name:** Komga
   - **URL:** `http://komga:25600/api/v1/actuator/health`
   - **Heartbeat Interval:** 60 seconds
   - **Retries:** 2
   - **Heartbeat Retry Interval:** 60 seconds
   - **Notification:** Select configured notification channel

3. Verify monitor shows green "Up" status

### Prometheus Integration

1. Add scrape config to Prometheus:
```bash
nano ~/code/homelab/monitoring/prometheus/prometheus.yml
```

Add under `scrape_configs`:
```yaml
  - job_name: 'komga'
    static_configs:
      - targets: ['komga:25600']
    metrics_path: '/api/v1/actuator/prometheus'
    scrape_interval: 15s
```

2. Reload Prometheus:
```bash
docker exec prometheus kill -HUP 1
```

3. Verify scrape target:
- Open `http://orangepi:9090/targets`
- Find `komga` job
- Should show "UP" in green

### Homepage Dashboard Integration

1. Add service to Homepage config:
```bash
nano ~/code/homelab/homepage/config/services.yaml
```

Add entry:
```yaml
- Media:
    - Komga:
        icon: komga.png
        href: http://orangepi:8081
        description: Comic/Manga Server
        widget:
          type: komga
          url: http://komga:25600
          username: your-komga-username
          password: your-komga-password
```

2. Reload Homepage:
```bash
docker compose -f ~/code/homelab/homepage/docker-compose.yml restart
```

## Post-Deployment

### 1. Configure Metadata Scraping (Optional)

**In Komga UI:**
1. Go to Settings → Metadata Sources
2. Enable providers:
   - For manga: Enable AniList and/or MyAnimeList
   - For western comics: Enable ComicVine
3. Set priority order (which source to check first)
4. Save settings

**Per-series metadata:**
1. Browse to series in Komga
2. Click "Edit Metadata"
3. Search for series in enabled providers
4. Select correct match from results
5. Save - metadata populates automatically

### 2. Configure Mobile Access (Optional)

**OPDS URL for comic reader apps:**
```
http://orangepi:8081/opds/v1.2/catalog
```

**Recommended apps:**
- **iOS:** Panels ($9.99), Chunky Reader (free)
- **Android:** Tachiyomi (free), Moon+ Reader (free)

**Setup in app:**
1. Add OPDS source
2. URL: `http://orangepi:8081/opds/v1.2/catalog`
3. Username/password: Your Komga credentials
4. Browse and download comics to read offline

### 3. Documentation
- [ ] Update service inventory in Obsidian
- [ ] Document actual resource usage after 24 hours
- [ ] Note any issues encountered during deployment

## Verification Checklist

**Before marking deployment complete:**

- [ ] Container running: `docker ps | grep komga`
- [ ] Health check passing: Docker shows "(healthy)"
- [ ] Service accessible via browser
- [ ] Initial admin account created
- [ ] Library added and scanned successfully
- [ ] Series appear correctly in web UI
- [ ] Uptime Kuma monitor added and green
- [ ] Prometheus scraping (if applicable)
- [ ] Homepage entry added
- [ ] Documentation updated

## Common Issues During Deployment

### Port Already in Use
```bash
# Find what's using the port
sudo netstat -tulpn | grep :8081

# Either:
# 1. Stop conflicting service
# 2. Change KOMGA_PORT in .env to different port
```

### Container Immediately Exits
```bash
# Check logs for error
docker logs komga

# Common causes:
# - Invalid KOMGA_MEDIA_PATH (directory doesn't exist)
# - Permission issues with config volume
# - User ID mismatch (PUID/PGID incorrect)
```

### Health Check Failing
```bash
# Check if service is actually ready
docker logs komga | grep "Started KomgaApplication"

# Manually test health endpoint
docker exec komga curl -f http://localhost:25600/api/v1/actuator/health

# If still failing after start_period, check logs for errors
```

### Permission Denied Errors
```bash
# Check media directory permissions
ls -la ~/media/comics

# Fix permissions if needed
sudo chown -R $(id -u):$(id -g) ~/media/comics

# Verify PUID/PGID in .env match your user
id -u  # Should match PUID
id -g  # Should match PGID
```

### Library Not Scanning
```bash
# Check container can see media files
docker exec komga ls -la /data

# If empty, verify KOMGA_MEDIA_PATH in .env is correct
cat .env | grep KOMGA_MEDIA_PATH

# Restart container after fixing .env
docker compose down && docker compose up -d
```

## Rollback Procedure

If deployment fails or service is broken:

```bash
# Stop and remove service
docker compose down

# Remove volumes (DESTRUCTIVE - only if acceptable)
docker volume rm komga-config

# Review logs to understand failure
docker logs komga > ~/komga-deployment-failure.log

# Fix issue and redeploy
nano .env  # or docker-compose.yml
docker compose up -d
```

## Next Steps

After successful deployment:
- Monitor resource usage in Grafana for 24 hours
- Verify Uptime Kuma shows green status
- Test service functionality thoroughly
- Begin organizing media collection (see Komga resource guide)
- Set up metadata scraping for your favorite series
- Configure OPDS for mobile reading
