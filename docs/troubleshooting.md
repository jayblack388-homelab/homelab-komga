# Troubleshooting Guide

## Quick Diagnostic Commands

```bash
# Container status
docker ps -a | grep komga

# Recent logs
docker logs --tail 100 komga

# Follow logs in real-time
docker logs -f komga

# Health check status
docker inspect komga | grep -A10 Health

# Resource usage
docker stats komga --no-stream

# Restart container
docker compose restart komga

# Full restart (down and up)
docker compose down && docker compose up -d
```

## Common Issues

### Service Won't Start

**Symptoms:**
- Container exits immediately after start
- `docker ps` doesn't show container
- `docker ps -a` shows "Exited (1)" or similar

**Diagnosis:**
```bash
# Check logs for error message
docker logs komga

# Check if port is already in use
sudo netstat -tulpn | grep 8081

# Verify environment variables
docker compose config | grep -A20 environment

# Check media path exists
ls -la ~/media/comics
```

**Common Causes:**

**1. Invalid Media Path**
```
Error: Cannot access /data: No such file or directory
```
Fix: Verify `KOMGA_MEDIA_PATH` in `.env` points to valid directory
```bash
# Check .env setting
cat .env | grep KOMGA_MEDIA_PATH

# Verify directory exists
ls -la /path/from/env

# Create if missing
mkdir -p ~/media/comics
```

**2. Port Conflict**
```
Error: bind: address already in use
```
Fix: Change `KOMGA_PORT` in `.env` or stop conflicting service
```bash
# Find what's using port 8081
sudo netstat -tulpn | grep :8081

# Either stop that service or change Komga port
nano .env  # Change KOMGA_PORT to 8082 or similar
docker compose up -d
```

**3. Permission Issues**
```
Error: permission denied: /config
```
Fix: Verify PUID/PGID match your user
```bash
# Check your IDs
id -u  # Should match PUID in .env
id -g  # Should match PGID in .env

# Update .env if needed
nano .env

# Recreate container
docker compose down && docker compose up -d
```

**4. Config Volume Corruption**
```
Error: Database corrupted or locked
```
Fix: Remove and recreate config volume
```bash
# DESTRUCTIVE: Removes all Komga settings and metadata
docker compose down
docker volume rm komga-config
docker compose up -d
# Will need to reconfigure Komga from scratch
```

### Health Check Failing

**Symptoms:**
- Container shows "unhealthy" in `docker ps`
- Uptime Kuma shows service down
- Container may be restarting

**Diagnosis:**
```bash
# Check health check status
docker inspect komga | jq '.[0].State.Health'

# Manually run health check command inside container
docker exec komga curl -f http://localhost:25600/api/v1/actuator/health

# Check if Komga is still starting
docker logs komga | grep "Started KomgaApplication"
```

**Common Causes:**

**1. Service Not Ready Yet**
- Health check runs before Komga finishes starting
- Wait for start_period (60s) to elapse
- Check logs for "Started KomgaApplication" message

**2. Out of Memory**
- Komga killed by OOM during startup
- Check: `docker logs komga | grep -i "killed"`
- Fix: Increase memory limit in docker-compose.yml

**3. Corrupted Database**
- Health endpoint fails even though container runs
- Fix: Remove config volume and restart (see above)

### Library Won't Scan

**Symptoms:**
- Click "Scan Library" but no books appear
- Scan completes with 0 books found
- Scan progress shows but never completes

**Diagnosis:**
```bash
# Check if container can see media files
docker exec komga ls -la /data

# Should list your comics/manga
# If empty, media path is wrong

# Check Komga logs during scan
docker logs -f komga
# Click "Scan Library" and watch for errors
```

**Common Causes:**

**1. Incorrect Media Path**
- Library root folder in Komga UI must be `/data`
- NOT the host path like `~/media/comics`
- Fix: Edit library in Komga UI, set root to `/data`

**2. Empty Media Directory**
- Host directory is empty or doesn't exist
- Fix: Add comics to `KOMGA_MEDIA_PATH` location on host

**3. Permission Issues**
- Container can't read media files
```bash
# Check permissions on host
ls -la ~/media/comics

# Should be readable by your user (PUID/PGID)
# Fix if needed
chmod -R 755 ~/media/comics
```

**4. Unsupported File Formats**
- Files aren't recognized as comics
- Supported: .cbz, .cbr, .cb7, .zip, .rar, image folders
- Fix: Convert files to supported format

### Books Missing After Scan

**Symptoms:**
- Scan completes successfully
- Some series or volumes don't appear
- Known files not showing up

**Causes:**
- Files too deeply nested (Komga scans 2 levels deep by default)
- Corrupted archives
- Unsupported image formats inside archives
- Files hidden by incorrect permissions

**Diagnosis:**
```bash
# Test archive integrity
unzip -t /path/to/comic.cbz
# OR
unrar t /path/to/comic.cbr

# Check Komga logs for specific file errors
docker logs komga | grep -i error | grep -i "/data"

# Verify file structure depth
ls -R ~/media/comics | head -50
```

**Fixes:**
1. **Move files to proper depth:**
   ```bash
   # Bad: ~/media/comics/Publisher/Series/Volume/file.cbz (4 levels)
   # Good: ~/media/comics/Series/file.cbz (2 levels)
   ```

2. **Fix corrupted archives:**
   ```bash
   # Extract and re-create CBZ
   unzip problem.cbz -d /tmp/fix/
   cd /tmp/fix
   zip -r ../fixed.cbz *
   mv ../fixed.cbz ~/media/comics/Series/
   ```

3. **Verify permissions:**
   ```bash
   # All files should be readable by your user
   chmod -R 644 ~/media/comics/**/*.cbz
   chmod -R 755 ~/media/comics/*/
   ```

### Metadata Not Scraping

**Symptoms:**
- Series has no cover image
- Description/summary missing
- "Search" in Edit Metadata returns no results

**Causes:**
- Provider API down or rate limited
- Series name doesn't match database
- Metadata scraper not enabled

**Diagnosis:**
```bash
# Check Komga logs during metadata search
docker logs -f komga
# Try searching in UI, watch for API errors

# Check network connectivity
docker exec komga curl -I https://anilist.co
```

**Fixes:**

**1. Enable Metadata Providers:**
- Settings → Metadata Sources
- Enable AniList, MyAnimeList, or ComicVine
- Save settings, try search again

**2. Fix Series Name Matching:**
- Series name in Komga must match database name
- Try editing series name to official title
- Use English title for better matches

**3. Manual Metadata Entry:**
- If series not in databases, add manually
- Edit Metadata → Fill in fields manually
- Upload cover image if available

**4. Wait for Rate Limit:**
- APIs have rate limits (e.g., 90 requests/minute)
- Wait 1-2 minutes and try again
- Don't spam search requests

### High Memory Usage

**Symptoms:**
- Container using more RAM than expected
- System becomes slow
- Komga crashes or gets killed

**Diagnosis:**
```bash
# Current memory usage
docker stats komga --no-stream

# Memory limit
docker inspect komga | grep -i memory

# System memory status
free -h

# Check for memory leak over time
watch -n 5 'docker stats komga --no-stream'
```

**Common Causes:**

**1. Large Library Scan**
- Memory usage spikes during initial scan
- Expected behavior with 10,000+ files
- Fix: Wait for scan to complete, memory will drop

**2. Insufficient Memory Limit**
- Limit too low for library size
- Fix: Increase in docker-compose.yml
```yaml
deploy:
  resources:
    limits:
      memory: 1G  # Increase from 512M
```

**3. Thumbnail Generation**
- Generating thumbnails for all books uses memory
- Fix: Settings → Thumbnails → Lower quality/size

**4. Memory Leak (Rare)**
- Memory usage increases over days/weeks
- Fix: Restart service periodically
```bash
docker compose restart komga
```

### High CPU Usage

**Symptoms:**
- System load average >6.0
- Fans running constantly
- Services slow to respond

**Diagnosis:**
```bash
# CPU usage per container
docker stats --no-stream

# System load
uptime

# Komga-specific processes
docker exec komga ps aux
```

**Common Causes:**

**1. Initial Library Scan**
- CPU intensive during first scan
- Expected behavior, wait for completion
- Scan progress visible in Tasks page

**2. Thumbnail Generation**
- Generating thumbnails for all books
- Expected during initial setup
- Reduce load: Settings → Thumbnails → Lower quality

**3. Metadata Scraping**
- Fetching metadata for many series at once
- Throttle by doing series one at a time

**4. Corrupted File Processing**
- Komga stuck trying to process bad archive
- Check logs for file being processed repeatedly
- Remove problematic file, rescan

### Cannot Access Service

**Symptoms:**
- Browser shows "connection refused" or timeout
- `curl http://localhost:8081` fails
- Service works locally but not via Tailscale

**Diagnosis:**
```bash
# Is container running?
docker ps | grep komga

# Is port actually mapped?
docker port komga

# Can you access locally?
curl http://localhost:8081

# Can you access from inside container?
docker exec komga curl http://localhost:25600

# Tailscale running?
tailscale status
```

**Common Causes:**

**1. Container Not Running**
- Service crashed or failed to start
- Check logs: `docker logs komga`
- Restart: `docker compose up -d`

**2. Port Mapping Wrong**
- .env has wrong port or missing
- Fix: Verify KOMGA_PORT in .env
```bash
cat .env | grep KOMGA_PORT
# Should match port you're trying to access
```

**3. Tailscale Issue**
- Tailscale not connected
```bash
tailscale status  # Check connection
sudo systemctl restart tailscaled  # Restart if needed
```

**4. Firewall Blocking**
- UFW blocking port
```bash
sudo ufw status
# If enabled, allow port
sudo ufw allow 8081
```

### Books Won't Open/Read

**Symptoms:**
- Series appears in library
- Click to read, page doesn't load
- "Error loading page" message

**Causes:**
- Corrupted archive
- Unsupported image format inside archive
- Permission issues preventing read
- Browser cache issue

**Fixes:**

**1. Test Archive Integrity:**
```bash
unzip -t ~/media/comics/Series/volume.cbz
# Should show "OK" for all files
# If errors, archive is corrupted
```

**2. Check Image Formats:**
```bash
unzip -l ~/media/comics/Series/volume.cbz | grep -E "\.(jpg|png|gif|webp)"
# Should only show supported image types
# If other formats, may not display
```

**3. Clear Browser Cache:**
- Hard refresh: Ctrl+Shift+R (Linux/Windows) or Cmd+Shift+R (Mac)
- Or clear browser cache completely

**4. Check Logs:**
```bash
docker logs komga | grep -i error
# Look for errors related to specific file
```

### OPDS Not Working on Mobile

**Symptoms:**
- Can't connect to OPDS feed from mobile app
- Authentication fails
- Catalog loads but shows no books

**Diagnosis:**
```bash
# Test OPDS endpoint
curl -u username:password http://orangepi:8081/opds/v1.2/catalog

# Should return XML with catalog data
```

**Fixes:**

**1. Wrong URL:**
- Must use Tailscale hostname: `http://orangepi:8081`
- NOT `localhost` or local IP from mobile

**2. Wrong Credentials:**
- Use Komga username/password, not system credentials
- Verify by logging into web UI

**3. Tailscale Not Connected:**
- Ensure mobile device connected to Tailscale
- Check Tailscale app shows "Connected"

**4. OPDS Path Wrong:**
- Full path: `http://orangepi:8081/opds/v1.2/catalog`
- Some apps need `/opds/v1.2/catalog`, others just `/opds`

## Service-Specific Issues

### Database Lock Errors

**Symptoms:**
```
Error: database is locked
```

**Cause:** SQLite database being accessed by multiple processes

**Fix:**
```bash
# Stop container
docker compose down

# Remove lock file
docker volume inspect komga-config
# Note the Mountpoint path
sudo rm /var/lib/docker/volumes/komga-config/_data/database.sqlite-shm
sudo rm /var/lib/docker/volumes/komga-config/_data/database.sqlite-wal

# Restart
docker compose up -d
```

### Series Grouped Incorrectly

**Symptoms:**
- Volumes from different series grouped together
- One series split into multiple

**Cause:** Inconsistent naming or metadata

**Fix:**
1. **In Komga UI:**
   - Select misplaced volumes
   - Right-click → Move to correct series
   - Or: Edit metadata to fix series name

2. **On disk (better long-term):**
   ```bash
   # Rename files consistently
   # Before: Naruto_1.cbz, naruto_vol_2.cbz
   # After: Naruto v01.cbz, Naruto v02.cbz
   # Then rescan library
   ```

## When to Seek Help

**Before posting to forums or GitHub:**

1. Collect diagnostic information:
```bash
# Save logs
docker logs komga > ~/komga-logs.txt

# Save configuration (redact passwords!)
docker compose config > ~/komga-config.yml

# Save system info
docker version > ~/system-info.txt
docker info >> ~/system-info.txt
free -h >> ~/system-info.txt
df -h >> ~/system-info.txt
```

2. Check Komga GitHub Issues: https://github.com/gotson/komga/issues
3. Search Komga Discord: https://discord.gg/TdRpkDu
4. Review official documentation: https://komga.org/

**What to include in help request:**
- Komga version (from docker-compose.yml)
- Error messages from logs
- Steps to reproduce
- What you've already tried
- System info (Orange Pi 5, Ubuntu, Docker version)
- Sanitized docker-compose.yml (remove passwords!)

## Advanced Troubleshooting

### Access Database Directly

```bash
# CAUTION: Only if you know what you're doing
# Stop Komga first
docker compose down

# Access database
docker run --rm -it \
  -v komga-config:/config \
  alpine:latest sh

# Inside container
apk add sqlite
cd /config
sqlite3 database.sqlite
# Run SQL queries
.tables
.quit
exit

# Restart Komga
docker compose up -d
```

### Force Re-scan Everything

```bash
# In Komga UI
# Settings → Server → Analyze All Books
# OR
# Settings → Libraries → [Library] → Empty Trash → Scan

# This rebuilds all metadata and thumbnails
```

### Export/Import Metadata

**Export (before major changes):**
1. Settings → Server → Export
2. Downloads JSON with all metadata
3. Save as backup

**Import (to restore):**
1. Settings → Server → Import
2. Upload previously exported JSON
3. Restores metadata to state at export time
