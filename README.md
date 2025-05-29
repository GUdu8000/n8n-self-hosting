# 🚀 Secure n8n on GCP Free Tier: A Student's Quick Start Guide

Deploy [n8n](https://n8n.io) on Google Cloud Free Tier using Docker and Caddy with HTTPS & Auto-Start.

---

## 📋 Requirements Checklist

| ✅ Required             | Description                                        |
|------------------------|----------------------------------------------------|
| GCP Account            | Sign up at [console.cloud.google.com](https://console.cloud.google.com) |
| Billing Enabled        | Required for Free Tier usage                      |
| Free Tier VM           | `e2-micro`, 30GB standard persistent disk         |
| Domain Name            | You must own one (e.g., `yourdomain.com`)         |
| Subdomain (for n8n)    | E.g., `n8n.yourdomain.com`                         |
| Linux Basics           | Comfortable using terminal & SSH                  |

---

## ☁️ Phase 1: Google Cloud VM Setup

### 1.1 VM Instance Configuration

| Setting      | Value                         |
|--------------|-------------------------------|
| Name         | `n8n-server`                  |
| Region       | `us-west1` / `us-central1`    |
| Machine Type | `e2-micro`                    |
| OS           | `Ubuntu 22.04 LTS`            |
| Disk         | 30GB Standard Persistent Disk |
| Firewall     | Leave HTTP/HTTPS unchecked    |

> 🔧 After creation, reserve a **Static IP** under `VPC Network > IP addresses`.

---

### 1.2 Firewall Rules

| Rule Name            | Port | Protocol | Source      | Purpose                     |
|----------------------|------|----------|-------------|-----------------------------|
| `n8n-allow-http`     | 80   | TCP      | `0.0.0.0/0` | Allow HTTP traffic          |
| `n8n-allow-https`    | 443  | TCP      | `0.0.0.0/0` | Allow HTTPS traffic         |
| `n8n-allow-direct-test` | 5678 | TCP | `0.0.0.0/0` | TEMPORARY: Direct access to n8n |

> ⚠️ Delete the `5678` rule after setting up Caddy SSL.

---

## 🐳 Phase 2: Install Docker

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release -y

sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker $USER
```

> 🔁 Log out and back in after adding yourself to the Docker group.

---

## ⚙️ Phase 3: Run n8n with Docker

```bash
# Create volume for data persistence
docker volume create n8n_data

# Start container
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_HOST="n8n.yourdomain.com" \
  -e N8N_PROTOCOL="https" \
  -e WEBHOOK_URL="https://n8n.yourdomain.com/" \
  n8nio/n8n
```

```bash
# Confirm it's running
docker ps

# If needed
docker logs n8n
docker start n8n
```

---

## 🌐 Phase 4: DNS Configuration

Update your domain registrar with the following:

| Record Type | Name (Subdomain) | Value               | TTL  |
|-------------|------------------|---------------------|------|
| A           | `n8n`            | Your Static IP from GCP | Auto |

If using **Cloudflare**, set Proxy to "Proxied" ✅.

Check propagation: [dnschecker.org](https://dnschecker.org)

---

## 🔒 Phase 5: Install Caddy (HTTPS Proxy)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/caddy-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/caddy-archive-keyring.gpg] \
https://dl.cloudsmith.io/public/caddy/stable/deb/debian $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install caddy -y
```

---

### 5.1 Configure Caddyfile

```bash
sudo nano /etc/caddy/Caddyfile
```

Paste:

```
n8n.yourdomain.com {
    reverse_proxy localhost:5678
}
```

Save (Ctrl + O, Enter, Ctrl + X) and restart:

```bash
sudo systemctl enable caddy
sudo systemctl restart caddy
```

Check logs:

```bash
sudo journalctl -u caddy --since "5 minutes ago" --no-pager
```

Look for: ✅ `certificate obtained successfully`

---

## 🔁 Phase 6: Reboot Test

```bash
sudo shutdown now
```

After VM restarts:

```bash
# SSH back in
docker ps        # n8n should be running
systemctl status caddy  # Caddy should be active
```

---

## ✅ Final Steps

- 🧪 Visit `https://n8n.yourdomain.com/` in browser
- 🔒 Confirm HTTPS & n8n UI loads
- 🧹 Delete firewall rule for port `5678`

---

## 🎉 Success!

| ✅ Setup Complete For | ✔ |
|----------------------|---|
| Docker & n8n         | ✅ |
| HTTPS via Caddy      | ✅ |
| Auto-Start on Reboot | ✅ |
| GCP Free Tier        | ✅ |

---

> 💡 **Tip:** Want to backup workflows or enable basic auth for security? Let me know — I’ll help you enhance this setup!
