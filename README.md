# n8n Docker Setup with Caddy HTTPS Reverse Proxy

A complete guide to deploy n8n automation platform using Docker with automatic HTTPS via Caddy reverse proxy.

## 🚀 Features

- ✅ n8n running in Docker container
- ✅ Automatic HTTPS with Let's Encrypt
- ✅ Caddy reverse proxy for SSL termination
- ✅ Persistent data storage
- ✅ Auto-restart containers
- ✅ Production-ready configuration

## 📋 Prerequisites

- Ubuntu/Debian server with root access
- Domain name pointing to your server
- Ports 80 and 443 open in firewall

## 🛠️ Installation

### Step 1: Update System & Install Docker

```bash
# Update package index
sudo apt update

# Install Docker
sudo apt install -y docker.io

# Enable Docker service
sudo systemctl enable docker

# Add current user to docker group (requires logout/login)
sudo usermod -aG docker $(whoami)
```

> **Note:** Log out and log back in after running the usermod command to apply group changes.

### Step 2: Create Persistent Data Volume

```bash
# Create Docker volume for n8n data persistence
sudo docker volume create n8n_data
```

### Step 3: Deploy n8n Container

Replace `n8n.yourdomain.com` with your actual domain name:

```bash
sudo docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_HOST=n8n.yourdomain.com \
  -e N8N_PROTOCOL=https \
  -e WEBHOOK_URL=https://n8n.yourdomain.com/ \
  n8nio/n8n
```

### Step 4: Install & Configure Caddy

#### 4.1 Install Caddy Web Server

```bash
# Install dependencies
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Add Caddy repository
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Update package index and install Caddy
sudo apt update
sudo apt install -y caddy
```

#### 4.2 Configure Reverse Proxy

Create the Caddyfile configuration:

```bash
sudo nano /etc/caddy/Caddyfile
```

Add the following configuration (replace with your domain):

```caddyfile
n8n.yourdomain.com {
    reverse_proxy localhost:5678
}
```

**Save and exit:** Press `Ctrl+O` → `Enter` → `Ctrl+X`

#### 4.3 Start Caddy Service

```bash
# Validate configuration
sudo caddy validate --config /etc/caddy/Caddyfile

# Enable and start Caddy service
sudo systemctl enable caddy
sudo systemctl start caddy
sudo systemctl restart caddy
```

## ✅ Verification

Your n8n instance should now be accessible at:

**🌐 https://n8n.yourdomain.com**

## 🔧 Useful Commands

### Container Management
```bash
# Check n8n container status
docker ps | grep n8n

# View n8n logs
docker logs n8n

# Restart n8n container
docker restart n8n

# Stop n8n container
docker stop n8n
```

### Caddy Management
```bash
# Check Caddy status
sudo systemctl status caddy

# Restart Caddy
sudo systemctl restart caddy

# View Caddy logs
sudo journalctl -u caddy -f
```

### Data Management
```bash
# Backup n8n data
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar czf /backup/n8n-backup.tar.gz /data

# Restore n8n data
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar xzf /backup/n8n-backup.tar.gz -C /
```

## 🔒 Security Notes

- Caddy automatically obtains and renews SSL certificates via Let's Encrypt
- n8n data is persisted in Docker volume `n8n_data`
- Container automatically restarts unless explicitly stopped
- Default n8n port (5678) is only accessible via reverse proxy

## 🚨 Troubleshooting

### Common Issues

**SSL Certificate Issues:**
```bash
# Check Caddy logs for certificate errors
sudo journalctl -u caddy -f
```

**n8n Not Accessible:**
```bash
# Verify container is running
docker ps

# Check container logs
docker logs n8n
```

**Domain Not Resolving:**
- Ensure DNS A record points to your server IP
- Allow ports 80 and 443 in firewall
- Wait for DNS propagation (up to 24 hours)

## 📝 Configuration Details

| Component | Port | Purpose |
|-----------|------|---------|
| n8n | 5678 | Internal application port |
| Caddy | 80/443 | HTTP/HTTPS traffic |

## 🤝 Contributing

Feel free to submit issues and enhancement requests!

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Happy Automating! 🎉**
