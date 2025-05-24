# Traefik Reverse Proxy Setup with Cloudflare Integration

This document outlines a three-phase approach to setting up Traefik as a reverse proxy for Nginx servers, progressively enhancing security and accessibility.

## Prerequisites

- Docker and Docker Compose installed
- A Cloudflare account with a registered domain (wardellsgarage.com)
- Basic understanding of networking and DNS

## Project Structure

```
.
├── docker-compose.yml
├── config/
│   ├── traefik.yml
│   └── dynamic/
│       └── tls.yml
├── nginx1/
│   └── html/
│       └── index.html
├── nginx2/
│   └── html/
│       └── index.html
├── certs/
│   ├── localhost.crt
│   └── localhost.key
├── cloudflared/
│   ├── config.yml
│   └── <tunnel-id>.json
└── .env
```

## Phase 1: Traefik with Self-Signed Certificates

This phase establishes HTTPS locally with self-signed certificates.

### Step 1: Create External Network

```bash
docker network create home-net
```

### Step 2: Generate Self-Signed Certificate

```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/localhost.key -out certs/localhost.crt \
  -subj "/CN=*.localhost" \
  -addext "subjectAltName = DNS:*.localhost,DNS:localhost"
```

### Step 3: Create Directory Structure

```bash
mkdir -p config/dynamic nginx1/html nginx2/html certs cloudflared
```

### Step 4: Create Traefik Configuration

**config/traefik.yml**:
```yaml
api:
  dashboard: true
  insecure: true  # For testing only

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http:
      tls:
        domains:
          - main: "localhost"
            sans: ["*.localhost"]

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: home-net
  file:
    directory: "/etc/traefik/dynamic"
    watch: true

log:
  level: "DEBUG"
```

### Step 5: Create Dynamic TLS Configuration

**config/dynamic/tls.yml**:
```yaml
tls:
  certificates:
    - certFile: /etc/certs/localhost.crt
      keyFile: /etc/certs/localhost.key
```

### Step 6: Create Docker Compose File

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Traefik dashboard
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./config/dynamic:/etc/traefik/dynamic:ro
      - ./certs:/etc/certs:ro
    networks:
      - home-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"

  nginx1:
    image: nginx:latest
    container_name: nginx1
    restart: unless-stopped
    volumes:
      - ./nginx1/html:/usr/share/nginx/html
    networks:
      - home-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx1.rule=Host(`nginx1.localhost`)"
      - "traefik.http.services.nginx1.loadbalancer.server.port=80"
      - "traefik.http.routers.nginx1.entrypoints=websecure"
      - "traefik.http.routers.nginx1.tls=true"

  nginx2:
    image: nginx:latest
    container_name: nginx2
    restart: unless-stopped
    volumes:
      - ./nginx2/html:/usr/share/nginx/html
    networks:
      - home-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx2.rule=Host(`nginx2.localhost`)"
      - "traefik.http.services.nginx2.loadbalancer.server.port=80"
      - "traefik.http.routers.nginx2.entrypoints=websecure"
      - "traefik.http.routers.nginx2.tls=true"

networks:
  home-net:
    external: true
```

### Step 7: Create HTML Content for Nginx Servers

**nginx1/html/index.html**:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Nginx Server 1</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background-color: #f0f8ff;
        }
        h1 {
            color: #0066cc;
        }
    </style>
</head>
<body>
    <h1>Welcome to Nginx Server 1</h1>
    <p>This is being served through the Traefik reverse proxy</p>
</body>
</html>
```

**nginx2/html/index.html**:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Nginx Server 2</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background-color: #f5f5dc;
        }
        h1 {
            color: #cc6600;
        }
    </style>
</head>
<body>
    <h1>Welcome to Nginx Server 2</h1>
    <p>This is being served through the Traefik reverse proxy</p>
</body>
</html>
```

### Step 8: Launch Services

```bash
docker-compose up -d
```

### Step 9: Testing

Add to your hosts file (`/etc/hosts` on Mac/Linux):
```
127.0.0.1 traefik.localhost nginx1.localhost nginx2.localhost
```

Access services:
- https://traefik.localhost:8080 - Traefik dashboard
- https://nginx1.localhost - First Nginx server
- https://nginx2.localhost - Second Nginx server

You'll need to accept browser security warnings about the self-signed certificate.

## Phase 2: Let's Encrypt with Cloudflare DNS Challenge

This phase implements proper SSL certificates using Let's Encrypt and Cloudflare.

### Step 1: Get Cloudflare API Token

1. Log into your Cloudflare account at https://dash.cloudflare.com/
2. Navigate to "My Profile" > "API Tokens"
3. Create a token with permissions for "Zone - DNS - Edit" for your domain
4. Copy the token value for use in the next steps

### Step 2: Create Environment File

**.env**:
```
CLOUDFLARE_EMAIL=your-email@example.com
CLOUDFLARE_API_KEY=your-api-key-or-token
```

### Step 3: Update Traefik Configuration

**config/traefik.yml**:
```yaml
api:
  dashboard: true
  insecure: true  # Consider securing for production

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: home-net
  file:
    directory: "/etc/traefik/dynamic"
    watch: true

certificatesResolvers:
  cloudflare:
    acme:
      email: your-email@example.com
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"

log:
  level: "DEBUG"
```

### Step 4: Update Docker Compose File

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  traefik:
    # ...existing configuration...
    env_file:
      - .env
    environment:
      - "CLOUDFLARE_EMAIL=${CLOUDFLARE_EMAIL}"
      - "CLOUDFLARE_API_KEY=${CLOUDFLARE_API_KEY}"
    # ...rest of traefik configuration...
    
  nginx1:
    # ...existing configuration...
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx1.rule=Host(`nginx1.wardellsgarage.com`)"
      - "traefik.http.services.nginx1.loadbalancer.server.port=80"
      - "traefik.http.routers.nginx1.entrypoints=websecure"
      - "traefik.http.routers.nginx1.tls=true"
      - "traefik.http.routers.nginx1.tls.certresolver=cloudflare"

  nginx2:
    # ...existing configuration...
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx2.rule=Host(`nginx2.wardellsgarage.com`)"
      - "traefik.http.services.nginx2.loadbalancer.server.port=80"
      - "traefik.http.routers.nginx2.entrypoints=websecure"
      - "traefik.http.routers.nginx2.tls=true"
      - "traefik.http.routers.nginx2.tls.certresolver=cloudflare"

networks:
  home-net:
    external: true
```

### Step 5: Restart Services

```bash
docker-compose down
docker-compose up -d
```

### Step 6: Configure DNS in Cloudflare

1. Log into Cloudflare dashboard
2. Add A records for your subdomains (nginx1, nginx2) pointing to your server's public IP
3. Ensure Cloudflare's proxy (orange cloud) is enabled for these records

## Phase 3: Cloudflare Tunnel Integration

This phase sets up secure external access via Cloudflare Tunnels.

### Step 1: Initialize Cloudflare Tunnel

```bash
# Log in to Cloudflare
docker run -it --rm -v ${PWD}/cloudflared:/etc/cloudflared cloudflare/cloudflared tunnel login

# Create tunnel
docker run -it --rm -v ${PWD}/cloudflared:/etc/cloudflared cloudflare/cloudflared tunnel create homelab
```

This will generate credentials in the `cloudflared` directory.

### Step 2: Configure the Tunnel

**cloudflared/config.yml**:
```yaml
tunnel: <your-tunnel-id>
credentials-file: /etc/cloudflared/<your-tunnel-id>.json
ingress:
  - hostname: nginx1.wardellsgarage.com
    service: https://nginx1
  - hostname: nginx2.wardellsgarage.com
    service: https://nginx2
  - hostname: traefik.wardellsgarage.com
    service: http://traefik:8080
  - service: http_status:404
```

Replace `<your-tunnel-id>` with the ID generated when creating the tunnel.

### Step 3: Update Docker Compose File

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  # ...existing services...
  
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    volumes:
      - ./cloudflared:/etc/cloudflared
    networks:
      - home-net
      
  # ...other services remain the same...
```

### Step 4: Configure DNS in Cloudflare Dashboard

1. Log into your Cloudflare dashboard at https://dash.cloudflare.com/
2. Select your domain (wardellsgarage.com)
3. Go to the "DNS" tab
4. For each service, add a CNAME record:
   - Click "Add record"
   - Type: CNAME
   - Name: nginx1 (for nginx1.wardellsgarage.com)
   - Content: <your-tunnel-id>.cfargotunnel.com
   - TTL: Auto
   - Proxy status: Proxied (orange cloud should be enabled)
5. Repeat this process for each subdomain (nginx2, traefik)
6. Navigate to "Network" section in your Cloudflare dashboard
7. Verify that "Cloudflare Tunnel" is enabled for your domain
8. Check "Zero Trust" > "Access" > "Tunnels" to confirm your tunnel is listed and active

### Step 5: Start Everything

```bash
docker-compose down
docker-compose up -d
```

### Step 6: Testing

Access your services through your domain:
- https://nginx1.wardellsgarage.com
- https://nginx2.wardellsgarage.com
- https://traefik.wardellsgarage.com

## Security Considerations

1. In production, secure the Traefik dashboard with authentication
2. Consider using non-root users in Docker containers
3. Regularly update all images and dependencies
4. Use specific image tags rather than 'latest'
5. Implement HTTP security headers
6. Consider using Cloudflare Access policies for additional protection

## Troubleshooting

### Certificate Issues
- Check Traefik logs: `docker logs traefik`
- Verify Cloudflare API token permissions
- Ensure DNS records are properly configured

### Connectivity Issues
- Verify tunnel status: `docker logs cloudflared`
- Check firewall settings
- Verify Docker network configuration
- Test local access before testing external access

### Service Discovery Problems
- Check Traefik dashboard for router and service status
- Verify label configurations in docker-compose.yml
- Check container health with `docker ps`