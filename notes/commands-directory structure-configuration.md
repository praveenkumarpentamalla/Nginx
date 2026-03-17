

## Basic Nginx Commands

### Concept Explanation

**Simple Terms:**
Think of Nginx commands like the controls of a car. You need to know how to start the engine (start), turn it off (stop), restart it (restart), and change settings while driving (reload). These commands control the Nginx service lifecycle.

**Deep Technical Explanation:**
Nginx uses a master process that controls worker processes. Commands send signals to these processes:
- **SIGTERM**: Graceful shutdown
- **SIGQUIT**: Graceful shutdown (waits for workers to finish)
- **SIGHUP**: Reload configuration (zero downtime)
- **SIGUSR1**: Reopen log files (log rotation)
- **SIGUSR2**: Upgrade binary smoothly

**Process Signal Flow:**
```
nginx -s [command] → Sends signal to master process
                           ↓
                    Master Process
                           ↓
    ┌──────────────┬──────────────┬──────────────┐
    ↓              ↓              ↓              ↓
Worker 1      Worker 2      Worker 3      Worker N
(handles      (handles      (handles      (handles
connections)  connections)  connections)  connections)
```

### Real-World Use Case

**Production Scenario:**
E-commerce platform during Black Friday:
- Can't afford downtime during peak traffic
- Need to update SSL certificates without disruption
- Must rotate logs daily without losing data
- Quick rollback if configuration error occurs

**Command Usage:**
```bash
# Zero-downtime SSL certificate renewal
nginx -s reload

# Log rotation without losing data
mv access.log access.log.old
nginx -s reopen

# Emergency stop if system overloaded
nginx -s quit  # Graceful
nginx -s stop  # Immediate
```

### Configuration Examples

**Systemd Commands (Modern Linux):**
```bash
# Start Nginx
sudo systemctl start nginx

# Stop Nginx
sudo systemctl stop nginx

# Restart (stop then start)
sudo systemctl restart nginx

# Reload (graceful configuration reload)
sudo systemctl reload nginx

# Enable at boot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

**Nginx Binary Commands:**
```bash
# Test configuration before reload
sudo nginx -t
# Output: nginx: configuration file /etc/nginx/nginx.conf test is successful

# Show version and configure arguments
nginx -V

# Reload configuration (graceful)
sudo nginx -s reload

# Stop immediately
sudo nginx -s stop

# Graceful shutdown
sudo nginx -s quit

# Reopen log files
sudo nginx -s reopen
```

**Init Script (Older Systems):**
```bash
# /etc/init.d/nginx script
#!/bin/bash
case "$1" in
    start)
        /usr/sbin/nginx
        ;;
    stop)
        /usr/sbin/nginx -s stop
        ;;
    restart)
        /usr/sbin/nginx -s stop
        sleep 1
        /usr/sbin/nginx
        ;;
    reload)
        /usr/sbin/nginx -s reload
        ;;
    status)
        if pgrep -x "nginx" > /dev/null; then
            echo "Nginx is running"
        else
            echo "Nginx is stopped"
        fi
        ;;
esac
```

### What Happens During Reload

**Zero-Downtime Reload Process:**
```
Before Reload:
┌─────────────────┐
│ Master (PID 123)│
├─────────────────┤
│ Worker (PID 124)│─── Handling connections
│ Worker (PID 125)│─── Handling connections
└─────────────────┘

Reload Command: nginx -s reload
         ↓
┌─────────────────┐
│ Master (PID 123)│─── Reads new config
└─────────────────┘
         ↓
┌─────────────────┐
│ Master (PID 123)│─── Creates new workers
└─────────────────┘
         ↓
┌─────────────────┐
│ Master (PID 123)│
├─────────────────┤
│ Old Worker 124  │─── Finishes connections
│ Old Worker 125  │─── Finishes connections
│ New Worker 126  │─── Handles new connections
│ New Worker 127  │─── Handles new connections
└─────────────────┘
         ↓
Old workers exit when connections complete
```

### Common Mistakes

1. **Not testing configuration before reload:**
   ```bash
   # WRONG - risky!
   sudo nginx -s reload
   
   # RIGHT - test first
   sudo nginx -t && sudo nginx -s reload
   ```

2. **Killing processes directly:**
   ```bash
   # WRONG - can cause file corruption
   sudo kill -9 $(pgrep nginx)
   
   # RIGHT - use Nginx commands
   sudo nginx -s quit
   ```

3. **Confusing restart with reload:**
   - Restart = Stop + Start (downtime)
   - Reload = Graceful config reload (no downtime)

### Best Practices

1. **Create aliases for common commands:**
   ```bash
   # Add to ~/.bashrc
   alias ngt='sudo nginx -t'
   alias ngr='sudo nginx -t && sudo nginx -s reload'
   alias ngst='sudo systemctl status nginx'
   ```

2. **Monitor reload events:**
   ```bash
   # Watch error log during reload
   tail -f /var/log/nginx/error.log | grep -i "reload"
   ```

3. **Version control configurations:**
   ```bash
   # Before making changes
   sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
   ```

### Debugging Tips

```bash
# Check if Nginx is running
ps aux | grep nginx

# Check which ports are listening
sudo ss -tlnp | grep nginx

# Trace signals sent to Nginx
sudo strace -p $(pgrep -o nginx) -e trace=signal

# Check recent logs for errors
sudo journalctl -u nginx --since "5 minutes ago"

# Monitor configuration test output
sudo nginx -T  # Test and print configuration
```

### Interview Questions

**Beginner:**
Q: What's the difference between `nginx -s stop` and `nginx -s quit`?
A: `stop` terminates immediately, dropping connections. `quit` completes current requests before exiting.

**Intermediate:**
Q: How does Nginx achieve zero-downtime reloads?
A: Master process reads new config, forks new workers, signals old workers to finish current connections, then exits old workers. New connections go to new workers immediately.

**Advanced:**
Q: Explain the signal handling mechanism in Nginx and how you'd implement a custom signal handler.
A: Nginx uses standard POSIX signals. Master process catches signals and coordinates workers. Custom handlers would require modifying source: register signal handler in `ngx_process.c`, add to `ngx_signal_t` array, and implement logic in master process loop.

### Hands-on Tasks

**Task 1: Practice Commands**
1. Start Nginx, verify it's running
2. Make a harmless config change
3. Test configuration
4. Reload gracefully
5. Check that old process IDs changed

**Task 2: Simulate Zero-Downtime**
1. Start Nginx
2. Run a long download (large file)
3. During download, reload configuration
4. Verify download continues uninterrupted

---

## Topic 5: Default Directory Structure

### Concept Explanation

**Simple Terms:**
Nginx organizes its files like a well-organized filing cabinet. Configuration files are blueprints, logs are activity records, web files are the actual content. Understanding where everything lives is crucial for administration.

**Deep Technical Explanation:**
The directory structure varies by installation method and OS, but follows FHS (Filesystem Hierarchy Standard) principles:

```
/etc/nginx/           # Configuration files (blueprints)
/var/log/nginx/       # Log files (activity records)
/var/www/            # Web content (actual content)
/usr/sbin/nginx      # Binary executable (engine)
/usr/lib/nginx/      # Modules (plugins)
/run/nginx.pid       # Process ID (engine status)
```

**Directory Tree:**
```
/
├── etc/
│   └── nginx/
│       ├── nginx.conf                 # Main configuration
│       ├── mime.types                  # File type mappings
│       ├── fastcgi_params              # FastCGI parameters
│       ├── proxy_params                 # Proxy parameters
│       ├── scgi_params                  # SCGI parameters
│       ├── uwsgi_params                 # uWSGI parameters
│       ├── conf.d/                      # Global configs
│       │   ├── default.conf
│       │   └── custom.conf
│       ├── sites-available/             # Available sites (Debian)
│       │   ├── default
│       │   └── example.com
│       ├── sites-enabled/                # Enabled sites (symlinks)
│       │   ├── default -> ../sites-available/default
│       │   └── example.com -> ../sites-available/example.com
│       ├── modules-available/            # Available modules
│       └── modules-enabled/               # Enabled modules (symlinks)
│
├── var/
│   ├── log/
│   │   └── nginx/
│   │       ├── access.log
│   │       ├── error.log
│   │       └── access.log.1 (rotated)
│   └── www/
│       ├── html/                         # Default web root
│       │   ├── index.html
│       │   └── 50x.html
│       └── example.com/                   # Custom site
│           ├── index.html
│           ├── css/
│           └── images/
│
├── usr/
│   ├── sbin/
│   │   └── nginx                         # Binary
│   └── lib/
│       └── nginx/
│           └── modules/                    # Dynamic modules
│
└── run/
    └── nginx.pid                          # Process ID file
```

### Real-World Use Case

**Production Scenario:**
A hosting company manages 100+ websites:
- Each site needs separate configuration
- Easy enable/disable without deleting files
- Consistent structure across servers
- Quick troubleshooting with standard log locations

**Solution:** Use sites-available/enabled pattern:
```bash
# Create new site
sudo cp /etc/nginx/sites-available/template /etc/nginx/sites-available/client1.com
sudo nano /etc/nginx/sites-available/client1.com

# Enable site
sudo ln -s /etc/nginx/sites-available/client1.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload

# Disable site
sudo rm /etc/nginx/sites-enabled/client1.com
sudo nginx -t && sudo nginx -s reload
```

### Configuration Examples

**Main Configuration File with Includes:**
```nginx
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    # Basic settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    # Gzip compression
    gzip on;
    
    # Virtual host configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

**Understanding Each Path:**
```bash
# Find Nginx binary location
which nginx
# /usr/sbin/nginx

# Find configuration files
nginx -t 2>&1 | grep configuration
# configuration file /etc/nginx/nginx.conf test is successful

# Find web root
grep root /etc/nginx/sites-enabled/*
# /etc/nginx/sites-enabled/default: root /var/www/html;

# Find logs
grep access_log /etc/nginx/nginx.conf
# access_log /var/log/nginx/access.log;
```

### Common Mistakes

1. **Editing wrong file:**
   ```bash
   # WRONG - editing symlinked file in sites-enabled
   sudo nano /etc/nginx/sites-enabled/example.com
   
   # RIGHT - edit in sites-available
   sudo nano /etc/nginx/sites-available/example.com
   ```

2. **Deleting instead of disabling:**
   ```bash
   # WRONG - deletes the original
   sudo rm /etc/nginx/sites-enabled/example.com
   
   # RIGHT - removes symlink only
   sudo unlink /etc/nginx/sites-enabled/example.com
   ```

3. **Wrong log permissions:**
   ```bash
   # Fix log permissions
   sudo chown -R www-data:adm /var/log/nginx
   sudo chmod 755 /var/log/nginx
   ```

### Best Practices

1. **Use include directives for modularity:**
   ```nginx
   # Separate configurations by purpose
   include /etc/nginx/security.conf;
   include /etc/nginx/caching.conf;
   include /etc/nginx/ssl-params.conf;
   ```

2. **Standard naming conventions:**
   ```bash
   # Domain-based config files
   sites-available/example.com.conf
   sites-available/api.example.com.conf
   
   # Number for ordering
   conf.d/01-main.conf
   conf.d/02-security.conf
   conf.d/99-custom.conf
   ```

3. **Document your structure:**
   ```bash
   # Create README in /etc/nginx/
   cat > /etc/nginx/README << 'EOF'
   Directory Structure:
   - conf.d/ - Global configuration fragments
   - sites-available/ - All virtual host configurations
   - sites-enabled/ - Symlinks to enabled sites
   - ssl/ - SSL certificates and keys
   EOF
   ```

### Debugging Tips

```bash
# Find all config files Nginx is using
nginx -T | grep -E "^# configuration file"

# Check where a specific directive is set
grep -r "root " /etc/nginx/

# Monitor which files are accessed during reload
sudo inotifywait -m /etc/nginx/ -e access

# Find all log files
find /var/log/nginx -type f

# Trace file operations
sudo strace -e openat -p $(pgrep -o nginx) 2>&1 | grep conf
```

### Interview Questions

**Beginner:**
Q: What's the difference between conf.d/ and sites-enabled/?
A: conf.d/ is for global configuration snippets affecting all sites. sites-enabled/ contains symlinks to specific virtual host configurations in sites-available/.

**Intermediate:**
Q: Why does Debian/Ubuntu use sites-available and sites-enabled pattern?
A: It provides a safe way to enable/disable sites without deleting configurations. Just create symlinks to enable, remove symlinks to disable. Original configs remain in sites-available.

**Advanced:**
Q: How would you implement a custom directory structure for a multi-tenant application?
A: Create tenant-specific directories with dynamic includes. Use variables in paths. Implement with map blocks for tenant detection. Example:
```nginx
map $host $tenant {
    default tenant1;
    tenant2.example.com tenant2;
}
root /var/www/tenants/$tenant;
include /etc/nginx/tenants/$tenant/*.conf;
```

### Hands-on Tasks

**Task 1: Explore Your Installation**
1. Run `nginx -V` and note compile-time paths
2. List all Nginx directories: `ls -la /etc/nginx/`
3. Follow symlinks in sites-enabled
4. Locate default index.html

**Task 2: Create Custom Structure**
1. Create sites-available/site1.conf
2. Enable it via symlink
3. Create conf.d/security-headers.conf
4. Verify both are included in main config

---

## Topic 6: Basic Configuration File (nginx.conf)

### Concept Explanation

**Simple Terms:**
nginx.conf is like a recipe book. It contains instructions organized in sections (contexts). Each instruction (directive) tells Nginx how to behave. The file is read from top to bottom, with sections nested inside each other.

**Deep Technical Explanation:**
Nginx configuration uses a declarative, hierarchical structure:
- **Directives**: Configuration options with values (e.g., `worker_processes 4;`)
- **Contexts**: Directive blocks (e.g., `http { ... }`)
- **Simple directives**: End with semicolon
- **Block directives**: Have nested directives in braces

**Configuration Hierarchy:**
```
main (global context)
├── events { }
├── http {
│   ├── upstream { }
│   ├── server {
│   │   ├── location / { }
│   │   ├── location /api { }
│   │   └── location ~ \.php$ { }
│   └── server {
│       └── location / { }
└── stream {
    └── server { }
```

### Real-World Use Case

**Production Scenario:**
Microservices architecture with:
- Multiple API services
- Static frontend
- WebSocket connections
- Rate limiting
- SSL termination

**Complete Production Configuration:**
```nginx
# /etc/nginx/nginx.conf
# ===== GLOBAL CONTEXT =====

# User to run worker processes
user www-data;

# Auto-detect CPU cores
worker_processes auto;

# PID file location
pid /run/nginx.pid;

# Include dynamic modules
include /etc/nginx/modules-enabled/*.conf;

# ===== EVENTS CONTEXT =====
events {
    # Max connections per worker
    worker_connections 4096;
    
    # Use epoll on Linux
    use epoll;
    
    # Accept all connections at once
    multi_accept on;
}

# ===== HTTP CONTEXT =====
http {
    # ===== Basic Settings =====
    # Include MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Optimize file serving
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Connection timeouts
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # Client settings
    client_max_body_size 10M;
    client_body_timeout 12;
    client_header_timeout 12;
    
    # ===== Logging =====
    # Log format with additional fields
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    '$request_time $upstream_response_time';
    
    # JSON log format for ELK stack
    log_format json escape=json '{'
        '"time_local":"$time_local",'
        '"remote_addr":"$remote_addr",'
        '"remote_user":"$remote_user",'
        '"request":"$request",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"http_referrer":"$http_referer",'
        '"http_user_agent":"$http_user_agent",'
        '"http_x_forwarded_for":"$http_x_forwarded_for"'
    '}';
    
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;
    error_log /var/log/nginx/error.log warn;
    
    # ===== Compression =====
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml application/atom+xml image/svg+xml;
    
    # ===== SSL Settings =====
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # ===== Upstream Definitions =====
    upstream backend {
        # Load balancing method
        least_conn;
        
        # Backend servers
        server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
        server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
        server 10.0.0.3:3000 backup;  # Backup server
        
        # Keepalive connections
        keepalive 32;
    }
    
    upstream api {
        server 10.0.0.4:8080 weight=3;
        server 10.0.0.5:8080 weight=2;
        server 10.0.0.6:8080 weight=1;
    }
    
    # ===== Rate Limiting Zones =====
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    # ===== Cache Zones =====
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m 
                     max_size=1g inactive=60m use_temp_path=off;
    
    # ===== Server Blocks =====
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Directive Categories

1. **Essential Directives:**
   ```nginx
   user www-data;          # Security: run as non-root
   worker_processes auto;  # Performance: match CPU cores
   error_log ...;          # Debugging: know what's wrong
   pid ...;                # Management: know process ID
   include ...;            # Organization: modular configs
   ```

2. **Performance Directives:**
   ```nginx
   sendfile on;           # Zero-copy file transfer
   tcp_nopush on;         # Optimize packet sending
   tcp_nodelay on;        # Disable Nagle's algorithm
   keepalive_timeout 65;  # Connection reuse
   gzip on;               # Compression
   ```

3. **Security Directives:**
   ```nginx
   client_max_body_size 10M;  # Prevent large uploads DoS
   client_body_timeout 12;    # Timeout for slow clients
   limit_req_zone ...;        # Rate limiting
   ssl_protocols ...;         # Secure TLS versions
   ```

### Configuration Parsing Process

**How Nginx Reads Config:**
```
1. Start main context
2. Read file line by line
3. Parse directives (semicolon-terminated)
4. Enter block contexts (braces)
5. Build configuration tree in memory
6. Validate directive values
7. Apply configuration or error if invalid

Memory Structure After Parsing:
┌─────────────────────┐
│ ngx_cycle_t         │
│   - conf_ctx[]      │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ ngx_http_conf_ctx_t │
│   - main_conf[]     │  → HTTP main config
│   - srv_conf[]      │  → Server configs
│   - loc_conf[]      │  → Location configs
└─────────────────────┘
```

### Common Mistakes

1. **Missing semicolons:**
   ```nginx
   # WRONG - will cause error
   worker_processes auto
   
   # RIGHT
   worker_processes auto;
   ```

2. **Duplicate directives:**
   ```nginx
   # WRONG - last one wins, may not be intended
   worker_processes 4;
   worker_processes 8;  # This overrides
   
   # RIGHT - only define once
   worker_processes 8;
   ```

3. **Wrong context placement:**
   ```nginx
   # WRONG - proxy_pass in http context
   http {
       proxy_pass http://backend;
   }
   
   # RIGHT - in location block
   http {
       server {
           location / {
               proxy_pass http://backend;
           }
       }
   }
   ```

### Best Practices

1. **Use include for organization:**
   ```nginx
   # Main config stays clean
   http {
       include snippets/*.conf;
       include conf.d/*.conf;
       include sites-enabled/*;
   }
   ```

2. **Comment important decisions:**
   ```nginx
   # WHY: CPU optimized for encryption
   worker_processes 8;  # 8 cores with AES-NI
   
   # WHY: Rate limit for security
   limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
   ```

3. **Use variables wisely:**
   ```nginx
   # Define at top
   set $asset_version "1.0.0";
   
   # Use throughout
   location /static/ {
       alias /var/www/static/$asset_version/;
   }
   ```

### Debugging Tips

```bash
# Test syntax with context info
nginx -t
# nginx: [emerg] "server" directive is not allowed here

# Print full configuration with line numbers
nginx -T | grep -n "^"

# Check configuration tree
nginx -T | grep -A 10 -B 10 "server_name"

# Validate with specific context
nginx -c /etc/nginx/nginx.conf -t

# Debug level logging
error_log /var/log/nginx/error.log debug;
```

### Interview Questions

**Beginner:**
Q: What are contexts in Nginx configuration?
A: Contexts are blocks that group related directives. Main contexts: events, http, server, location. Directives inherit from outer to inner contexts.

**Intermediate:**
Q: Explain directive inheritance in Nginx.
A: Directives are inherited from outer to inner contexts unless overridden. For example, root in http applies to all servers unless a server block specifies its own root.

**Advanced:**
Q: How would you implement dynamic configuration based on environment variables?
A: Use env directive in main context, then access via perl_set or lua. Or use template system with envsubst at startup. Example:
```nginx
env DATABASE_HOST;
http {
    perl_set $db_host 'sub { return $ENV{"DATABASE_HOST"}; }';
}
```

### Hands-on Tasks

**Task 1: Build from Scratch**
1. Create minimal nginx.conf
2. Add http, server, location blocks
3. Add logging with custom format
4. Add gzip compression
5. Test each addition

**Task 2: Modular Configuration**
1. Split config into logical files
2. Create snippets for common settings
3. Use include to assemble them
4. Verify with nginx -T

---

## Topic 7: Serving Static Files

### Concept Explanation

**Simple Terms:**
Serving static files means delivering unchanging content like HTML, CSS, images, and JavaScript directly from disk. Nginx excels at this because it reads files and sends them without complex processing.

**Deep Technical Explanation:**
Static file serving in Nginx uses:
- **Zero-copy I/O**: `sendfile` transfers data directly between file descriptor and socket without copying to user space
- **Kernel space optimization**: Data goes from disk cache → NIC directly
- **Etag and caching headers**: Browser caching support
- **Efficient file handling**: Open file cache, directory indexing

**Comparison: Nginx vs Apache for Static Files:**
```
Apache:   read() → buffer → write() → socket  (4 context switches)
Nginx:    sendfile() → socket                 (2 context switches)

Performance Impact:
10,000 concurrent requests:
Apache:  ~800 MB RAM, 80% CPU, 5k req/sec
Nginx:   ~80 MB RAM, 30% CPU, 15k req/sec
```

### Real-World Use Case

**Production Scenario:**
CDN edge server serving millions of images:
- Images stored on SSD
- Browser caching for 1 year
- Automatic compression
- Directory listing disabled for security
- Custom error pages

### Configuration Examples

**Basic Static File Server:**
```nginx
server {
    listen 80;
    server_name static.example.com;
    
    # Root directory for all files
    root /var/www/static;
    
    # Default file to serve
    index index.html index.htm;
    
    # Handle all requests
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Specific location for images
    location /images/ {
        alias /data/images/;
        
        # Cache images for 30 days
        expires 30d;
        
        # Add cache control headers
        add_header Cache-Control "public, immutable";
        
        # Prevent direct access to directories
        autoindex off;
    }
    
    # CSS and JavaScript with different caching
    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public";
        
        # Gzip compress these types
        gzip_static on;
    }
    
    # Error pages
    error_page 404 /custom_404.html;
    error_page 500 502 503 504 /custom_50x.html;
    
    location = /custom_404.html {
        root /var/www/errors;
        internal;  # Only accessible via error_page
    }
}
```

**Advanced Static File Optimization:**
```nginx
http {
    # Open file cache
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    server {
        listen 80;
        server_name cdn.example.com;
        
        root /var/www/cdn;
        
        # Optimize sending
        sendfile on;
        sendfile_max_chunk 1m;
        tcp_nopush on;
        
        # Output buffering
        output_buffers 32 32k;
        postpone_output 1460;
        
        # Static files handling
        location /assets/ {
            alias /var/www/assets/;
            
            # Versioned assets - cache forever
            location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                
                # Try compressed version first
                gzip_static on;
                
                # Remove ETag since we use far-future expires
                etag off;
            }
            
            # HTML files - shorter cache
            location ~* \.html?$ {
                expires 1h;
                add_header Cache-Control "public, must-revalidate";
            }
        }
        
        # Directory listing (only for specific location)
        location /downloads/ {
            alias /var/www/downloads/;
            autoindex on;
            autoindex_format html;
            autoindex_localtime on;
            
            # Limit access
            satisfy all;
            allow 192.168.1.0/24;
            deny all;
        }
        
        # Serve pre-compressed files
        location /static/ {
            gzip_static always;
            gunzip on;  # Fallback for clients without gzip
            
            location ~* \.(svgz)$ {
                # Don't double compress
                gzip off;
                add_header Content-Encoding gzip;
            }
        }
    }
}
```

**Understanding Each Directive:**

| Directive | Purpose | Production Value |
|-----------|---------|------------------|
| `sendfile` | Zero-copy file transfer | `on` |
| `tcp_nopush` | Optimize packet sending | `on` |
| `expires` | Browser caching duration | `30d`, `1y` |
| `open_file_cache` | Cache file descriptors | `max=1000 inactive=20s` |
| `gzip_static` | Serve pre-compressed files | `on` |
| `autoindex` | Directory listing | `off` (security) |

### Common Mistakes

1. **Not setting proper cache headers:**
   ```nginx
   # WRONG - no caching, clients request every time
   location /images/ {
       alias /data/images/;
   }
   
   # RIGHT - cache for performance
   location /images/ {
       alias /data/images/;
       expires 30d;
       add_header Cache-Control "public";
   }
   ```

2. **Wrong file permissions:**
   ```bash
   # Check permissions
   ls -la /var/www/static/
   
   # Fix if needed
   sudo chown -R www-data:www-data /var/www/static/
   sudo find /var/www/static/ -type f -exec chmod 644 {} \;
   sudo find /var/www/static/ -type d -exec chmod 755 {} \;
   ```

3. **Path traversal vulnerability:**
   ```nginx
   # WRONG - alias without trailing slash can be dangerous
   location /images {
       alias /data/images;  # /images../etc/passwd possible
   }
   
   # RIGHT - always use trailing slash
   location /images/ {
       alias /data/images/;
   }
   ```

### Best Practices

1. **Use CDN for production:**
   ```nginx
   location /static/ {
       # Cache at edge
       expires max;
       
       # Allow CDN to cache
       add_header Cache-Control "public, s-maxage=31536000";
       
       # Vary by encoding
       add_header Vary Accept-Encoding;
   }
   ```

2. **Implement cache busting:**
   ```nginx
   # In HTML template
   <link rel="stylesheet" href="/css/style.abc123.css">
   
   # In Nginx
   location ~* \.(css|js)\.[a-f0-9]+\.(css|js)$ {
       expires 1y;
       add_header Cache-Control "public, immutable";
       
       # Extract original filename
       rewrite ^(.+)\.[a-f0-9]+\.(css|js)$ $1.$2 break;
   }
   ```

3. **Monitor cache hit ratio:**
   ```nginx
   # Add to log format
   log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for" '
                   'sent_http_cache_control="$sent_http_cache_control"';
   ```

### Debugging Tips

```bash
# Check if files are accessible
curl -I http://localhost/image.jpg

# Verify cache headers
curl -I http://localhost/style.css | grep -i cache

# Test with different accept-encoding
curl -H "Accept-Encoding: gzip" -I http://localhost/script.js

# Monitor file access
sudo tail -f /var/log/nginx/access.log | grep "GET /images/"

# Check open file cache
sudo kill -SIGUSR1 $(pgrep -o nginx)  # Reopen logs
sudo tail -f /var/log/nginx/error.log | grep "open file cache"
```

### Interview Questions

**Beginner:**
Q: What's the difference between root and alias?
A: root appends location to root path, alias replaces location part. Example: root /var/www + /images/ → /var/www/images/; alias /data/images/ + /images/ → /data/images/

**Intermediate:**
Q: How does sendfile work internally and why is it faster?
A: sendfile transfers data between file descriptors in kernel space without copying to user space. Traditional read/write: disk → kernel buffer → user buffer → kernel socket buffer → NIC. sendfile: disk → kernel buffer → NIC directly.

**Advanced:**
Q: Design a high-performance static file serving architecture for global users.
A: Multi-layer approach:
- Edge: CDN with PoPs worldwide
- Origin: Nginx with SSD, open_file_cache
- Optimization: sendfile, tcp_nopush, gzip_static
- Caching: Immutable cache for versioned assets
- Monitoring: Cache hit ratio, latency by region

### Hands-on Tasks

**Task 1: Build Photo Gallery**
1. Create directory with sample images
2. Configure Nginx to serve them
3. Add caching headers (30 days for images, 1 day for HTML)
4. Test with curl to verify headers

**Task 2: Performance Testing**
1. Create 1000 small files
2. Benchmark with: `ab -n 10000 -c 100 http://localhost/`
3. Enable sendfile and retest
4. Compare results

---

## Topic 8: Understanding Server and Location Blocks

### Concept Explanation

**Simple Terms:**
Server blocks are like different apartments in a building, each with its own address (domain name). Location blocks are like rooms within an apartment, handling specific types of requests (like "/images" or "/api").

**Deep Technical Explanation:**
- **Server blocks**: Virtual hosts that listen on IP:port combinations and match against `Host` header
- **Location blocks**: Define how to process specific URI paths within a server
- **Matching algorithm**: Nginx selects server first, then most specific location

**Server Selection Algorithm:**
```
1. Check listen directive (IP:port)
2. Compare server_name against Host header:
   - Exact match
   - Longest wildcard starting with *
   - Longest wildcard ending with *
   - First matching regex
3. If no match, use default server
```

**Location Selection Algorithm:**
```
For a given URI /images/logo.png:
1. Check prefix locations (literal strings)
2. Remember longest matching prefix
3. Check regex locations (~ and ~*)
4. Use first matching regex, otherwise longest prefix

Priority:
= (exact) → ^~ (prefix, stop regex) → ~ (regex) → (plain prefix)
```

### Real-World Use Case

**Production Scenario:**
SaaS platform with multiple services:
- Main website (example.com)
- API service (api.example.com)
- Admin panel (admin.example.com)
- Static assets (static.example.com)
- Different paths handled differently (/api → Node.js, /admin → Python)

### Configuration Examples

**Multiple Server Blocks:**
```nginx
# Main website
server {
    listen 80;
    server_name example.com www.example.com;
    
    root /var/www/main;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Blog at /blog
    location /blog/ {
        alias /var/www/blog/;
        
        # PHP handling for blog
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
    }
}

# API server
server {
    listen 80;
    server_name api.example.com;
    
    location / {
        # Proxy to Node.js API
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # API documentation
    location /docs/ {
        alias /var/www/api-docs/;
        autoindex on;
    }
    
    # API status endpoint (internal only)
    location /status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}

# Admin panel with IP restriction
server {
    listen 80;
    server_name admin.example.com;
    
    root /var/www/admin;
    index index.php;
    
    # Restrict to office IPs
    location / {
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;
        
        try_files $uri $uri/ /index.php?$args;
    }
    
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}

# Static assets server
server {
    listen 80;
    server_name static.example.com;
    
    root /var/www/static;
    
    # Optimize static file serving
    sendfile on;
    sendfile_max_chunk 1m;
    
    # Cache all static assets
    location / {
        expires 1y;
        add_header Cache-Control "public, immutable";
        
        # Try pre-compressed files
        gzip_static on;
    }
    
    # No access to dotfiles
    location ~ /\. {
        deny all;
        return 404;
    }
}
```

**Advanced Location Examples:**
```nginx
server {
    listen 80;
    server_name example.com;
    
    root /var/www/html;
    
    # 1. EXACT MATCH - highest priority
    location = / {
        # Special handling for homepage
        add_header X-Homepage "true";
        try_files /homepage.html =404;
    }
    
    # 2. PREFIX MATCH with ^~ (stops regex checking)
    location ^~ /static/ {
        # Static files - no need for regex
        alias /var/www/static/;
        expires max;
    }
    
    # 3. REGEX MATCH - case insensitive
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        # All static file extensions
        expires 30d;
        add_header Cache-Control "public";
        
        # Log these separately
        access_log /var/log/nginx/static_access.log;
    }
    
    # 4. REGEX MATCH - case sensitive
    location ~ \.php$ {
        # PHP files
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
    
    # 5. PREFIX MATCH - regular
    location /api/ {
        # API requests
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        
        # Rate limit API
        limit_req zone=api burst=20;
    }
    
    # 6. NESTED LOCATIONS
    location /admin/ {
        alias /var/www/admin/;
        
        # Nested location for admin API
        location /admin/api/ {
            proxy_pass http://admin_api;
        }
        
        # Nested location for admin static
        location /admin/static/ {
            alias /var/www/admin-static/;
        }
    }
    
    # 7. LOCATION WITH CONDITIONS
    location /proxy/ {
        # Conditional proxy based on user agent
        if ($http_user_agent ~* "mobile") {
            set $backend "mobile_backend";
        }
        
        proxy_pass http://$backend;
    }
    
    # 8. NAMED LOCATION (for internal redirects)
    location @fallback {
        # Internal only, not accessible directly
        return 404 "Resource not found";
    }
    
    location /dynamic/ {
        try_files $uri @fallback;
    }
}
```

**Server Name Variations:**
```nginx
server {
    # Exact name
    server_name example.com;
    
    # Wildcard - starts with
    server_name *.example.com;
    
    # Wildcard - ends with
    server_name www.*;
    
    # Multiple names
    server_name example.com *.example.com www.*;
    
    # Regex
    server_name ~^(www\.)?(?<domain>.+)$;
    
    # Default server
    listen 80 default_server;
}
```

### Matching Priority Examples

**Request URI: /images/logo.png**
```
Location blocks defined:
1. location = /images/logo.png     → Exact match (1st priority)
2. location ^~ /images/             → Prefix with stop (2nd)
3. location ~ \.png$                 → Regex (3rd)
4. location /images/                 → Regular prefix (4th)

Nginx will stop at the first match that applies:
- If #1 matches, use it
- Else if #2 matches, use it and skip regex
- Else check regex (#3)
- Else use longest prefix (#4)
```

### Common Mistakes

1. **Incorrect root in location:**
   ```nginx
   # WRONG - root inside location appends path twice
   location /blog/ {
       root /var/www;  # Looks for /var/www/blog/
   }
   
   # RIGHT - use alias or adjust root
   location /blog/ {
       alias /var/www/blog/;  # Looks for /var/www/blog/
   }
   ```

2. **Regex location order:**
   ```nginx
   # WRONG - first regex catches all
   location ~* \.(php|html)$ {
       # ...
   }
   
   location ~ \.php$ {
       # This will never be reached
   }
   
   # RIGHT - most specific first
   location ~ \.php$ {
       # PHP files only
   }
   
   location ~* \.(html)$ {
       # HTML files
   }
   ```

3. **Missing default server:**
   ```nginx
   # WRONG - no catch-all for invalid hosts
   server {
       listen 80;
       server_name example.com;
   }
   
   # RIGHT - add default server
   server {
       listen 80 default_server;
       server_name _;
       return 444;  # Close connection
   }
   ```

### Best Practices

1. **Organize server blocks by domain:**
   ```nginx
   # /etc/nginx/sites-available/example.com
   server {
       listen 80;
       server_name example.com www.example.com;
       
       include snippets/common.conf;
       include snippets/security.conf;
       
       root /var/www/example.com;
       
       location / {
           include snippets/static.conf;
       }
   }
   ```

2. **Use named locations for fallbacks:**
   ```nginx
   location @maintenance {
       return 503;
       add_header Retry-After 3600;
   }
   
   location / {
       try_files $uri $uri/ @maintenance;
   }
   ```

3. **Document your matching strategy:**
   ```nginx
   # LOCATION ORDERING
   # 1. Exact matches (/favicon.ico)
   # 2. Static files (^~ /static/)
   # 3. PHP files (~ \.php$)
   # 4. Everything else (/)
   ```

### Debugging Tips

```bash
# Test which server block matches
curl -H "Host: example.com" -I http://localhost

# Test location matching
curl -I http://localhost/images/test.jpg

# Enable debug logging for location testing
error_log /var/log/nginx/error.log debug;

# Check which location is used
tail -f /var/log/nginx/access.log | grep "GET /images/"

# Test with different URIs
for uri in / /images/ /images/logo.png /api/test; do
    echo "Testing $uri:"
    curl -s -I http://localhost$uri | grep -i "http/"
done
```

### Interview Questions

**Beginner:**
Q: What's the difference between server_name and listen?
A: listen specifies IP:port to bind to. server_name matches Host header to select virtual host. Multiple domains can share same listen.

**Intermediate:**
Q: Explain the location matching algorithm with examples.
A: 1) Exact match (=) first 2) Preferential prefix (^~) 3) Regex (~) in order 4) Longest prefix. Example: URI /img/test.png - checks exact, then ^~, then regex, then longest prefix.

**Advanced:**
Q: How would you implement A/B testing using server/location blocks?
A: Use split_clients module or map with cookies:
```nginx
split_clients "${remote_addr}${http_user_agent}" $variant {
    50%     "A";
    50%     "B";
}

server {
    location / {
        if ($variant = "A") {
            rewrite ^ /version-a$uri last;
        }
        rewrite ^ /version-b$uri last;
    }
    
    location /version-a {
        alias /var/www/a;
    }
    
    location /version-b {
        alias /var/www/b;
    }
}
```

### Hands-on Tasks

**Task 1: Virtual Hosting**
1. Create 3 server blocks for different domains
2. Each serves different content
3. Test with curl using Host header
4. Add default server for unmatched domains

**Task 2: Location Lab**
1. Create locations with different modifiers
2. Test with various URIs
3. Document which location matches which pattern
4. Create nested location structure

---
