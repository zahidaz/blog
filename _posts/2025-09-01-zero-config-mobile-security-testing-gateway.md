---
layout: post
title: "One-Click Mobile Traffic Interception"
date: 2025-09-01
categories: [Mobile Security, Docker, Network Gateway, Traffic Interception]
author: "Zahid"
---

Traditional mobile security testing requires configuring proxy settings on each device, manually switching between intercepted and normal modes, and reconfiguring settings when passing devices between team members. This creates overhead and breaks testing flow.

This article presents a network-level solution that eliminates device configuration entirely. A containerized router acts as an intelligent gateway, allowing teams to dynamically redirect traffic from any device to any analyst's machine through simple API calls. Devices connect normally via DHCP while the router uses targeted iptables rules for selective interception.

The interception operates transparently at the network layer, bypassing anti-proxy detection while preserving access to debugging tools through excluded SSH, ADB, and Frida ports. Since it's Docker-based, the setup is easily redeployable across environments with consistent configurations.

## High-Level Overview

The system consists of three components on the same network:

- **Docker container with macvlan networking** - appears as a physical device with its own MAC and IP
- **REST service** - manages iptables rules for dynamic traffic redirection via API calls  
- **Mobile devices** - configured to use container IP as default gateway for transparent interception

## Prerequisites

- Docker installed with root privileges
- Linux host machine running Docker (Ubuntu/Debian recommended for network interface management)
- Network interface with promiscuous mode support
- Administrative access for network modifications
- Network subnet information (gateway IP, subnet range, available IPs)

## Network Setup

### Single Interface Configuration

```bash
# Enable promiscuous mode on your network interface
sudo ip link set dev eth0 promisc on

# Restart networking service (may cause brief disconnection)
sudo systemctl restart NetworkManager

# Create Docker macvlan network
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 \
    interceptor-network
```

Replace `eth0` with your interface name and adjust subnet/gateway to match your network.

## Container Deployment

### Build the Router Image

```bash
# Clone or create project directory with source files
# Build the container image
docker build -t traffic-interceptor .
```

### Deploy the Container

```bash
# Run the interceptor container
docker run -d --privileged \
    --name traffic-interceptor \
    --network=interceptor-network \
    --ip=192.168.1.200 \
    --restart unless-stopped \
    -e ROUTER_DNS=8.8.8.8,1.1.1.1 \
    -e ROUTER_SUBNET=192.168.1.0/24 \
    traffic-interceptor
```

Key parameters:
- `--privileged`: Required for iptables manipulation
- `--ip`: Static IP that devices will use as gateway  
- `--restart unless-stopped`: Survives host reboots
- `ROUTER_DNS`: DNS servers for name resolution
- `ROUTER_SUBNET`: Network range for traffic filtering

## Device Configuration

Configure mobile devices to use the container as default gateway:

### Manual Configuration
- **Gateway**: 192.168.1.200 (container IP)
- **DNS**: 8.8.8.8, 1.1.1.1
- **Subnet**: Match your network (e.g., 192.168.1.0/24)

### DHCP Server Configuration (Recommended)
Configure your DHCP server to assign the container IP as default gateway, eliminating manual device configuration. For consistency and reliable API targeting, ensure each device receives a static IP assignment through DHCP reservations based on MAC address - this prevents IP changes that would break existing redirection rules.

## How It Works

The container runs a REST service that manages iptables rules through system calls. When you make API requests, the service:

1. **Receives REST API calls** on port 8000 for traffic redirection requests
2. **Executes iptables commands** using sudo (configured for passwordless iptables access)
3. **Applies DNAT rules** in the PREROUTING chain to redirect traffic from source IPs
4. **Applies SNAT rules** in the POSTROUTING chain to ensure return traffic routes correctly

### Traffic Redirection Mechanism

When adding a redirect rule, the system creates two iptables rules:

```bash
# Redirect incoming traffic from source IP to destination
iptables -t nat -A PREROUTING -p tcp -s <srcIp> -j DNAT --to-destination <destIp>:<destPort>

# Ensure response traffic returns via the container
iptables -t nat -A POSTROUTING -p tcp -d <destIp> --dport <destPort> -j SNAT --to-source <containerIP>
```

The container preserves access to debugging ports (SSH:22, ADB:5555, Frida:27042) by excluding them from redirection rules, allowing researchers to maintain direct access to devices.

## API Usage

The container exposes a REST API on port 8000 for traffic management:

### Redirect Traffic
```bash
curl -X POST http://192.168.1.200:8000/api/router/addRedirect \
  -H "Content-Type: application/json" \
  -d '{
    "srcIp": "192.168.1.150",
    "destIp": "192.168.1.100",
    "destPort": 8080
  }'
```

Redirects all traffic from device `192.168.1.150` to analysis tools on `192.168.1.100:8080`.

### Remove Redirection
```bash
curl -X DELETE http://192.168.1.200:8000/api/router/deleteRedirect \
  -H "Content-Type: application/json" \
  -d '{
    "srcIp": "192.168.1.150",
    "destIp": "192.168.1.100",
    "destPort": 8080
  }'
```

### List Active Redirections
```bash
curl http://192.168.1.200:8000/api/router/getRedirects
```

## Workflow Integration

The system enables seamless testing workflows:

1. **Device Assignment**: Devices connect normally and auto-route through container
2. **Traffic Redirection**: API calls redirect specific devices to analysis workstations
3. **Dynamic Switching**: Instantly toggle between intercepted and normal internet access
4. **Team Collaboration**: Reassign devices between members without reconfiguration
5. **Parallel Testing**: Run multiple simultaneous interception sessions

## Troubleshooting

### Container Connectivity Issues
```bash
# Test container network connectivity
docker exec -it traffic-interceptor ping 8.8.8.8

# Check container logs
docker logs traffic-interceptor

# Verify interface supports promiscuous mode
```

### Device Communication Problems
```bash
# Check iptables rules
docker exec -it traffic-interceptor iptables -t nat -L

# Verify no IP conflicts with existing devices
```

### API Access Issues
```bash
# Confirm API service is listening
docker exec -it traffic-interceptor netstat -tlnp | grep 8000

# Check for firewall blocks to container
```

This network-level approach transforms mobile security testing by eliminating configuration overhead and enabling dynamic, transparent traffic redirection. Teams can focus on analysis rather than proxy configurations, while the containerized solution ensures consistent, reproducible testing environments.