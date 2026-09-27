# WGDashboard — Installation

Install files for the WGDashboard panel and its `wg-node` agent. The panel's
source code is private; installation only uses Docker and the prebuilt
images (`ghcr.io/hosseinpv1379/*`).

## Install Docker

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### CentOS / RHEL

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

## Open the required firewall ports

```bash
# On the panel server
sudo ufw allow 10086/tcp   # web panel
sudo ufw allow 51820/udp   # WireGuard (panel's local node)

# On each node server
sudo ufw allow 51820/udp   # WireGuard
```

Using CentOS/`firewalld` instead:

```bash
sudo firewall-cmd --permanent --add-port=10086/tcp
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
```

Also open `80/tcp` and `443/tcp` on the panel server if you use the HTTPS profile.

## Install the panel

### Download the files

```bash
mkdir -p /opt/WGDashboard/docker && cd /opt/WGDashboard/docker
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/docker/compose.yaml -o compose.yaml
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/docker/.env.example -o .env
```

### Configure `.env`

```bash
nano .env
```

Set at least these values:

| Variable | Description |
| --- | --- |
| `WGD_ADMIN_PASSWORD` | panel admin password |
| `POSTGRES_PASSWORD` | database password |
| `PUBLIC_IP` | server's public IP |
| `WGD_DOMAIN` | (optional) domain for automatic HTTPS — also set `COMPOSE_PROFILES=https` when you set this |

### Run

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

The panel comes up at `http://PUBLIC_IP:10086` (or `https://WGD_DOMAIN` if you enabled HTTPS).

### Stop

```bash
docker compose --env-file .env down
```

### Update

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d --remove-orphans
```

### Logs

```bash
docker compose --env-file .env logs -f wgdashboard
```

## Install a node

Each node runs on its own server. First create the node from **Panel → Nodes** and copy its **Node ID** and **Node Token**.

### Download the files

```bash
mkdir -p /opt/wg-node && cd /opt/wg-node
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/wg-node/compose.example.yaml -o compose.yaml
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/wg-node/.env.example -o .env
```

### Configure `.env`

```bash
nano .env
```

Set at least these values:

| Variable | Description |
| --- | --- |
| `WG_PANEL_URL` | the panel's address, e.g. `https://panel.example.com` |
| `WG_NODE_ID` / `WG_NODE_TOKEN` | from the panel's Nodes page |
| `WG_NODE_PUBLIC_ENDPOINT` | this node server's IP, e.g. `1.2.3.4:51820` |
| `WG_NODE_NAME` / `WG_NODE_REGION` | node name and location |

### Run

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

### Stop

```bash
docker compose --env-file .env down
```

### Update

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

### Logs

```bash
docker compose --env-file .env logs -f
```
