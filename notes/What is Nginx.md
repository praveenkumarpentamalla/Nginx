---

## What is Nginx?

### Concept Explanation

**Simple Terms:**
Nginx (pronounced "Engine-X") is software that handles web traffic. Think of it as a smart traffic controller for the internet. When someone visits your website, Nginx decides what to do with that request - serve a file directly, pass it to an application server, or distribute it across multiple servers.

**Deep Technical Explanation:**
Nginx is a high-performance HTTP server, reverse proxy, load balancer, and API gateway. Originally created by Igor Sysoev in 2004 to solve the C10K problem (handling 10,000+ concurrent connections), it's now the most popular web server on the internet.

**Core Capabilities:**
1. **Web Server**: Serves static files (HTML, CSS, JS, images)
2. **Reverse Proxy**: Forwards client requests to backend servers
3. **Load Balancer**: Distributes traffic across multiple servers
4. **API Gateway**: Routes, throttles, and authenticates API requests
5. **SSL/TLS Terminator**: Handles HTTPS encryption/decryption
6. **Cache Server**: Stores responses to reduce backend load

**Comparison with Apache:**
| Aspect | Nginx | Apache |
|--------|-------|--------|
| Architecture | Event-driven, asynchronous | Process/thread-based |
| Memory Usage | Low (static files) | Higher per connection |
| Concurrency | Handles 10K+ connections easily | Struggles at high concurrency |
| Configuration | Declarative, no reload needed for some changes | .htaccess files, requires reload |
| Dynamic Content | Passes to external processes | Can embed interpreters |

### Real-World Use Case

**Production Scenario:** 
A media streaming platform serves millions of users daily. They use Nginx as:
- **Edge server**: Terminates SSL at the edge
- **Reverse proxy**: Routes API requests to microservices
- **Load balancer**: Distributes traffic across 50 backend servers
- **Cache server**: Caches popular video thumbnails and API responses
- **Rate limiter**: Prevents API abuse by limiting requests per IP

### How It Works Internally

**Request Flow:**
```
Client Request → Nginx Master Process → Worker Process
                                         ↓
                              Event Loop (epoll/kqueue)
                                         ↓
                              Check: Is it static content?
                                    /        \
                                  Yes         No
                                  /            \
                      Serve from disk    Pass to upstream
                                          (proxy_pass)
```

**ASCII Flow Diagram:**
```
┌─────────┐     ┌─────────────────────────────────────┐
│ Client  │────▶│         Nginx Master Process         │
└─────────┘     │  (Reads config, manages workers)     │
                └─────────────────┬───────────────────┘
                                  │
                ┌─────────────────┴───────────────────┐
                ▼                                       ▼
        ┌──────────────┐                      ┌──────────────┐
        │ Worker 1     │                      │ Worker N     │
        │ Event Loop   │                      │ Event Loop   │
        └──────┬───────┘                      └──────┬───────┘
               │                                      │
        ┌──────┴──────────────────────────────────────┴──────┐
        ▼                                                      ▼
┌───────────────┐                                    ┌───────────────┐
│ Static File   │                                    │ Backend       │
│ /var/www      │                                    │ Application   │
└───────────────┘                                    │ (Node.js,     │
                                                      │ Python, etc.) │
                                                      └───────────────┘
```

### Configuration Examples

**Basic Web Server Configuration:**
```nginx
# /etc/nginx/nginx.conf
events {
    worker_connections 1024;  # Max connections per worker
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    server {
        listen 80;                    # Listen on port 80
        server_name example.com;       # Domain name
        
        root /var/www/html;            # Root directory for files
        
        location / {
            try_files $uri $uri/ =404; # Try to serve file, then directory, then 404
        }
        
        location /images/ {
            alias /data/images/;       # Serve from different directory
            expires 30d;               # Cache for 30 days
        }
    }
}
```

**Line-by-Line Explanation:**
- `events { worker_connections 1024; }`: Defines how many connections each worker can handle
- `http { ... }`: Main block for HTTP server configuration
- `include mime.types`: Loads file extension mappings (e.g., .html → text/html)
- `server { listen 80; }`: Creates a virtual server on port 80
- `root /var/www/html`: Base directory for all file lookups
- `location / { try_files ... }`: For requests to root, try file paths in order
- `location /images/ { alias ... }`: Maps `/images/` URL to different directory

### Common Mistakes

1. **Forgetting to test configuration:**
   ```bash
   # Always test before reloading
   nginx -t
   ```

2. **Wrong file permissions:**
   ```bash
   # Nginx runs as www-data user typically
   chown -R www-data:www-data /var/www/html
   ```

3. **Missing mime.types:**
   ```nginx
   # Without this, CSS files might be served as text/plain
   include /etc/nginx/mime.types;
   ```

### Best Practices

1. **Follow the principle of least privilege**: Run Nginx as non-root user
2. **Keep configurations modular**: Use `include` statements
3. **Version control your configs**: Store in Git
4. **Use environment variables** in Docker environments
5. **Monitor with metrics**: Enable status page or use monitoring tools

### Debugging Tips

```bash
# Check if Nginx is running
ps aux | grep nginx

# Check configuration syntax
nginx -t

# View error logs
tail -f /var/log/nginx/error.log

# Test with curl
curl -I http://localhost

# Check open ports
ss -tlnp | grep 80
```

### Interview Questions

**Beginner:**
Q: What's the difference between Nginx and Apache?
A: Nginx uses event-driven architecture handling many connections with few threads, while Apache uses process/thread-per-connection model. Nginx better for static content and high concurrency.

**Intermediate:**
Q: Explain the master-worker process model in Nginx.
A: Master process reads config, binds to ports, and spawns workers. Workers do actual request processing. This model allows zero-downtime reloads and efficient CPU usage.

**Advanced:**
Q: How does Nginx handle the C10K problem internally?
A: Nginx uses asynchronous, non-blocking I/O with event notification mechanisms (epoll on Linux, kqueue on BSD). Instead of creating threads per connection, it processes connections in a single-threaded event loop, switching context only when data is available.

### Hands-on Tasks

**Task 1: Basic Web Server Setup**
1. Install Nginx
2. Create a custom index.html
3. Configure a server block for a domain (use localhost)
4. Test serving static files
5. Check logs to verify access

**Task 2: Multiple Sites**
1. Create two different websites in `/var/www/site1` and `/var/www/site2`
2. Configure Nginx to serve them on different ports
3. Access both through browser

---

## Topic 2: Nginx Architecture

### Concept Explanation

**Simple Terms:**
Imagine a restaurant with one chef (master process) and several waiters (worker processes). The chef assigns tasks, and waiters handle customers without blocking each other. When a waiter is waiting for food from the kitchen, they can serve another customer. This is how Nginx handles many requests simultaneously.

**Deep Technical Explanation:**
Nginx architecture is built around these key principles:

1. **Master-Worker Process Model:**
   - **Master**: Reads config, binds to ports, creates/destroys workers
   - **Workers**: Handle connections, do actual work
   - **Cache Loader/Manager**: Special processes for cache management

2. **Event-Driven Architecture:**
   - Uses OS-specific event notification mechanisms
   - Linux: epoll()
   - BSD/macOS: kqueue()
   - Solaris: event ports

3. **Asynchronous I/O:**
   - Never blocks on I/O operations
   - Uses non-blocking sockets
   - Requests don't consume resources while waiting

**Process Flow Diagram:**
```
Start nginx
    ↓
Master Process
    ↓
Read Configuration
    ↓
Bind to Ports (80, 443)
    ↓
Fork Workers
    ↓
┌─────────────────────────────────────┐
│ Worker 1    Worker 2    Worker N    │
│ Event Loop  Event Loop  Event Loop  │
│ epoll()     epoll()     epoll()     │
└─────────────────────────────────────┘
    ↓           ↓           ↓
Accept connections on shared socket
    ↓
Add to event loop
    ↓
Process non-blocking I/O
```

### How It Works Internally

**Connection Handling:**
```
1. Worker calls accept() on shared listening socket
2. Creates connection object
3. Adds socket to event monitoring (epoll_ctl)
4. Returns to event loop (epoll_wait)
5. When data arrives, epoll_wait returns with event
6. Process request without blocking
7. Repeat
```

**Memory Pool (Pool Allocator):**
```c
// Simplified internal structure
struct ngx_pool_s {
    void           *last;      // Last allocated position
    void           *end;       // End of current pool block
    ngx_pool_t     *next;      // Next pool block
    void           *failed;    // Failed allocations counter
};
```

### Real-World Use Case

**Production Scenario:**
Cloudflare handles 20+ million requests per second using Nginx. Their architecture:
- Each server runs multiple Nginx workers (1 per CPU core)
- Workers handle 100,000+ concurrent connections
- Memory usage stays under 2GB per instance
- Zero downtime for configuration reloads

### Configuration Examples

**Tuning Worker Processes:**
```nginx
user  nginx;
worker_processes  auto;  # Automatically set to CPU cores
worker_rlimit_nofile 65535;  # File descriptor limit

events {
    worker_connections  4096;  # Connections per worker
    use epoll;  # Explicitly set event model (Linux)
    multi_accept on;  # Accept all new connections at once
    accept_mutex on;  # Prevent thundering herd
}

http {
    # Optimize send operations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Keepalive settings
    keepalive_timeout 65;
    keepalive_requests 100;
}
```

**Line-by-Line Explanation:**
- `worker_processes auto`: Detects CPU cores and creates equal workers
- `worker_rlimit_nofile`: Raises OS file descriptor limit for Nginx
- `worker_connections`: Maximum connections each worker can handle
- `use epoll`: Uses Linux's efficient event notification
- `multi_accept`: Worker accepts all pending connections at once
- `accept_mutex`: Prevents multiple workers waking up for new connection
- `sendfile`: Zero-copy file transfer (kernel space optimization)
- `tcp_nopush`: Optimizes packet sending for better network utilization

### Common Mistakes

1. **Setting worker_processes too high:**
   ```nginx
   # Wrong - causes context switching overhead
   worker_processes 100;
   
   # Correct - match CPU cores
   worker_processes auto;
   ```

2. **Ignoring file descriptor limits:**
   ```bash
   # Check system limits
   ulimit -n
   
   # Must be higher than worker_connections * worker_processes
   ```

3. **Not monitoring worker connections:**
   ```bash
   # Monitor active connections
   curl http://localhost/nginx_status
   ```

### Best Practices

1. **Set worker_processes to auto** or CPU core count
2. **Calculate max connections:**
   ```
   Max Clients = worker_processes * worker_connections
   ```
3. **Monitor with stub_status module:**
   ```nginx
   location /nginx_status {
       stub_status on;
       allow 127.0.0.1;
       deny all;
   }
   ```
4. **Use CPU affinity for NUMA systems:**
   ```nginx
   worker_cpu_affinity auto;
   ```

### Debugging Tips

```bash
# Check Nginx build configuration
nginx -V

# View open file descriptors
lsof -p $(pgrep -f 'nginx: master')

# Monitor event loop
strace -p <worker_pid> -e epoll_wait,accept,read,write

# Check connection statistics
netstat -an | grep :80 | wc -l

# Use nginx-debug binary
nginx-debug -t
```

### Interview Questions

**Beginner:**
Q: What is the master-worker architecture in Nginx?
A: Single master process reads config and manages workers. Multiple worker processes handle connections. This provides stability and zero-downtime reloads.

**Intermediate:**
Q: How does Nginx handle thousands of connections without threads?
A: Through asynchronous, non-blocking I/O. Each worker uses an event loop (epoll/kqueue) to monitor all connections. When a connection becomes readable/writable, it's processed immediately; otherwise, the worker handles other connections.

**Advanced:**
Q: Explain the "thundering herd" problem and how Nginx solves it.
A: Thundering herd occurs when multiple workers wake up simultaneously for new connections. Nginx uses accept_mutex - workers take turns accepting new connections. The mutex is released after accepting a configurable number (accept_mutex_delay).

### Hands-on Tasks

**Task 1: Architecture Exploration**
1. Install Nginx and check processes: `ps aux | grep nginx`
2. Count workers vs master
3. Monitor with `htop` to see CPU usage per worker
4. Generate load with `ab` (Apache Bench) and observe

**Task 2: Tuning Exercise**
1. Configure different worker_connections values
2. Benchmark with `wrk` or `ab`
3. Find optimal settings for your system
4. Document the impact

---

## Topic 3: Installing Nginx

### Concept Explanation

**Simple Terms:**
Installing Nginx is like installing any other software, but there are multiple ways: using package managers (apt, yum), compiling from source, or running in Docker. Each method has trade-offs between convenience and customization.

**Deep Technical Explanation:**
Nginx installation involves:
1. Binary installation (pre-compiled packages)
2. Source compilation (custom modules, optimizations)
3. Container deployment (isolation, reproducibility)
4. Configuration management (file locations, permissions)

**Installation Methods Comparison:**
| Method | Pros | Cons | Use Case |
|--------|------|------|----------|
| Package Manager | Easy, updates automatic | May have older version | Production servers |
| Source Compile | Custom modules, latest version | Complex, manual updates | Special requirements |
| Docker | Isolated, reproducible | Overhead, orchestration needed | Microservices, dev |
| Cloud Marketplace | Integrated, managed | Vendor lock-in | Cloud-native apps |

### Real-World Use Case

**Production Scenario:**
A fintech company needs:
- Custom security modules (ModSecurity)
- Specific OpenSSL version for compliance
- Performance optimizations for their hardware
- Same environment across dev/staging/prod

Solution: Docker with custom build ensures consistency everywhere.

### Installation Examples

**Method 1: Ubuntu/Debian (Package Manager)**
```bash
# Update package index
sudo apt update

# Install Nginx
sudo apt install nginx -y

# Check version
nginx -v

# Start and enable at boot
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

**Method 2: CentOS/RHEL**
```bash
# Add EPEL repository
sudo yum install epel-release -y

# Install Nginx
sudo yum install nginx -y

# Start and enable
sudo systemctl start nginx
sudo systemctl enable nginx

# Open firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

**Method 3: Compile from Source**
```bash
# Install dependencies
sudo apt install build-essential libpcre3-dev libssl-dev zlib1g-dev -y

# Download source
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -xzf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# Configure with modules
./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --modules-path=/usr/lib/nginx/modules \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_stub_status_module

# Compile and install
make
sudo make install
```

**Method 4: Docker Installation**
```dockerfile
# Dockerfile
FROM nginx:alpine

# Copy custom config
COPY nginx.conf /etc/nginx/nginx.conf
COPY website/ /usr/share/nginx/html/

# Expose ports
EXPOSE 80 443
```

```bash
# Build and run
docker build -t my-nginx .
docker run -d -p 80:80 --name web-server my-nginx

# Docker Compose
docker-compose.yml:
version: '3'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html
```

### Installation Directory Structure

```
/etc/nginx/
├── nginx.conf          # Main configuration
├── conf.d/             # Additional configs
├── sites-available/    # Available sites (Debian style)
├── sites-enabled/      # Symlinks to enabled sites
├── modules-available/  # Available modules
├── modules-enabled/    # Enabled modules
├── mime.types          # MIME type mappings
├── fastcgi_params      # FastCGI parameters
├── proxy_params        # Proxy parameters
├── scgi_params         # SCGI parameters
└── uwsgi_params        # uWSGI parameters

/var/log/nginx/
├── access.log          # Access logs
└── error.log           # Error logs

/var/www/html/          # Default web root
```

### Common Mistakes

1. **Not verifying GPG keys:**
   ```bash
   # Always verify packages
   curl -fsSL https://nginx.org/keys/nginx_signing.key | apt-key add -
   ```

2. **Wrong file permissions:**
   ```bash
   # Nginx needs read access to web root
   sudo chown -R www-data:www-data /var/www/
   ```

3. **Firewall blocking:**
   ```bash
   # Check if port is blocked
   sudo ufw status
   sudo ufw allow 'Nginx Full'
   ```

4. **SELinux issues (RHEL/CentOS):**
   ```bash
   # Allow Nginx to make network connections
   sudo setsebool -P httpd_can_network_connect on
   ```

### Best Practices

1. **Use official repositories** from nginx.org for latest versions
2. **Pin versions** in production to avoid unexpected updates
3. **Automate with configuration management** (Ansible, Puppet)
4. **Document custom builds** with build flags and reasons
5. **Test installation** with security benchmarks (CIS benchmarks)

### Debugging Tips

```bash
# Check if process is running
ps aux | grep nginx

# Test configuration after installation
sudo nginx -t

# Check ports
sudo netstat -tlnp | grep nginx

# View error logs
sudo tail -f /var/log/nginx/error.log

# Check systemd service status
sudo systemctl status nginx

# Verify SELinux context
ls -Z /usr/sbin/nginx
```

### Interview Questions

**Beginner:**
Q: What are the different ways to install Nginx?
A: Package managers (apt, yum), source compilation, Docker containers, cloud marketplaces. Each suited for different use cases.

**Intermediate:**
Q: How do you compile Nginx with custom modules?
A: Download source, run ./configure with --with-* flags for modules, make, make install. Need dependencies like PCRE for regex, OpenSSL for SSL.

**Advanced:**
Q: Explain the security implications of different installation methods and how you'd secure each.
A: Package managers: Automatic updates but may have older versions. Source: Full control but manual updates. Docker: Isolation but larger attack surface. Securing involves: minimal modules, proper file permissions, running as non-root, SELinux/apparmor, regular updates.

### Hands-on Tasks

**Task 1: Package Installation**
1. Install Nginx using your OS package manager
2. Find all installed files: `dpkg -L nginx` or `rpm -ql nginx`
3. Create a simple website
4. Access from browser

**Task 2: Docker Deployment**
1. Create Dockerfile with custom nginx.conf
2. Build image
3. Run container with volume mounts
4. Test modifications without rebuild

**Task 3: Source Compile Challenge**
1. Download Nginx source
2. Add a custom module (like echo-nginx-module)
3. Compile with specific flags
4. Compare binary size vs package manager version

---
