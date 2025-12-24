# slurpit-docker-komodo-traefik
Compose files to use with Komodo, to deploy Slurpit network discovery behind a Traefik reverse proxy with Cloudflare DNS validation for TLS certs.

I had some struggles getting Slurpit (slurpit.io) working in my preferred Docker-Komodo-Traefik stack.  This is just a simple repo with how I got it working, as well as splitting out credentials etc. into an env file instead of leaving them in the compose file.  Bad developer!  Bad!  No treats for you!

Mainly it was re-focusing on docker instead of podman, env stuff, and fixing some junk with nginx in the portal container not playing nice with the extra layer of reverse proxy from Traefik.  See Cursor's lovely writeup about what it did to fix it below.

Have fun!  If it doesn't work for you, fix it.  There are enough AI tools out there that you can fix it without being a developer.

---


## Part 1: MongoDB Connection Errors

### Issues Found

1. **Warehouse Container Failing to Connect to MongoDB**
   - Error: `pymongo.errors.InvalidURI: Username and password must be escaped according to RFC 3986`
   - The MongoDB password contained special characters (`+`, `=`, `@`) that needed URL encoding in the connection URI

2. **Cascading Failures**
   - Scraper and Scanner containers couldn't connect to warehouse (connection refused)
   - Portal container stuck in "Created" state waiting for warehouse health check

### Root Cause

The MongoDB password contained special characters that must be URL-encoded when used in a MongoDB connection URI. The `@` character is particularly problematic as it's used as a delimiter in URIs.

**Original Password Pattern:** `[REDACTED - contained +, =, and @ characters]`

### Fixes Applied

#### 1. URL-Encoded MongoDB Password

Added new environment variable to `/etc/komodo/stacks/slurpit/.env`:

```bash
MONGO_INITDB_ROOT_PASSWORD_ENCODED=[URL-ENCODED-PASSWORD]
```

**Encoding Reference:**
- `+` → `%2B`
- `=` → `%3D`
- `@` → `%40`

To encode a password, use Python:
```python
from urllib.parse import quote_plus
encoded = quote_plus('your-password-here')
```

#### 2. Updated Docker Compose Configuration

Modified `/etc/komodo/stacks/slurpit/compose.yaml`:

**Before:**
```yaml
WAREHOUSE_MONGODB_URI: mongodb://${MONGO_INITDB_ROOT_USERNAME}:${MONGO_INITDB_ROOT_PASSWORD}@slurpit-mongodb:27017/${MONGO_DATABASE}?authSource=admin
```

**After:**
```yaml
WAREHOUSE_MONGODB_URI: mongodb://${MONGO_INITDB_ROOT_USERNAME}:${MONGO_INITDB_ROOT_PASSWORD_ENCODED}@slurpit-mongodb:27017/${MONGO_DATABASE}?authSource=admin
```

#### 3. Recreated Services

```bash
cd /etc/komodo/stacks/slurpit
docker compose up -d --force-recreate slurpit-warehouse
docker compose restart slurpit-scraper slurpit-scanner
docker compose up -d slurpit-portal
```

### Result

All containers started successfully:
- ✅ Warehouse connected to MongoDB
- ✅ Scraper and Scanner connected to warehouse  
- ✅ Portal became healthy

---

## Part 2: Nginx/HTTPS Proxy Configuration Fixes

### Issues Found

1. **Insecure Connection Warning**
   - Browser warning: "The information you have entered on this page will be sent over an insecure connection"
   - Portal couldn't detect it was behind HTTPS/TLS

2. **JSON Redirect with HTTP URL**
   - After login, received JSON: `{"success":true,"url":"http://slurpit.domain.tld/admin/dashboard"}`
   - Should redirect to HTTPS but was generating HTTP URLs

### Root Cause

The portal's Nginx configuration wasn't:
1. Trusting the Traefik proxy (missing `real_ip` configuration)
2. Passing `X-Forwarded-Proto` headers to PHP
3. Setting PHP `HTTPS` and `REQUEST_SCHEME` variables correctly

Traefik was setting `X-Forwarded-Proto: https`, but Nginx wasn't using it to inform PHP about the actual connection scheme.

### Fixes Applied

#### 1. Created Custom Nginx Configuration Files

##### a) `/etc/komodo/stacks/slurpit/nginx-conf/nginx.conf`

Created with proxy trust configuration:

```nginx
http {
    # ... existing configuration ...
    
    # Trust proxy networks (Traefik and Docker networks)
    set_real_ip_from 172.23.0.0/16;
    set_real_ip_from 172.22.0.0/16;
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from 172.16.0.0/12;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    
    # Map X-Forwarded-Proto to HTTPS variable for PHP
    map $http_x_forwarded_proto $forwarded_https {
        default off;
        https on;
    }
    
    # Map X-Forwarded-Proto to scheme
    map $http_x_forwarded_proto $forwarded_scheme {
        default $scheme;
        https https;
        http http;
    }
    
    # ... rest of configuration ...
}
```

**Key Points:**
- `set_real_ip_from` trusts specific Docker network ranges
- `real_ip_header X-Forwarded-For` extracts client IP from proxy headers
- `map` directives convert `X-Forwarded-Proto` header to PHP-friendly variables

##### b) `/etc/komodo/stacks/slurpit/nginx-conf/default.conf`

Modified PHP location block to pass proxy headers:

```nginx
location ~* \.php$ {
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    include fastcgi.conf;
    
    # Pass proxy headers to PHP for HTTPS detection
    fastcgi_param HTTPS $forwarded_https;
    fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
    fastcgi_param REQUEST_SCHEME $forwarded_scheme;
}
```

**Key Points:**
- `fastcgi_param HTTPS` sets `$_SERVER['HTTPS']` in PHP
- `fastcgi_param REQUEST_SCHEME` sets `$_SERVER['REQUEST_SCHEME']` in PHP
- Uses mapped variables from `nginx.conf` to properly detect HTTPS

#### 2. Updated Docker Compose File

Modified `/etc/komodo/stacks/slurpit/compose.yaml` to mount custom Nginx configs:

```yaml
slurpit-portal:
  # ... other configuration ...
  volumes:
    - ./nginx-conf/nginx.conf:/etc/nginx/nginx.conf:ro
    - ./nginx-conf/default.conf:/etc/nginx/conf.d/default.conf:ro
    - slurpit-nginx-logs:/var/log/nginx/
    - slurpit-php-logs:/var/log/php/
    - slurpit-certs:/etc/nginx/certs/
    - slurpit-portal-backup:/backup/files
```

**Note:** Files are mounted as read-only (`:ro`) to prevent container modifications.

#### 3. Traefik Configuration (Already Present)

The Traefik middleware was already configured correctly:

```yaml
labels:
  - "traefik.http.middlewares.slurpit-headers.headers.customrequestheaders.X-Forwarded-Proto=https"
  - "traefik.http.middlewares.slurpit-headers.headers.customrequestheaders.X-Forwarded-Host=${SLURPIT_DOMAIN}"
  - "traefik.http.routers.slurpit.middlewares=slurpit-sec-headers@docker,slurpit-headers@docker"
```

#### 4. Recreated Portal Container

```bash
cd /etc/komodo/stacks/slurpit
docker compose up -d --force-recreate slurpit-portal
```

### How the Fix Works

1. **Traefik** sends requests with `X-Forwarded-Proto: https` header
2. **Nginx** trusts the proxy and extracts real client IP from `X-Forwarded-For`
3. **Nginx** maps `X-Forwarded-Proto` header to variables:
   - `$forwarded_https` = "on" when proto is "https"
   - `$forwarded_scheme` = "https" when proto is "https"
4. **PHP** receives correct environment variables:
   - `$_SERVER['HTTPS'] = 'on'`
   - `$_SERVER['REQUEST_SCHEME'] = 'https'`
   - `$_SERVER['HTTP_X_FORWARDED_PROTO'] = 'https'`
5. **PHP Application** correctly detects HTTPS and generates HTTPS URLs

### Result

- ✅ No more insecure connection warnings
- ✅ Portal correctly redirects to HTTPS URLs after login
- ✅ All URLs generated by the application use HTTPS scheme

---

## Files Created/Modified

### Created Files:
1. `/etc/komodo/stacks/slurpit/nginx-conf/nginx.conf` - Custom Nginx main config with proxy trust
2. `/etc/komodo/stacks/slurpit/nginx-conf/default.conf` - Custom Nginx server config with PHP proxy headers

### Modified Files:
1. `/etc/komodo/stacks/slurpit/.env` - Added `MONGO_INITDB_ROOT_PASSWORD_ENCODED` variable
2. `/etc/komodo/stacks/slurpit/compose.yaml` - Updated MongoDB URI and added Nginx volume mounts

---

## Verification Steps

### Verify MongoDB Connection:
```bash
docker logs slurpit-warehouse | grep -i "MongoDB response time"
# Should show: [WARERHOUSE][INFO] MongoDB response time: XX.XX ms
```

### Verify Nginx Configuration:
```bash
docker exec slurpit-portal nginx -t
# Should show: nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Verify Proxy Headers:
```bash
docker exec slurpit-portal cat /etc/nginx/nginx.conf | grep -A 5 "set_real_ip"
docker exec slurpit-portal cat /etc/nginx/conf.d/default.conf | grep "fastcgi_param HTTPS"
```

### Verify Container Status:
```bash
docker ps --filter "name=slurpit" --format "table {{.Names}}\t{{.Status}}"
# All containers should show as "Up" and portal should show "(healthy)"
```

---

## Notes

- The MongoDB password encoding is necessary because Docker Compose doesn't automatically URL-encode environment variables when constructing URIs
- The Nginx configuration uses `map` directives which must be in the `http` block (not `server` block)
- Network ranges in `set_real_ip_from` should match your Docker network configuration
- The custom Nginx configs are mounted as read-only to prevent accidental modifications
- Always test Nginx configuration with `nginx -t` before restarting services

---

## Troubleshooting

### If MongoDB connection still fails:
1. Verify the encoded password is correct: `echo $MONGO_INITDB_ROOT_PASSWORD_ENCODED`
2. Check warehouse logs: `docker logs slurpit-warehouse --tail 50`
3. Verify MongoDB is healthy: `docker ps | grep mongodb`

### If HTTPS detection still doesn't work:
1. Verify Traefik headers are being sent: Check Traefik access logs
2. Verify Nginx config is loaded: `docker exec slurpit-portal nginx -T | grep forwarded`
3. Test from container: `docker exec slurpit-portal curl -H "X-Forwarded-Proto: https" http://localhost`
4. Check PHP environment: Add `phpinfo()` temporarily to verify `$_SERVER['HTTPS']` value

---
