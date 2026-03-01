# NCOS API Documentation

Comprehensive NCOS (NetCloud OS) API documentation for Cradlepoint SDK application development.

## Documentation Structure

### API Trees
- [status/](status/README.md) - Status API (read-only, 87 endpoints) - WAN, GPS, modem, system health
- [config/](config/README.md) - Config API (persistent settings, 500+ paths) - WAN rules, firewall, features
- [control/](control/README.md) - Control API (actions) - Reboot, ping, GPIO, speedtest
- [state/](state/README.md) - State API (internal use)

### Reference Files
- [config/PATHS.md](config/PATHS.md) - Complete config path index
- [FEATURES_TO_ENABLE.md](FEATURES_TO_ENABLE.md) - Feature flags and UUIDs
- [config/dtd/NCOS-DTD-7.25.101.json](config/dtd/NCOS-DTD-7.25.101.json) - Full config schema

### Scripts
- **explore_status.py** - Query live router: `python3 explore_status.py status/wan/connection_state`
- **generate_config_paths.py** - Regenerate config index: `python3 generate_config_paths.py`

## API Access Methods

| Method | Use Case | Authentication |
|--------|----------|----------------|
| **SDK (on router)** | SDK apps | None (local only) |
| **REST (HTTP)** | Remote queries, scripts | Basic auth (`admin:password`) |
| **SSH CLI** | Interactive exploration | SSH login |

```python
# SDK
import cp
result = cp.get('status/wan/connection_state')

# REST
curl -u admin:password "http://ROUTER_IP/api/status/wan/connection_state"

# SSH CLI
ssh admin@ROUTER_IP
get status/wan/connection_state
```

## Common Tasks

### Monitor WAN Connection
```python
import cp

state = cp.get('status/wan/connection_state')
if state == 'connected':
    device = cp.get('status/wan/primary_device')
    ipinfo = cp.get('status/wan/ipinfo')
    cp.log(f'Connected via {device}: {ipinfo.get("ip_address")}')
```

### Get WAN Traffic Stats
```python
import cp

stats = cp.get('status/wan/stats')
if stats:
    in_bytes = stats.get('in', 0)
    out_bytes = stats.get('out', 0)
    cp.log(f'RX: {in_bytes} bytes, TX: {out_bytes} bytes')
```

### Reboot Router
```python
import cp
cp.put('control/system/reboot', 1)
```

### Read/Write GPIO
```python
import cp

# Read
gpio = cp.get('status/gpio')
led_state = gpio.get('LED_SS_0')

# Write
cp.put('control/gpio/LED_SS_0', 1)  # On
cp.put('control/gpio/LED_SS_0', 0)  # Off
```

### Check System Health
```python
import cp

sys = cp.get('status/system')
temp = sys.get('temperature', 0)
uptime = sys.get('uptime', 0)
load = sys.get('load_avg', {}).get('1min', 0)
cp.log(f'Temp: {temp}°C, Uptime: {uptime}s, Load: {load}')
```

### Monitor LAN Clients
```python
import cp

lan = cp.get('status/lan')
if lan:
    clients = lan.get('clients', [])
    for client in clients:
        mac = client.get('mac', 'unknown')
        ip = client.get('ip_address', 'unknown')
        cp.log(f'{mac} = {ip}')
```

### Get DNS Info
```python
import cp

dns = cp.get('status/dns')
if dns:
    cache = dns.get('cache', {})
    servers = cache.get('servers', [])
    for server in servers:
        addr = server.get('addr', 'unknown')
        queries = server.get('queries', 0)
        cp.log(f'DNS: {addr} ({queries} queries)')
```

### Configure WAN Rules
```python
import cp

rules = cp.get('config/wan/rules2')
if rules and isinstance(rules, list):
    for rule in rules:
        trigger_name = rule.get('trigger_name', 'Unknown')
        priority = rule.get('priority', 0)
        cp.log(f'{trigger_name}: priority={priority}')
```

#### WAN Rules Deep Dive

WAN rules define profiles matched to detected WAN devices using `trigger_string` criteria.

**Rule Fields:**
- `_id_` - Unique rule ID
- `trigger_string` - Match criteria (e.g., `type|is|ethernet`)
- `trigger_name` - Human-readable name
- `priority` - Sort order (lower = higher priority)

**Common trigger_string patterns:**
```
type|is|ethernet                              # Ethernet WAN
type|is|mdm                                   # Any cellular
type|is|mdm%sim|is|sim1%port|is|int1         # Internal modem SIM1
type|is|mdm%tech|is|lte                       # LTE-only
```

**Get rule for a device:**
```python
import cp

devices = cp.get('status/wan/devices') or {}
for dev_id, dev in devices.items():
    info = dev.get('info', {})
    config_id = info.get('config_id')
    if config_id:
        rule = cp.get(f'config/wan/rules2/{config_id}')
        if rule:
            cp.log(f'{dev_id} matches {rule.get("trigger_name")}')
```

### Monitor Firewall
```python
import cp

fw = cp.get('status/firewall')
if fw:
    state_timeouts = fw.get('state_timeouts', {})
    conn_count = state_timeouts.get('state_entry_count', 0)
    conn_limit = state_timeouts.get('state_entry_limit', 0)
    cp.log(f'Connections: {conn_count}/{conn_limit}')
```

### Check Features
```python
import cp

feat = cp.get('status/feature')
if feat:
    db = feat.get('db', [])
    for entry in db:
        if isinstance(entry, list) and len(entry) >= 2:
            uuid, name = entry[0], entry[1]
            cp.log(f'{name}: {uuid}')
```

### Get Product Info
```python
import cp

prod = cp.get('status/product_info')
if prod:
    model = prod.get('product_name', 'unknown')
    mac = prod.get('mac0', 'unknown')
    mfg = prod.get('manufacturing', {})
    serial = mfg.get('serial_num', 'unknown') if isinstance(mfg, dict) else 'unknown'
    cp.log(f'{model}, MAC: {mac}, SN: {serial}')
```

### Check App Status
```python
import cp

sys = cp.get('status/system')
if sys:
    sdk = sys.get('sdk', {})
    service = sdk.get('service', 'unknown')
    apps = sdk.get('apps', [])
    cp.log(f'SDK service: {service}, Apps: {len(apps)}')
    for app in apps:
        name = app.get('app', {}).get('name', 'unknown')
        state = app.get('state', 'unknown')
        cp.log(f'  {name}: {state}')
```

### Monitor Network Interfaces
```python
import cp

ethernet = cp.get('status/ethernet')
if ethernet and isinstance(ethernet, list):
    for port in ethernet:
        port_name = port.get('port_name', 'unknown')
        link = port.get('link', 'down')
        cp.log(f'{port_name}: {link}')
```

### Monitor VPN Tunnels
```python
import cp

vpn = cp.get('status/vpn')
if vpn:
    tunnels = vpn.get('tunnels', [])
    for tunnel in tunnels:
        name = tunnel.get('name', 'unknown')
        state = tunnel.get('state', 'unknown')
        cp.log(f'{name}: {state}')
```

### Monitor Cellular/Modem
```python
import cp

modem = cp.get('status/modem')
if modem:
    # Structure varies by model - see status/modem.md
    cp.log(f'Modem data present')
```
