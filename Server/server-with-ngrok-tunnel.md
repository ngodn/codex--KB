# Multi-App Server Setup with Ngrok Domain

## System Information
- **OS**: Ubuntu 25.04 (Plucky Puffin)
- **Kernel**: Linux 6.14.0-27-generic
- **Architecture**: x86_64
- **Domain**: eins0fx.aequify.gg (ngrok)

## Goal
Set up a server to host multiple applications on subpaths:
- `eins0fx.aequify.gg/d42` → Device42 Application (VM at 192.168.1.42)
- `eins0fx.aequify.gg/localapp` → Local Application
- `eins0fx.aequify.gg/staticapp` → Static Sample Application

## Server Implementation: Nginx (Production-Ready)

After extensive testing, **Nginx proved to be the optimal solution** for complex web application proxying, especially for applications like Device42 that require:
- Advanced URL rewriting and content filtering
- Complex redirection handling
- Base tag injection for relative URL resolution
- Mixed content security policy management
- Embedded analytics dashboard support (Superset)

### Why Nginx Over Caddy

While Caddy is excellent for simple setups, our Device42 implementation required:
- ✅ **Advanced `sub_filter` capabilities** for HTML content rewriting
- ✅ **Complex redirect handling** with `proxy_redirect` 
- ✅ **Fine-grained content type filtering** (`sub_filter_types`)
- ✅ **Robust session management** with proper cookie scoping
- ✅ **CSP header manipulation** for mixed content resolution
- ✅ **POST data preservation** during redirects

## Complete Working Implementation

### Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx
```

### Step 2: Create the Production Nginx Configuration

Create `/etc/nginx/sites-available/multi-app`:

```nginx
server {
    listen 80;
    server_name _;
    
    # Security headers including CSP upgrade-insecure-requests
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Content-Security-Policy "upgrade-insecure-requests; base-uri 'self'; default-src 'self'; frame-src 'self' https://app.pendo.io blob:; frame-ancestors 'self'; connect-src 'self' http://eins0fx.aequify.gg https://eins0fx.aequify.gg https://w1.device42.com https://events.mapbox.com https://api.mapbox.com https://registration.device42.com https://app.pendo.io; object-src 'none'; script-src 'self' blob: https://code.jquery.com https://cdn4.device42.com https://*.cdni.device42.com https://app.pendo.io 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline' data: https: http:; img-src 'self' data: https://*.device42.com https://*.cdni.device42.com https://app.pendo.io; font-src 'self' data:;";
    
    # Landing page at root
    location = / {
        root /srv/www;
        try_files /index.html =404;
    }
    
    # CRITICAL FIX: Handle double /d42/ URLs
    location ~ ^/d42/d42/(.*)$ {
        return 301 /d42/$1;
    }
    
    # CRITICAL: Redirect /vmapp/ to /d42/ for old JavaScript references
    location /vmapp/ {
        rewrite ^/vmapp/(.*)$ /d42/$1 permanent;
    }
    
    # NEW: Redirect Superset to go through /d42/ for proper authentication context
    location /superset/ {
        return 301 /d42$request_uri;
    }
    
    # CRITICAL FIX: Redirect all absolute paths to /d42/ to maintain session context
    location ~ ^/(admin|api|jsi18n|internal|media|ajax|login|logout|oidcauth)(/.*)?$ {
        return 301 /d42$request_uri;
    }
    
    # VM App - Base tag injection + comprehensive fixes + Superset support
    location /d42/ {
        proxy_pass https://192.168.1.42/;
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        
        # Essential proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Script-Name /d42;
        proxy_set_header X-Forwarded-Prefix /d42;
        
        # Hide Device42's restrictive CSP header
        proxy_hide_header Content-Security-Policy;
        
        # CRITICAL: Disable compression for sub_filter to work
        proxy_set_header Accept-Encoding "";
        
        # Cookie and session handling
        proxy_cookie_path / /d42/;
        
        # FIXED: Smart redirect handling to prevent double /d42/
        proxy_redirect / /d42/;
        proxy_redirect https://192.168.1.42/ /d42/;
        proxy_redirect http://192.168.1.42/ /d42/;
        proxy_redirect ~^/d42/(.*)$ /d42/$1;  # Prevent double d42
        
        # ULTIMATE BASE TAG INJECTION FIX
        sub_filter_types text/html application/javascript text/javascript text/css;
        
        # 1. Base tag injection - FORCES all relative URLs to use /d42/
        sub_filter '<head>' '<head><base href="https://eins0fx.aequify.gg/d42/">';
        
        # 2. Fix any remaining absolute references in JavaScript
        sub_filter '"/admin' '"/d42/admin';
        sub_filter '"/api' '"/d42/api';
        sub_filter '"/internal' '"/d42/internal';
        sub_filter '"/ajax' '"/d42/ajax';
        sub_filter '"/superset' '"/d42/superset';
        sub_filter "'/admin" "'/d42/admin";
        sub_filter "'/api" "'/d42/api";
        sub_filter "'/internal" "'/d42/internal";
        sub_filter "'/ajax" "'/d42/ajax";
        sub_filter "'/superset" "'/d42/superset";
        
        # 3. Fix form actions and redirects in HTML
        sub_filter 'action="/admin' 'action="/d42/admin';
        sub_filter 'href="/admin' 'href="/d42/admin';
        sub_filter 'src="/static' 'src="/d42/static';
        sub_filter 'href="/superset' 'href="/d42/superset';
        
        # 4. Force HTTPS and fix old references
        sub_filter 'http://eins0fx.aequify.gg' 'https://eins0fx.aequify.gg';
        sub_filter '/vmapp/' '/d42/';
        
        # 5. CRITICAL: Prevent double d42 in content
        sub_filter '/d42/d42/' '/d42/';
        
        sub_filter_once off;
        
        # Proxy timeouts (longer for dashboards)
        proxy_connect_timeout 60s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;
    }
    
    # Static assets - Direct proxy (no session needed)
    location ~ ^/(static|tenant)/ {
        proxy_pass https://192.168.1.42$request_uri;
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        proxy_set_header Host $host;
        proxy_set_header Accept-Encoding "";
    }
    
    # VM App - Service 2 (HTTPS, same VM different service)
    location /localapp/ {
        proxy_pass https://192.168.1.42/;
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Static sample app
    location /staticapp/ {
        alias /srv/static-sample/;
        try_files $uri $uri/ =404;
    }
}
```

### Step 3: Enable the Configuration

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/multi-app /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # Remove default site

# Test configuration
sudo nginx -t

# Start nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Step 4: Set Up Ngrok

```bash
# Install ngrok (official method)
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
  && echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" \
  | sudo tee /etc/apt/sources.list.d/ngrok.list \
  && sudo apt update \
  && sudo apt install ngrok

# Configure auth token
ngrok config add-authtoken 1ohNk9Nm8kVlwiceSD1MlaZVn9Y_aH9E2HzjcUaEYE6hqXnM

# Start ngrok tunnel
ngrok http 80 --domain=eins0fx.aequify.gg
```

## Key Technical Solutions Implemented

### 1. Base Tag Injection (Critical Fix)

The most important fix was injecting a `<base>` tag into all HTML responses:

```nginx
sub_filter '<head>' '<head><base href="https://eins0fx.aequify.gg/d42/">';
```

**Why this works:**
- Forces **all relative URLs** in HTML and JavaScript to resolve with `/d42/` prefix
- Eliminates need to rewrite every relative reference manually
- Handles dynamically generated URLs in JavaScript
- Prevents POST data loss from redirects

### 2. Mixed Content Resolution

**Problem**: HTTPS page trying to load HTTP resources
**Solution**: Content Security Policy with `upgrade-insecure-requests`

```nginx
add_header Content-Security-Policy "upgrade-insecure-requests; ...";
```

### 3. Superset Analytics Integration

**Problem**: Device42 Insights+ page uses embedded Superset dashboards
**Solution**: Route `/superset/` through `/d42/` for proper authentication

```nginx
location /superset/ {
    return 301 /d42$request_uri;
}
```

### 4. Double Redirect Prevention

**Problem**: Login flows creating `/d42/d42/` URLs
**Solution**: Catch and fix double prefixes

```nginx
location ~ ^/d42/d42/(.*)$ {
    return 301 /d42/$1;
}
```

### 5. Session Context Preservation

**Problem**: Direct absolute path access loses session context
**Solution**: Redirect all absolute paths to `/d42/` equivalent

```nginx
location ~ ^/(admin|api|jsi18n|internal|media|ajax|login|logout)(/.*)?$ {
    return 301 /d42$request_uri;
}
```

## Troubleshooting Guide

### Issue: Application Redirects to Wrong URL

**Symptoms**: App redirects to `https://eins0fx.aequify.gg/admin/core/home` instead of `https://eins0fx.aequify.gg/d42/admin/core/home`

**Root Cause**: Application generates absolute redirects without subpath awareness

**Solution**: Implemented in our configuration
1. Base tag injection forces relative URL resolution
2. `proxy_redirect` fixes Location headers  
3. Absolute path redirects catch escaped references

### Issue: Static Assets Not Loading

**Symptoms**: 404 errors for CSS, JS, images

**Root Cause**: Assets referenced with absolute paths

**Solution**: 
```nginx
location ~ ^/(static|tenant)/ {
    proxy_pass https://192.168.1.42$request_uri;
    # Direct proxy - no rewriting needed
}
```

### Issue: Mixed Content Errors

**Symptoms**: HTTPS page blocked from loading HTTP resources

**Root Cause**: JavaScript making HTTP requests from HTTPS page

**Solutions Implemented**:
1. CSP `upgrade-insecure-requests` directive
2. Content rewriting: `sub_filter 'http://eins0fx.aequify.gg' 'https://eins0fx.aequify.gg'`
3. Base tag forcing HTTPS context

### Issue: Login Loop After Logout

**Symptoms**: Double `/d42/` prefixes in URLs after login

**Root Cause**: Application redirects adding prefix to already-prefixed URLs

**Solution**: 
```nginx
location ~ ^/d42/d42/(.*)$ {
    return 301 /d42/$1;
}
```

### Issue: Device Detail Pages Empty

**Symptoms**: Page loads but content areas are blank

**Root Cause**: POST requests being redirected, losing form data

**Solution**: Base tag injection prevents redirects by ensuring correct URLs from start

### Issue: Superset Dashboards Not Loading

**Symptoms**: 404 errors for `/superset/dashboard/` requests

**Root Cause**: Missing routing for Superset analytics

**Solution**: 
```nginx
location /superset/ {
    return 301 /d42$request_uri;
}
```

### Issue: Content Security Policy Blocking Stylesheets

**Symptoms**: Console errors like "Refused to load the stylesheet because it violates the following Content Security Policy directive: style-src..."

**Root Cause**: Device42 sends its own restrictive CSP header, conflicting with nginx's permissive CSP

**Solution**: Hide Device42's CSP header and use only nginx's permissive policy
```nginx
# In /d42/ location block
proxy_hide_header Content-Security-Policy;

# At server level - permissive style-src
add_header Content-Security-Policy "... style-src 'self' 'unsafe-inline' data: https: http: ...";
```

## Testing Your Setup

### Basic Connectivity Tests

```bash
# Test nginx configuration
sudo nginx -t

# Test local access
curl -I http://localhost/d42/admin/login/

# Test through ngrok
curl -I https://eins0fx.aequify.gg/d42/admin/login/

# Check for base tag injection
curl -s http://localhost/d42/admin/login/ | grep -A2 "<base"
```

### Advanced Testing

```bash
# Monitor nginx logs in real-time
sudo tail -f /var/log/nginx/access.log

# Test POST request handling (should not redirect)
curl -X POST -I http://localhost/d42/admin/login/

# Check cookie path scoping
curl -I http://localhost/d42/admin/login/ | grep "Set-Cookie"

# Test Superset redirect
curl -I http://localhost/superset/dashboard/17/
```

## Performance Optimizations

### Enable Gzip Compression

```nginx
# Add to /etc/nginx/nginx.conf in http block
gzip on;
gzip_types
    text/plain
    text/css
    text/js
    text/xml
    text/javascript
    application/javascript
    application/xml+rss
    application/json;
gzip_proxied any;
```

### Browser Caching for Static Assets

```nginx
location ~ ^/(static|tenant)/ {
    proxy_pass https://192.168.1.42$request_uri;
    proxy_ssl_verify off;
    
    # Cache static assets
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

## Security Enhancements

### Rate Limiting

```nginx
# Add to /etc/nginx/nginx.conf in http block
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

# Add to login location
location ~ /d42/(admin/)?login/ {
    limit_req zone=login burst=3 nodelay;
    # ... rest of configuration
}
```

### SSL/TLS Hardening

```nginx
# Add stricter security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()";
```

## Monitoring and Maintenance

### Log Analysis

```bash
# Monitor successful requests
tail -f /var/log/nginx/access.log | grep " 200 "

# Monitor redirects
tail -f /var/log/nginx/access.log | grep -E " 30[0-9] "

# Monitor errors
tail -f /var/log/nginx/error.log

# Analyze common requests
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -20
```

### Health Checks

```bash
# Create health check script
cat > /usr/local/bin/check-d42-health.sh << 'EOF'
#!/bin/bash
if curl -f -s http://localhost/d42/admin/login/ > /dev/null; then
    echo "✅ D42 proxy healthy"
    exit 0
else
    echo "❌ D42 proxy unhealthy"
    exit 1
fi
EOF

chmod +x /usr/local/bin/check-d42-health.sh

# Add to crontab for monitoring
echo "*/5 * * * * /usr/local/bin/check-d42-health.sh" | crontab -
```

## Final Result

After implementing this configuration, you have:

- ✅ **Device42 Application**: `https://eins0fx.aequify.gg/d42/` → Fully functional with all features
- ✅ **Device Management**: All device detail pages load completely
- ✅ **Insights+ Analytics**: Superset dashboards working perfectly
- ✅ **Authentication Flows**: Login/logout working without loops
- ✅ **Static Assets**: All CSS, JS, images loading correctly
- ✅ **Mixed Content**: Resolved with CSP and content rewriting
- ✅ **Session Management**: Proper cookie scoping and persistence
- ✅ **Error Handling**: Graceful fallbacks and redirects

**Status: ✅ PRODUCTION READY**

The setup successfully handles complex web application proxying with advanced features like base tag injection, content filtering, and embedded analytics support. All major Device42 functionality is working correctly through the reverse proxy.
