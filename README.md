# Self-Hosting SSL-enabled n8n on Google Cloud Free Tier with Docker & Caddy

This guide provides step-by-step instructions to self-host [n8n](https://n8n.io), a free and open-source workflow automation tool, on a **Google Cloud Platform (GCP) free-tier VM**, using **Docker**, **Docker Compose**, and **Caddy** as a reverse proxy for HTTPS, with or without a **custom domain name**.

---

## Step 1: Create a Free Tier VM Instance on Google Cloud

1. Go to **Compute Engine > VM Instances** and click **Create Instance**.
2. Choose:
   - **Name**: `n8n-server`
   - **Region**: `us-central1` (Free tier eligible)
   - **Machine type**: `e2-micro`
   - **Boot Disk**: Ubuntu 22.04 LTS (30 GB)
3. Under **Firewall**, leave both boxes unchecked (we will configure it manually).
4. Click **Create**.

---

## Step 2: Reserve a Static IP

1. Go to **VPC Network > IP Addresses**.
2. Change the assigned ephemeral IP to a **Static IP** for your VM instance.

---

## Step 3: Configure Firewall Rules

1. Go to **VPC > Firewall rules** and create rules:

| Name         | Protocol | Port | Source       | Target Tags  |
|--------------|----------|------|--------------|--------------|
| allow-http   | TCP      | 80   | 0.0.0.0/0    | n8n-server   |
| allow-https  | TCP      | 443  | 0.0.0.0/0    | n8n-server   |
| allow-n8n    | TCP      | 5678 | 0.0.0.0/0    | n8n-server *(optional)* |

2. Go to your VM → Click **Edit** → Add **Network tag**: `n8n-server`

---

## Step 4: Connect via SSH and Install Docker & Docker Compose

```bash
# Update and install prerequisites
sudo apt update && sudo apt install \
    ca-certificates curl gnupg lsb-release -y

# Add Docker GPG key and repository
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker & Docker Compose
sudo apt update && sudo apt install \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin -y

# Add your user to the Docker group
sudo usermod -aG docker $USER
```

> ⚠️ **Important**: Log out and back in for Docker group changes to apply.

---

## Step 5: Choose Your Setup Type

You can run n8n in 3 ways:

---

### 🔸 A) Using a **Subdomain** (e.g., `n8n.example.com`)

1. Set an A record for `n8n.example.com` pointing to your VM's static IP.

2. Create `docker-compose.yml`:

```bash
mkdir n8n && cd n8n
nano docker-compose.yml
```

Paste this:

```yaml
version: '3.7'

services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    environment:
      - N8N_HOST=n8n.example.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.example.com/
    volumes:
      - n8n_data:/home/node/.n8n

  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - n8n

volumes:
  n8n_data:
  caddy_data:
```

Then create `Caddyfile`:

```bash
nano Caddyfile
```

Paste:

```caddy
n8n.example.com {
    reverse_proxy n8n:5678
}
```

Start:

```bash
docker compose up -d
```

Access at: `https://n8n.example.com`

---

### 🔸 B) Using a **Root Domain** (e.g., `example.com`)

Just change `n8n.example.com` to `example.com` in all files above:

- `N8N_HOST=example.com`
- `WEBHOOK_URL=https://example.com/`
- In `Caddyfile`: `example.com { ... }`

Update DNS A record for `example.com` → Static IP

Start with:

```bash
docker compose up -d
```

Access at: `https://example.com`

---

### 🔸 C) Without Any Domain (Access via IP and Port)

Skip Caddy and use Docker only:

```bash
docker run -d --restart unless-stopped --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Then open:  
```
http://your-server-ip:5678
```

⚠️ This is NOT secure. Use only for **local testing** or behind a VPN.  
To add HTTPS, use a domain with Caddy or Nginx.

---

## Optional: Remove 5678 Port from Firewall (After HTTPS Setup)

```bash
gcloud compute firewall-rules delete allow-n8n
```

---

## Backup & Update

Backup:

```bash
cp -r ~/.docker/volumes/n8n_n8n_data/_data ~/n8n-backup
```

Update:

```bash
docker compose pull
docker compose down
docker compose up -d
```

---

## Troubleshooting

- Caddy logs: `docker logs <caddy-container-id>`
- n8n logs: `docker logs <n8n-container-id>`
- Restart containers: `docker compose restart`

---

## Why Use Caddy?

- 📦 Auto HTTPS with Let's Encrypt
- 🧠 Simple reverse proxy config
- 🔁 Automatic renewals
- 🛡 Modern & fast

---

## Final Tips

- ✅ Use a domain for production (subdomain or root)
- 🔐 Never expose port 5678 directly in production
- 💾 Always backup workflows before upgrades

---

## Links

- n8n docs: https://docs.n8n.io/
- Caddy docs: https://caddyserver.com/docs/
