# WireBalancer

> Randomize the IP of your web requests.

Wirebalancer is an outbound load balancer designed to distribute outbound traffic across multiple WireGuard VPN connections. 
It launches multiple WireGuard connections from inside a single container, and exposes them as SOCKS5 proxies


See it in action:

1. Configure the `config.yaml`

1. launch the proxy on your machine:

```bash
docker compose up -d
```

1. make randomized requests! Python example:

```python
import requests

proxies = {
    'http': 'socks5://localhost:9930',
    'https': 'socks5://localhost:9930'
}

response = requests.get('https://api.ipify.org', proxies=proxies)
print(response.text) 
# > Your Ip will be different on each request
```

Check out the [quick start guide](#quick-start) for a complete guide.

## Why WireBalancer?

This project simplifies the process of distributing outbound traffic across multiple WireGuard VPN connections. 
This is especially useful for applications that require multiple IP addresses or need to bypass geo-restrictions, censorship or rate limits imposed by certain services (e.g., web scraping, API access).

## Features

- **WireGuard Integration**: Seamlessly manages multiple WireGuard VPN connections
- **SOCKS5 Proxies**: Exposes SOCKS5 proxies for easy integration with existing applications and libraries
- **Web Dashboard**: Monitor connection health and statistics in real-time via a simple web interface
- **Health Checks**: Automatic health monitoring and failover for WireGuard connections
- **Randomized Proxy Selection**: Distributes traffic evenly across all healthy connections
- **Dedicated Proxies**: Allows binding specific applications to specific WireGuard connections

## Architecture

```
┌────────────────────────────────────────────┐
│              WireBalancer                  │
│                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  SOCKS5  │  │  SOCKS5  │  │  SOCKS5  │  │
│  │  :9930   │  │  :9931   │  │  :9932   │  │
│  │ (Random) │  │  (WG0)   │  │  (WG1)   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │             │             │        │
│  ┌────▼─────────────▼─────────────▼─────┐  │
│  │           Connection Manager         │  │
│  └────┬─────────────┬─────────────┬─────┘  │
│       │             │             │        │
│  ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   │
│  │  wg0    │   │  wg1    │   │  wg2    │   │
│  └─────────┘   └─────────┘   └─────────┘   │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │    Web Dashboard :9929               │  │
│  │    (Stats & Health Monitoring)       │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

## Usecases

### Usecase 1: Scraping with well known VPN providers

Imagine you want to scrape data from a website that limits requests based on IP addresses.
With WireBalancer, you can buy a cheap VPN service that provides multiple connections (many well-known VPN providers offer 3 to 5 simultaneous connections even on their basic plans),
than set up WireBalancer to distribute your scraping requests across these connections, effectively making your scraper appear to come from different IPs, thus increasing your scraping speed and reducing the chance of being blocked.
You can also buy multiple cheap VPS instances and set up WireGuard on each of them to get more unique IP addresses for your scraping tasks.

#### Usecase 2: Network Resilience

Imagine you're running a critical application that requires constant internet connectivity, such as a remote monitoring system or a VoIP service.
By using WireBalancer with multiple WireGuard connections, you can ensure that if one VPN connection goes down, your application can seamlessly switch to another healthy connection without any downtime.

### Usecase 3: Only SOCKS5 Support

Some applications only support SOCKS5 proxies for outbound connections, or at least it's the easiest way to configure them.
With WireBalancer, you can leverage multiple WireGuard VPN connections and expose them as SOCKS5 proxies, allowing these applications to benefit from load balancing and failover without needing native WireGuard support.
For example the "requests" library in Python has built-in SOCKS5 support.


## Quick Start

### Prerequisites

- Docker and Docker Compose (recommended) **or** Go 1.25 installed
- WireGuard configuration files
- Root/sudo access (required for WireGuard)

### Installation

1. Clone the repository:
```bash
git clone https://github.com/tomventa/wirebalancer.git
cd wirebalancer
```

2. Create your WireGuard configurations:
```bash
# Copy your WireGuard .conf files to this directory
cp /path/to/wg0.conf wireguard-configs/
cp /path/to/wg1.conf wireguard-configs/
```

3. Create your configuration file:
```bash
cp config.example.yml config.yml
# Edit config.yml to match your setup
```

4. Start the service:
```bash
docker compose up -d
```

5. Access the dashboard:
```
http://localhost:9929
```

## Configuration

### Configuration File (`config.yml`)

```yaml
wireguard:
  # Health check settings
  health_check_url: "https://cloudflare.com/cdn-cgi/trace"
  health_check_interval: 30  # seconds
  failure_threshold: 3       # consecutive failures before marking unhealthy

  # WireGuard connections
  connections:
    - name: "US East"
      interface_name: "wg0"
      config_path: "/etc/wireguard/wg0.conf"
    
    - name: "EU West"
      interface_name: "wg1"
      config_path: "/etc/wireguard/wg1.conf"

proxy:
  base_port: 9930           # Starting port number
  read_timeout: 30          # Connection read timeout (seconds)
  write_timeout: 30         # Connection write timeout (seconds)
  failure_http_code: 580    # Error code for unhealthy connections
  buffer_size: 32768        # Data transfer buffer size (bytes)

webserver:
  port: 9929                # Web dashboard port
```

**Note**: If you change base_port, ensure your Docker Compose or Docker run command maps the correct ports.

## Port Mapping

| Port | Description |
|------|-------------|
| 9929 | Web dashboard (HTTP) |
| 9930 | SOCKS5 proxy - Random connection selection |
| 9931 | SOCKS5 proxy - WireGuard connection 0 |
| 9932 | SOCKS5 proxy - WireGuard connection 1 |
| 9933 | SOCKS5 proxy - WireGuard connection 2 |
| 9934 | SOCKS5 proxy - WireGuard connection 3 |
| ... | Additional connections as configured |

## Usage

### SOCKS5 Proxy

#### Using curl with random connection:
```bash
curl -x socks5://localhost:9930 https://api.ipify.org
```

#### Using curl with specific connection:
```bash
# Use first WireGuard connection
curl -x socks5://localhost:9931 https://api.ipify.org

# Use second WireGuard connection
curl -x socks5://localhost:9932 https://api.ipify.org
```

#### Python example:
```python
import requests

proxies = {
    'http': 'socks5://localhost:9930',
    'https': 'socks5://localhost:9930'
}

response = requests.get('https://api.ipify.org', proxies=proxies)
print(response.text)
```

#### Browser configuration:
Configure your browser to use SOCKS5 proxy:
- Host: `localhost`
- Port: `9930` (or specific port for dedicated connection)

## Building from Source

### Without Docker:

```bash
# Install dependencies
go mod download

# Build
go build -o wirebalancer .

# Run (requires root for WireGuard)
sudo ./wirebalancer -config config.yml
```

### With Docker and without Docker Compose:

```bash
docker build -t wirebalancer .
docker run --cap-add=NET_ADMIN --cap-add=SYS_MODULE \
  -v $(pwd)/config.yml:/app/config.yml \
  -v $(pwd)/wireguard-configs:/etc/wireguard \
  -p 9929:9929 -p 9930-9935:9930-9935 \
  wirebalancer
```

## Monitoring

### Web Dashboard

Access the real-time dashboard at `http://localhost:9929`:

- Total requests across all connections
- Individual connection statistics
- Health status for each WireGuard connection
- Average latency per connection
- Uptime tracking

### API Endpoint

Get JSON statistics programmatically:

```bash
curl http://localhost:9929/api/stats
```

Response:
```json
{
  "total_requests": 1234,
  "uptime": "2h15m30s",
  "uptime_seconds": 8130,
  "connections": [
    {
      "index": 0,
      "name": "US East",
      "healthy": true,
      "request_count": 456,
      "average_latency": "45ms",
      "latency_ms": 45.2,
      "last_check": "14:30:15"
    }
  ]
}
```

## Performance Optimization

The application is designed for high performance:

1. **Connection Pooling**: Reuses buffers and connections
2. **Concurrent Processing**: Each proxy port runs in its own goroutine
3. **Zero-copy Operations**: Efficient data transfer between connections
4. **Atomic Operations**: Lock-free statistics tracking
5. **Buffer Pool**: Shared buffer pool reduces GC pressure

## Quick start

See [Quickstart.md](Quickstart.md) for a step-by-step guide to get started quickly.


## License

MIT License - See LICENSE file for details

## A small note

WireBalancer is an open-source project developed in my spare time and in a hurry (hence the somewhat rough edges).
While I use it personally and it works well for my needs, please use it at your own risk and always review the code and configurations to ensure they meet your security and operational requirements.
This project is provided "as is" without any warranties.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
