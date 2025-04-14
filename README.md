# OpenVPN Main UDP Server (Dockerized)

This project provides a fully containerized **OpenVPN UDP server**, based on Alpine Linux with Easy-RSA for certificate management. It is designed to run as a standalone, reusable service, connected to a shared Docker network (e.g. with a backend monitoring system).

## 🧰 Features

- ✅ Alpine-based lightweight image
- ✅ Easy-RSA 3.x certificate handling
- ✅ `crl.pem` auto-generated on first run
- ✅ IP forwarding and NAT (iptables)
- ✅ Dynamic `server.conf` generation
- ✅ DNS and subnet configurable via environment variables
- ✅ Management port support
- ✅ Uses Docker volume for persistent state

---

## 📦 Docker Compose Usage

```bash
docker-compose up -d
```

### `docker-compose.yml` example:

```yaml
services:
  openvpn_main_udp:
    build:
      context: ./openvpn
      dockerfile: Dockerfile
    container_name: openvpn_main_udp
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - "/dev/net/tun:/dev/net/tun"
    volumes:
      - openvpn_main_data_udp:/openvpn-main-udp
    ports:
      - "1195:1195/udp"
    environment:
      DATA_DIR: /openvpn-main-udp
      PORT: "1195"
      PROTO: udp
      MGMT_PORT: "5095"
      DNS1: 8.8.8.8
      DNS2: 8.8.4.4
      VPN_SUBNET: 10.51.28.0
      VPN_NETMASK: 255.255.255.0
    networks:
      - backend_network

volumes:
  openvpn_main_data_udp:

networks:
  backend_network:
    external: true
    name: openvpngatemonitor_backend_network
```

---

## ⚙️ Environment Variables

| Variable        | Default         | Description                          |
|----------------|-----------------|--------------------------------------|
| `PORT`         | `1194`          | OpenVPN UDP port                     |
| `PROTO`        | `udp`           | Protocol (`udp` or `tcp`)            |
| `MGMT_PORT`    | `5092`          | Management interface port            |
| `DATA_DIR`     | `/mnt`          | Data/config directory in container   |
| `DNS1`         | `8.8.8.8`       | Primary DNS pushed to clients        |
| `DNS2`         | `8.8.4.4`       | Secondary DNS pushed to clients      |
| `VPN_SUBNET`   | `10.51.28.0`    | Subnet for VPN clients               |
| `VPN_NETMASK`  | `255.255.255.0` | Netmask for VPN clients              |

---

## 📂 Volumes

- `openvpn_main_data_udp`: contains Easy-RSA PKI, logs, config, etc.

---

## 📡 Integration

This container is intended to be connected to a shared Docker network such as `openvpngatemonitor_backend_network`, so that other services (e.g., dashboards, bots) can interact via management port or file volumes.

---

## 🏁 First Run

Certificates and keys are auto-generated if not found. The default `server.conf` is generated from environment variables if not present.

---

## 🛠 Maintainer

Developed by [you ❤️]. If you use this setup in your own infrastructure, feel free to fork or improve.