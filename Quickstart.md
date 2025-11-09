# WireBalancer - Quick Start Guide

## What is WireBalancer?

WireBalancer is a high-performance load balancer that routes your traffic through multiple WireGuard VPN connections via SOCKS5 proxies. It's perfect for:

- Load balancing across multiple VPN connections
- Automatic failover when connections go down
- Real-time monitoring of connection health
- Choosing specific connections for different applications (this simplifies multi-threaded scraping tasks)

## Installation (5 minutes)

### Step 1: Download the project
```bash
cd /path/to/download
```

### Step 2: Add your WireGuard configs
```bash
mkdir wireguard-configs
# Copy your .conf files here
cp /path/to/your/wg0.conf wireguard-configs/
cp /path/to/your/wg1.conf wireguard-configs/
```

### Step 3: Configure WireBalancer
```bash
cp config.example.yml config.yml
nano config.yml  # Edit to match your setup
```

Example config:
```yaml
wireguard:
  connections:
    - name: "US Server"
      interface_name: "wg0"
      config_path: "/etc/wireguard/wg0.conf"
    
    - name: "EU Server"
      interface_name: "wg1"
      config_path: "/etc/wireguard/wg1.conf"
```

### Step 4: Start with Docker
```bash
# Option A: Using docker compose
docker compose up -d

# Option B: Using make
make docker-up
```

### Step 5: Access the dashboard
Open http://localhost:9929 in your browser

## Using the SOCKS5 Proxy

### SOCKS5 Port Assignments:
- **9930**: Random connection (load balanced)
- **9931**: First WireGuard connection (wg0)
- **9932**: Second WireGuard connection (wg1)
- **9933**: Third WireGuard connection (wg2)
- ... and so on

### Examples:

#### Test with curl:
```bash
# Random connection
curl -x socks5://localhost:9930 https://api.ipify.org

# Specific connection
curl -x socks5://localhost:9931 https://api.ipify.org
```

#### Configure your browser:
1. Open browser settings
2. Find proxy/network settings
3. Set SOCKS5 proxy:
   - Host: `localhost`
   - Port: `9930` (or specific port)

#### Python:
```python
import requests

proxies = {
    'http': 'socks5://localhost:9930',
    'https': 'socks5://localhost:9930'
}

response = requests.get('https://api.ipify.org', proxies=proxies)
print(f"Your IP: {response.text}")
```

#### Node.js:
```javascript
const SocksProxyAgent = require('socks-proxy-agent');
const fetch = require('node-fetch');

const agent = new SocksProxyAgent('socks5://localhost:9930');

fetch('https://api.ipify.org', { agent })
  .then(res => res.text())
  .then(ip => console.log('Your IP:', ip));
```

#### Go:
```go
package main

import (
    "fmt"
    "io/ioutil"
    "net/http"
    "golang.org/x/net/proxy"
)

func main() {
    dialer, err := proxy.SOCKS5("tcp", "localhost:9930", nil, proxy.Direct)
    if err != nil {
        panic(err)
    }

    transport := &http.Transport{Dial: dialer.Dial}
    client := &http.Client{Transport: transport}

    resp, err := client.Get("https://api.ipify.org")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }

    fmt.Println("Your IP:", string(body))
}
```

## Monitoring

### Dashboard (http://localhost:9929)
- View all connection statuses
- See request counts per connection
- Monitor latency
- Check uptime

### API Endpoint
```bash
# Get JSON stats
curl http://localhost:9929/api/stats | jq
```

## Common Commands

```bash
# View logs
docker compose logs -f

# Restart
docker compose restart

# Stop
docker compose down

# Rebuild after config changes
docker compose up -d --force-recreate

# Check status
docker compose ps

# Get stats
make stats
```

## Troubleshooting

### Connections showing as unhealthy?
```bash
# Check WireGuard status
docker exec wirebalancer wg show

# Check logs
docker compose logs wirebalancer

# Test connectivity
docker exec wirebalancer ping 8.8.8.8
```

### Proxy not working?
```bash
# Test SOCKS5 proxy
curl -v -x socks5://localhost:9930 https://google.com

# Check if ports are open
netstat -tlnp | grep 9930
```

### Need to update config?
```bash
# Edit config
nano config.yml

# Restart
docker compose restart
```

### WireGuard interfaces not coming up

1. Check WireGuard configuration files are valid
2. manual builds/outside docker: Ensure container has proper capabilities (`NET_ADMIN`, `SYS_MODULE`)
3. manual builds/outside docker: Verify kernel modules are loaded: `lsmod | grep wireguard`

### Connections marked as unhealthy

1. Check WireGuard connection: `docker exec wirebalancer wg show`
2. Verify routing: `docker exec wirebalancer ip route`
3. Test connectivity: `docker exec wirebalancer ping 8.8.8.8`
4. Check logs: `docker compose logs -f wirebalancer`


## Advanced Usage

### Running without Docker:
```bash
# Build
make build

# Run (requires root for WireGuard)
sudo ./wirebalancer -config config.yml
```

### System Service:
```bash
# Install as systemd service
sudo cp wirebalancer /usr/local/bin/
sudo cp wirebalancer.service /etc/systemd/system/
sudo systemctl enable wirebalancer
sudo systemctl start wirebalancer
```

### Multiple Instances:
You can run multiple instances on different ports by:
1. Copy the directory
2. Change port numbers in config.yml
3. Run with different compose project names:
```bash
docker compose -p wirebalancer2 up -d
```

## Performance Tips

1. **Buffer Size**: Increase `buffer_size` in config for high-bandwidth connections
2. **Health Checks**: Adjust `health_check_interval` based on your needs (lower = more accurate, higher = less overhead)
3. **Network Mode**: Use `host` network mode for best performance

