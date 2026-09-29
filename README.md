# WGDashboard — Installation

Install files for the WGDashboard panel and its `wg-node` (WireGuard) and
`ov-node` (OpenVPN) node agents. The panel's source code is private;
installation only uses Docker and the prebuilt images
(`ghcr.io/hosseinpv1379/*`).

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

# On each WireGuard node server
sudo ufw allow 51820/udp   # WireGuard

# On each OpenVPN node server
sudo ufw allow 1194/udp    # OpenVPN (UDP instance)
sudo ufw allow 1195/tcp    # OpenVPN (TCP instance)
```

Using CentOS/`firewalld` instead:

```bash
sudo firewall-cmd --permanent --add-port=10086/tcp
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --permanent --add-port=1194/udp
sudo firewall-cmd --permanent --add-port=1195/tcp
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

### Enable IP forwarding

Once, before creating any interface, on the node server:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

The agent runs unprivileged inside a container, so it can't set this from
inside itself (Docker mounts `/proc/sys` read-only unless the container is
`--privileged`) — and since these compose files use `network_mode: host`,
the host's own setting is the only one that matters anyway. Skipping this
doesn't stop the node from coming online, but clients get no internet
access through it (you'll see a `Read-only file system` warning in the
agent's logs).

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

Once the node shows `online` in the panel (usually under a minute), go to
**Panel → Nodes → this node → Add interface** to create its first WireGuard
interface — the agent registers the node but never creates an interface on
its own. Leave the address pool/port blank to let the panel pick one, or
give an exact CIDR like `10.88.0.0/24`. Repeat for any extra interface;
delete one from the same page's trash icon (refused while a peer or
outbound still uses it).

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

## Install an OpenVPN node

`ov-node` is the OpenVPN sibling of `wg-node` — same registration/heartbeat
flow, different protocol. Each node runs on its own server. First create
the node from **Panel → OpenVPN** and copy its **Node ID** and **Node
Token**.

### Enable IP forwarding

Same reason and same command as the WireGuard node above — once, before
creating any interface:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Download the files

```bash
mkdir -p /opt/ov-node && cd /opt/ov-node
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/ov-node/compose.example.yaml -o compose.yaml
curl -fsSL https://raw.githubusercontent.com/hosseinpv1379/wgdashboard-install/main/ov-node/.env.example -o .env
```

### Configure `.env`

```bash
nano .env
```

Set at least these values:

| Variable | Description |
| --- | --- |
| `OV_PANEL_URL` | the panel's address, e.g. `https://panel.example.com` |
| `OV_NODE_ID` / `OV_NODE_TOKEN` | from the panel's OpenVPN page |
| `OV_NODE_PUBLIC_ENDPOINT` | this node server's IP, e.g. `1.2.3.4:1194` |
| `OV_NODE_NAME` / `OV_NODE_REGION` | node name and location |

No need to install OpenVPN, generate any certificate, or write a config by
hand — the agent runs its own certificate authority and provisions
everything itself.

### Run

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

Once the node shows `online` in the panel (usually under a minute), go to
**Panel → OpenVPN → this node → Add interface** to create its first
interface — the agent registers the node but never creates an interface on
its own. Every interface always runs a UDP and a TCP OpenVPN instance side
by side, so one certificate connects over either; leave the ports/pool
blank to use the defaults, or set your own. Repeat for any extra interface;
delete one from the same page's trash icon (refused while a user still
targets it).

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
