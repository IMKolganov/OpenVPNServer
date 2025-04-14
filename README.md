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
---

## 🔐 Generating a client `.ovpn` file

To create a client certificate and generate an `.ovpn` file, you can use the following script inside the container:

### 📜 Example script: `generate-client.sh`

```bash
#!/bin/bash
set -e

CLIENT_NAME=$1
EASYRSA_DIR=/openvpn-main-udp/easy-rsa
OUTPUT_DIR=/openvpn-main-udp/clients/$CLIENT_NAME

if [ -z "$CLIENT_NAME" ]; then
    echo "Usage: $0 <client-name>"
    exit 1
fi

cd "$EASYRSA_DIR"
export EASYRSA_PKI="$EASYRSA_DIR/pki"
export EASYRSA_BATCH=1

./easyrsa gen-req "$CLIENT_NAME" nopass
./easyrsa sign-req client "$CLIENT_NAME"

mkdir -p "$OUTPUT_DIR"

cp "$EASYRSA_PKI/issued/$CLIENT_NAME.crt" "$OUTPUT_DIR/"
cp "$EASYRSA_PKI/private/$CLIENT_NAME.key" "$OUTPUT_DIR/"
cp "$EASYRSA_PKI/ca.crt" "$OUTPUT_DIR/"
cp "$EASYRSA_PKI/ta.key" "$OUTPUT_DIR/"

cat > "$OUTPUT_DIR/$CLIENT_NAME.ovpn" <<EOF
client
dev tun
proto udp
remote your-server-address 1195
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
tls-crypt ta.key
cipher AES-256-CBC
auth SHA256
verb 3

<ca>
$(cat "$OUTPUT_DIR/ca.crt")
</ca>

<cert>
$(cat "$OUTPUT_DIR/$CLIENT_NAME.crt")
</cert>

<key>
$(cat "$OUTPUT_DIR/$CLIENT_NAME.key")
</key>

<tls-crypt>
$(cat "$OUTPUT_DIR/ta.key")
</tls-crypt>
EOF

echo "✅ Client config created at: $OUTPUT_DIR/$CLIENT_NAME.ovpn"
```

> Replace `your-server-address` with your actual public domain or IP.

### 🚀 Usage inside the container:

```bash
docker exec -it openvpn_main_udp bash
./generate-client.sh alice
```

The resulting `.ovpn` file will be located at:
```
/openvpn-main-udp/clients/alice/alice.ovpn
```