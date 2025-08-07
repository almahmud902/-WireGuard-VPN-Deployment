# Self-Hosted WireGuard VPN on Cloud VPS

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![WireGuard](https://img.shields.io/badge/Protocol-WireGuard-%238817F2)](https://www.wireguard.com/)
[![Shell Script](https://img.shields.io/badge/Shell-Bash-%23121011)](https://www.gnu.org/software/bash/)

A secure, high-performance VPN solution deployed on a cloud VPS using WireGuard. Eliminates third-party VPN dependencies while providing full control over network privacy and geo-routing.

## ✨ Features

- **Zero-Trust Architecture**: End-to-end encrypted tunnels
- **Automated Client Onboarding**: QR code generation for mobile devices
- **Geo-Flexibility**: Choose your egress location by deploying in different regions
- **Performance Optimized**: ~20% faster than OpenVPN/IPSec in benchmarks
- **Firewall Integration**: UFW/iptables rules for enhanced security

## 🛠️ Technical Stack

| Component       | Technology Used |
|----------------|----------------|
| VPN Protocol   | WireGuard (UDP) |
| Infrastructure | Cloud VPS (1GB+) |
| Automation     | Bash Scripting |
| Security       | UFW, fail2ban |
| Monitoring     | `wg show`, Prometheus (optional) |

## 🚀 Deployment

### Prerequisites
- Linux VPS (Ubuntu 20.04+ recommended)
- Root access
- Open ports: UDP 51820 (WireGuard)

### One-Line Install
```bash
wget https://raw.githubusercontent.com/your-repo/main/install.sh && chmod +x install.sh && sudo ./install.sh


Used scripts in this project:(this scripts are general scripts not with real data for security perpose)

#!/bin/bash
# WireGuard VPN Auto-Setup Script
# Creates server config + generates client configs with QR codes
# Tested on Ubuntu/Debian. Run as root.

set -e  # Exit on error

# Configuration
WG_PORT="51820"
WG_NET="10.8.0.1/24"
DNS_SERVERS="1.1.1.1,8.8.8.8"
SERVER_PUBLIC_IP="5.189.176.25"  # Replace with your VPS IP

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

# Dependencies check
check_deps() {
  for cmd in wg qrencode ip iptables; do
    if ! command -v $cmd &> /dev/null; then
      echo -e "${RED}Error: $cmd not found. Installing...${NC}"
      apt-get update && apt-get install -y wireguard qrencode iproute2 iptables
    fi
  done
}

# Firewall setup
setup_firewall() {
  ufw allow $WG_PORT/udp
  ufw allow OpenSSH
  echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
  sysctl -p
}

# Generate server config
init_server() {
  mkdir -p /etc/wireguard
  wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey

  cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
PrivateKey = $(cat /etc/wireguard/privatekey)
Address = $WG_NET
ListenPort = $WG_PORT
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
EOF

  chmod 600 /etc/wireguard/{privatekey,wg0.conf}
}

# Add new client
add_client() {
  CLIENT_NAME="$1"
  [ -z "$CLIENT_NAME" ] && { echo -e "${RED}Usage: $0 add <client_name>${NC}"; exit 1; }

  # Generate keys
  CLIENT_PRIVKEY=$(wg genkey)
  CLIENT_PUBKEY=$(echo "$CLIENT_PRIVKEY" | wg pubkey)
  NEXT_IP=$(expr $(grep -oP '10.8.0.\K\d+' /etc/wireguard/wg0.conf | sort -n | tail -1) + 1 2>/dev/null || echo 2)

  # Server config update
  wg set wg0 peer "$CLIENT_PUBKEY" allowed-ips "10.8.0.$NEXT_IP/32"
  wg-quick save wg0

  # Client config
  mkdir -p /etc/wireguard/clients
  cat > "/etc/wireguard/clients/$CLIENT_NAME.conf" <<EOF
[Interface]
PrivateKey = $CLIENT_PRIVKEY
Address = 10.8.0.$NEXT_IP/24
DNS = $DNS_SERVERS

[Peer]
PublicKey = $(cat /etc/wireguard/publickey)
Endpoint = $SERVER_PUBLIC_IP:$WG_PORT
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

  # Generate QR
  qrencode -t ansiutf8 < "/etc/wireguard/clients/$CLIENT_NAME.conf"
  echo -e "${GREEN}Config saved to: /etc/wireguard/clients/$CLIENT_NAME.conf${NC}"
}

# Main
case "$1" in
  install)
    check_deps
    setup_firewall
    init_server
    systemctl enable --now wg-quick@wg0
    echo -e "${GREEN}WireGuard installed! Use '$0 add <client>' to create clients${NC}"
    ;;
  add)
    add_client "$2"
    ;;
  *)
    echo -e "${RED}Usage:${NC}"
    echo "  $0 install    - Initial server setup"
    echo "  $0 add <name> - Add new client"
    exit 1
    ;;
esac
