# Supabase Installation Guide for Debian 13 Virtual Machine

## Complete Walkthrough: Installing Supabase with Docker Compose on a VM

This guide will walk you through setting up Supabase on a Debian 13 virtual machine using Docker Compose.

---

## Prerequisites

### Virtual Machine Requirements

- **Hypervisor**: VMware, VirtualBox, Proxmox, KVM, Hyper-V, or cloud provider (AWS, Azure, GCP, DigitalOcean)
- **Operating System**: Fresh Debian 13 installation
- **CPU**: Minimum 2 cores (4+ recommended)
- **RAM**: Minimum 4GB (8GB+ recommended for production)
- **Disk Space**: 20GB+ free disk space (40GB+ recommended)
- **Network**: Bridged or NAT network adapter with internet access
- **Access**: Root or sudo access
- **SSH**: Enabled for remote access (optional but recommended)

---

## Initial VM Setup

### Step 0.1: Configure VM Network Settings

Depending on your hypervisor and use case, configure networking:

#### Option A: Bridged Network (Recommended for LAN Access)
- Allows VM to get an IP on your local network
- Makes VM accessible from other machines on your network
- Best for development/production environments

#### Option B: NAT with Port Forwarding
- VM hidden behind host machine
- Requires port forwarding rules for external access
- Good for isolated development

#### Example Port Forwarding Rules (if using NAT):
- Host Port 3000 → VM Port 3000 (Supabase Studio)
- Host Port 8000 → VM Port 8000 (API Gateway)
- Host Port 5432 → VM Port 5432 (PostgreSQL - optional)
- Host Port 22 → VM Port 22 (SSH - recommended)

### Step 0.2: Find Your VM's IP Address

After VM network is configured, find the IP address:

```bash
# Check IP address
ip addr show

# Or use hostname command
hostname -I
```

Make note of this IP address - you'll use it to access Supabase Studio and APIs.

### Step 0.3: Set Static IP (Recommended)

For production or persistent development, configure a static IP:

```bash
# Edit network configuration
sudo nano /etc/network/interfaces
```

Example static IP configuration:

```bash
auto ens33
iface ens33 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

Or if using netplan (newer Debian systems):

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Apply changes:

```bash
# For interfaces method
sudo systemctl restart networking

# For netplan method
sudo netplan apply
```

### Step 0.4: Configure Hostname (Optional)

Set a meaningful hostname for your VM:

```bash
# Set hostname
sudo hostnamectl set-hostname supabase-vm

# Edit hosts file
sudo nano /etc/hosts
```

Add this line:
```
127.0.1.1    supabase-vm
```

### Step 0.5: Enable SSH Access (Recommended)

If SSH is not already installed:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Test SSH connection from your host machine:

```bash
ssh username@YOUR_VM_IP
```

---

## Step 1: Update System Packages

First, ensure your system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 2: Install Required Dependencies

Install necessary packages for Docker installation:

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release git
```

---

## Step 3: Install Docker

### Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify Docker Installation

```bash
sudo docker --version
sudo docker compose version
```

### Add Your User to Docker Group (Optional)

This allows running Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

**Note:** Log out and back in for group changes to take full effect.

---

## Step 4: Install Supabase

### Clone the Supabase Repository

```bash
cd ~
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
```

### Copy Environment File

```bash
cp .env.example .env
```

---

## Step 5: Configure Supabase

### Generate Secure Secrets

You need to generate secure JWT secrets and passwords. Use the following commands:

```bash
# Generate JWT Secret (should be at least 32 characters)
openssl rand -base64 32

# Generate Postgres password
openssl rand -base64 32

# Generate service role key (JWT)
openssl rand -base64 32
```

### Edit the .env File

Open the `.env` file in your preferred text editor:

```bash
nano .env
```

### Update Critical Environment Variables

Locate and update the following variables with your generated secrets:

```bash
############
# Secrets
############
POSTGRES_PASSWORD=your_generated_postgres_password_here
JWT_SECRET=your_generated_jwt_secret_here
ANON_KEY=your_generated_anon_key_here
SERVICE_ROLE_KEY=your_generated_service_role_key_here

############
# Database
############
POSTGRES_HOST=db
POSTGRES_DB=postgres
POSTGRES_PORT=5432

############
# API
############
KONG_HTTP_PORT=8000
KONG_HTTPS_PORT=8443

############
# Studio
############
STUDIO_PORT=3000

############
# Site URL
############
SITE_URL=http://localhost:3000
API_EXTERNAL_URL=http://localhost:8000
```

### Important Configuration Notes

1. **POSTGRES_PASSWORD**: Strong password for PostgreSQL database
2. **JWT_SECRET**: Used to sign JWT tokens (minimum 32 characters)
3. **ANON_KEY**: Public API key for client-side requests
4. **SERVICE_ROLE_KEY**: Admin API key with full access (keep secure!)
5. **SITE_URL**: Your application's URL (update for production)

**Security Warning:** Never commit your `.env` file to version control or share these secrets publicly.

---

## Step 6: Start Supabase

### Pull Docker Images

```bash
docker compose pull
```

### Start All Services

```bash
docker compose up -d
```

This will start all Supabase services in detached mode.

### Verify Services Are Running

```bash
docker compose ps
```

You should see all services in a "running" state.

---

## Step 7: Access Supabase Studio

Once all services are running, you can access Supabase from your host machine or other devices on the network:

### Access URLs

Replace `YOUR_VM_IP` with your VM's actual IP address:

- **Supabase Studio (Dashboard)**: http://YOUR_VM_IP:3000
- **API Gateway (Kong)**: http://YOUR_VM_IP:8000
- **Database**: YOUR_VM_IP:5432

### Examples:

If your VM IP is `192.168.1.100`:
- Studio: http://192.168.1.100:3000
- API: http://192.168.1.100:8000
- Database: 192.168.1.100:5432

### Accessing from Host Machine

**Windows**:
- Open browser and navigate to http://YOUR_VM_IP:3000
- Or use Remote Desktop, VNC, or VM console

**macOS/Linux**:
- Open browser: http://YOUR_VM_IP:3000
- Or SSH into VM: `ssh username@YOUR_VM_IP`

### Configure Firewall (If Enabled)

If you have UFW or iptables enabled on your VM, allow the necessary ports:

```bash
# Allow Supabase Studio
sudo ufw allow 3000/tcp

# Allow API Gateway
sudo ufw allow 8000/tcp

# Allow PostgreSQL (only if needed externally)
sudo ufw allow 5432/tcp

# Allow SSH
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable
```

### Default Studio Credentials

On first access, you may need to use:
- **Email**: Set in your `.env` (or default)
- **Password**: Set in your `.env` (or default)

---

## Step 8: Configure VM-Specific Settings

### Update Environment Variables for VM Access

Edit your `.env` file to reflect your VM's IP address:

```bash
cd ~/supabase/docker
nano .env
```

Update these variables:

```bash
############
# Site URL - Use your VM IP
############
SITE_URL=http://YOUR_VM_IP:3000
API_EXTERNAL_URL=http://YOUR_VM_IP:8000

# For production with domain name:
# SITE_URL=https://yourdomain.com
# API_EXTERNAL_URL=https://api.yourdomain.com
```

After updating, restart services:

```bash
docker compose restart
```

### Configure CORS for External Access

If accessing from browsers on different machines, you may need to configure CORS. Edit `.env`:

```bash
ADDITIONAL_REDIRECT_URLS=http://YOUR_VM_IP:3000,http://localhost:3000
ADDITIONAL_CORS_ORIGINS=http://YOUR_VM_IP:3000,http://localhost:3000
```

### VM Snapshot (Recommended)

After successful installation, create a VM snapshot:

**VirtualBox**:
```
VM → Snapshots → Take Snapshot
```

**VMware**:
```
VM → Snapshot → Take Snapshot
```

This allows you to restore to a clean installation state if needed.

---

## Step 9: Verify Installation

### Check Service Health

```bash
# View logs
docker compose logs

# Check specific service logs
docker compose logs kong
docker compose logs postgres
docker compose logs studio
```

### Test API Connection

```bash
curl http://localhost:8000
```

You should receive a response from the Kong API gateway.

---

## Common Commands

### Stop Supabase

```bash
docker compose stop
```

### Start Supabase

```bash
docker compose start
```

### Restart Supabase

```bash
docker compose restart
```

### Stop and Remove Containers

```bash
docker compose down
```

### Stop and Remove Containers + Volumes (Full Reset)

**Warning:** This deletes all data!

```bash
docker compose down -v
```

### View Real-Time Logs

```bash
docker compose logs -f
```

### Update Supabase

```bash
cd ~/supabase/docker
git pull origin master
docker compose pull
docker compose up -d
```

---

## Troubleshooting

### Cannot Access Supabase Studio from Host Machine

1. **Verify VM IP address**:
   ```bash
   hostname -I
   ```

2. **Check if services are listening on all interfaces**:
   ```bash
   sudo netstat -tulpn | grep -E '(3000|8000|5432)'
   ```

3. **Test from within the VM first**:
   ```bash
   curl http://localhost:3000
   curl http://localhost:8000
   ```

4. **Check VM firewall rules**:
   ```bash
   sudo ufw status
   ```

5. **Verify network mode** (bridged vs NAT)

6. **Check host firewall** (Windows Firewall, macOS Firewall)

7. **Ping VM from host**:
   ```bash
   # From host machine
   ping YOUR_VM_IP
   ```

### VM Network Issues

**Bridged Network Not Working**:
- Ensure VM network adapter is set to "Bridged"
- Check if host's network adapter allows promiscuous mode
- Try different physical network adapter for bridging

**NAT Port Forwarding Not Working**:
- Verify port forwarding rules in hypervisor settings
- Ensure VM is using NAT network mode
- Check for port conflicts on host machine
- Restart VM after changing port forwarding rules

### Services Won't Start

1. Check if ports are already in use:
   ```bash
   sudo netstat -tulpn | grep -E '(3000|8000|5432)'
   ```

2. View service logs:
   ```bash
   docker compose logs
   ```

3. Check available resources:
   ```bash
   free -h  # Check RAM
   df -h    # Check disk space
   ```

### Permission Denied Errors

Ensure your user is in the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Database Connection Issues

1. Verify PostgreSQL is running:
   ```bash
   docker compose ps postgres
   ```

2. Check PostgreSQL logs:
   ```bash
   docker compose logs postgres
   ```

3. Test database connection from within VM:
   ```bash
   docker compose exec postgres psql -U postgres
   ```

### Out of Memory Errors

1. Check VM RAM allocation in hypervisor settings
2. Increase RAM to at least 8GB
3. Monitor memory usage:
   ```bash
   free -h
   htop
   ```

4. Adjust Docker memory limits in `/etc/docker/daemon.json`:
   ```json
   {
     "default-ulimits": {
       "memlock": {
         "hard": -1,
         "soft": -1
       }
     }
   }
   ```

### VM Performance Issues

1. **Increase CPU allocation**: Allocate at least 2-4 cores to VM
2. **Enable nested virtualization** (if using KVM/VMware)
3. **Use SSD storage**: Move VM to SSD if on HDD
4. **Disable unnecessary services**:
   ```bash
   sudo systemctl disable bluetooth
   sudo systemctl disable cups
   ```

### Slow Network Performance

1. Check network adapter type in VM settings (use virtio for KVM)
2. Enable jumbo frames if supported
3. Use bridged instead of NAT if possible
4. Increase network buffer sizes

---

## Production Considerations

### VM-Specific Production Setup

#### 1. Resource Allocation

For production workloads, allocate sufficient resources:

- **CPU**: 4+ cores
- **RAM**: 16GB+ (8GB minimum)
- **Disk**: 100GB+ SSD storage
- **Network**: Dedicated network interface if possible

#### 2. VM Backup Strategy

**Hypervisor-Level Backups**:
- Schedule regular VM snapshots
- Use incremental backups to save space
- Store backups on separate storage

**Application-Level Backups**:
```bash
# Backup script
#!/bin/bash
BACKUP_DIR="/backups/supabase"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
docker compose exec -T postgres pg_dump -U postgres postgres > $BACKUP_DIR/db_$DATE.sql

# Backup volumes
docker compose exec -T postgres tar czf - /var/lib/postgresql/data > $BACKUP_DIR/pgdata_$DATE.tar.gz

# Keep only last 7 days
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete
```

Schedule with cron:
```bash
crontab -e
# Add: 0 2 * * * /path/to/backup-script.sh
```

#### 3. VM Hardening

```bash
# Install fail2ban
sudo apt install -y fail2ban

# Configure SSH
sudo nano /etc/ssh/sshd_config
```

Recommended SSH settings:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222  # Non-standard port
```

```bash
# Restart SSH
sudo systemctl restart sshd
```

#### 4. Monitoring VM Resources

Install monitoring tools:

```bash
# Install htop for interactive monitoring
sudo apt install -y htop

# Install sysstat for historical data
sudo apt install -y sysstat
sudo systemctl enable sysstat
```

Monitor resources:
```bash
# CPU and memory
htop

# Disk usage
df -h
du -sh ~/supabase/docker/*

# Network stats
iftop
```

#### 5. VM Networking for Production

**Option A: Bridge with Static IP + Firewall**:
```bash
# Configure UFW
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw enable
```

**Option B: Reverse Proxy on Host**:
- Run nginx/Caddy on host machine
- Proxy requests to VM ports
- Easier SSL/TLS management

### Security Hardening

1. **Change all default passwords and secrets**
2. **Enable HTTPS** using a reverse proxy (nginx/Caddy)
3. **Configure firewall rules** to restrict access
4. **Use strong JWT secrets** (64+ characters)
5. **Regularly update** Supabase and Docker images
6. **Backup your database** regularly
7. **Enable monitoring** and logging
8. **Disable root SSH access**
9. **Use SSH keys** instead of passwords
10. **Enable automatic security updates**

### Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Reverse Proxy Setup (Nginx Example)

If running nginx on the host or in a separate VM:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://VM_IP:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://VM_IP:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Running Nginx in the Same VM

If you prefer to run nginx directly in the VM:

```bash
# Install nginx
sudo apt install -y nginx certbot python3-certbot-nginx

# Configure nginx
sudo nano /etc/nginx/sites-available/supabase
```

Add configuration:
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/supabase /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# Get SSL certificate
sudo certbot --nginx -d yourdomain.com
```

### Backup Database

```bash
docker compose exec postgres pg_dump -U postgres postgres > backup.sql
```

### Restore Database

```bash
docker compose exec -T postgres psql -U postgres postgres < backup.sql
```

---

## Quick Reference for VM Administration

### Essential Commands

```bash
# Check VM IP
hostname -I

# Check VM resources
free -h && df -h

# Supabase service management
cd ~/supabase/docker
docker compose ps                    # Check status
docker compose up -d                 # Start all services
docker compose down                  # Stop all services
docker compose restart               # Restart services
docker compose logs -f               # Follow logs

# System monitoring
htop                                 # Interactive process viewer
sudo netstat -tulpn                  # Check open ports
sudo ufw status                      # Firewall status

# Backup database
docker compose exec postgres pg_dump -U postgres postgres > backup.sql

# Update Supabase
git pull origin master
docker compose pull
docker compose up -d
```

### Common VM IP Addresses by Network Type

- **Bridged**: Usually 192.168.1.x or 192.168.0.x (same as host network)
- **NAT**: Usually 10.0.2.x (VirtualBox) or 192.168.x.x (VMware)
- **Host-Only**: Usually 192.168.56.x (VirtualBox) or 192.168.x.x (VMware)

### Access URLs Template

Replace `YOUR_VM_IP` with your actual VM IP address:

```
Studio:   http://YOUR_VM_IP:3000
API:      http://YOUR_VM_IP:8000
Database: YOUR_VM_IP:5432
```

### Hypervisor-Specific Notes

**VirtualBox**:
- Bridged adapter: Choose your active network adapter
- Port forwarding: Settings → Network → Advanced → Port Forwarding
- Shared folders: Devices → Shared Folders

**VMware Workstation/Player**:
- Bridged: Automatically connects to physical adapter
- NAT: Edit → Virtual Network Editor for port forwarding
- Snapshots: VM → Snapshot menu

**Proxmox**:
- Network: Choose bridge (vmbr0) for bridged networking
- Resources: Hardware tab in VM settings
- Backup: Backup tab or datacenter-level backup jobs

**KVM/QEMU**:
- Network: Bridge to br0 for network access
- Performance: Use virtio drivers
- Management: virt-manager GUI or virsh CLI

---

## Resources

- **Official Documentation**: https://supabase.com/docs
- **GitHub Repository**: https://github.com/supabase/supabase
- **Community Discord**: https://discord.supabase.com
- **Docker Documentation**: https://docs.docker.com

## VM Management Tips

### Allocating More Resources

If you need to increase VM resources:

1. **Shut down the VM completely**
2. **Edit VM settings in hypervisor**:
   - CPU: Increase core count
   - RAM: Increase memory allocation
   - Disk: Expand virtual disk
3. **Start VM and verify**:
   ```bash
   # Check CPU
   nproc
   
   # Check RAM
   free -h
   
   # Check disk (may need to expand partition)
   df -h
   ```

### Expanding VM Disk

If you need more storage space:

```bash
# After expanding virtual disk in hypervisor
# Check new disk size
sudo fdisk -l

# Resize partition (example for ext4)
sudo resize2fs /dev/sda1

# Verify
df -h
```

### Cloning Your Supabase VM

To create development copies:

1. **Shut down VM**
2. **Clone VM in hypervisor** (Full clone recommended)
3. **Start cloned VM**
4. **Change hostname and IP**:
   ```bash
   sudo hostnamectl set-hostname supabase-dev
   ```
5. **Regenerate SSH keys** (if needed):
   ```bash
   sudo rm /etc/ssh/ssh_host_*
   sudo dpkg-reconfigure openssh-server
   ```

### Moving VM to Different Host

1. **Export VM** (OVF/OVA format)
2. **Copy to new host**
3. **Import VM**
4. **Update network settings** for new environment
5. **Verify Supabase services**:
   ```bash
   docker compose ps
   ```

---

## Summary

You now have a fully functional Supabase instance running on a Debian 13 virtual machine with Docker Compose! The setup includes:

- PostgreSQL database
- RESTful API (PostgREST)
- Authentication service (GoTrue)
- Storage service
- Real-time subscriptions
- Supabase Studio (Web UI)

### Quick Access Checklist

✅ VM is running and accessible on network  
✅ Docker and Docker Compose are installed  
✅ Supabase services are running (`docker compose ps`)  
✅ Studio accessible at http://YOUR_VM_IP:3000  
✅ API accessible at http://YOUR_VM_IP:8000  
✅ Firewall configured (if enabled)  
✅ Backup strategy in place  
✅ VM snapshot created  

### Next Steps

1. **Configure your application** to connect to `http://YOUR_VM_IP:8000`
2. **Set up your database schema** in Studio
3. **Configure authentication** providers
4. **Set up storage buckets** for file uploads
5. **Create database policies** for Row Level Security
6. **Configure backups** for production use
7. **Set up monitoring** and logging
8. **Configure domain and SSL** for production

### Accessing From Development Machine

Configure your application's environment variables:

```bash
# .env for your application
SUPABASE_URL=http://YOUR_VM_IP:8000
SUPABASE_ANON_KEY=your_anon_key_from_.env
SUPABASE_SERVICE_KEY=your_service_role_key_from_.env
```

---

**Installation Type**: Virtual Machine  
**Last Updated**: February 2026  
**Supabase Version**: Latest (self-hosted)  
**Debian Version**: 13  
**Hypervisors Tested**: VMware, VirtualBox, Proxmox, KVM
