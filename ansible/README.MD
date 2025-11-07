# ICU Application - Ansible Deployment Automation

Automated infrastructure provisioning and deployment for the ICU (Intensive Care Unit) Application stack using Ansible. This setup deploys a complete production environment with Docker containers, reverse proxy, and automatic HTTPS.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Customization Guide](#customization-guide)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Usage & Commands](#usage--commands)
- [CI/CD Integration](#cicd-integration)
- [Roles Documentation](#roles-documentation)
- [Security Best Practices](#security-best-practices)
- [What Gets Deployed](#what-gets-deployed)

## Architecture Overview

This Ansible automation deploys a full-stack application on a single DigitalOcean droplet with the following architecture:

```
Internet
    |
    v
[Caddy Reverse Proxy] (Automatic HTTPS)
    |
    +---> Frontend (icu.darwishathibi.com) --> SvelteKit App (Port 5173)
    |
    +---> Backend (api-icu.darwishathibi.com) --> ASP.NET Core API (Port 5141)
                                                        |
                                                        v
                                                   MySQL 8.0 (Port 3306)
```

**Technology Stack:**
- **Frontend**: SvelteKit application (containerized)
- **Backend**: ASP.NET Core API (containerized)
- **Database**: MySQL 8.0 (containerized)
- **Reverse Proxy**: Caddy web server with automatic Let's Encrypt SSL
- **Container Runtime**: Docker with Docker Compose
- **Firewall**: UFW (Uncomplicated Firewall)

**Default Target Infrastructure** (customizable):
- Server: DigitalOcean Droplet (188.166.249.118) - *Configure in inventory/hosts.yml*
- Organization: Deta-Technologies - *Configure in group_vars/all.yml*
- Container Registry: GitHub Container Registry (ghcr.io)

> **Note**: These are default values. See the [Customization Guide](#customization-guide) to deploy to your own infrastructure.

## Prerequisites

### Control Machine (where Ansible runs)

1. **Ansible** installed (version 2.9 or higher)
   ```bash
   # Install on macOS
   brew install ansible

   # Install on Ubuntu/Debian
   sudo apt update && sudo apt install ansible
   ```

2. **SSH access** configured with key-based authentication
   ```bash
   # Generate SSH key if you don't have one
   ssh-keygen -t ed25519 -C "your_email@example.com"

   # Copy public key to server
   ssh-copy-id -i ~/.ssh/id_ed25519 root@188.166.249.118
   ```

3. **Environment variables** set with sensitive credentials
   ```bash
   export MYSQL_ROOT_PASSWORD="your_secure_root_password"
   export MYSQL_PASSWORD="your_database_password"
   export GITHUB_TOKEN="ghp_your_github_personal_access_token"
   export GITHUB_USERNAME="your_github_username"
   ```

### Target Server

- **Operating System**: Ubuntu 20.04/22.04 or Debian-based system
- **Access**: Root access or sudo privileges
- **Network**: SSH access enabled (port 22)
- **Connectivity**: Internet access for package downloads

### External Dependencies

1. **DNS Configuration**: Your domains must point to your server IP
   - Example: `icu.darwishathibi.com` → 188.166.249.118
   - Example: `api-icu.darwishathibi.com` → 188.166.249.118
   - *See [Customization Guide](#customization-guide) for your own domains*

2. **GitHub Container Registry**: Access to private Docker images
   - Images hosted at `ghcr.io/deta-technologies/`
   - Requires valid GitHub token with `read:packages` permission

3. **Let's Encrypt**: Automatic SSL requires DNS to be properly configured

## Customization Guide

**This section is for users who want to deploy to their own infrastructure with different domains, IP addresses, and ports.**

### Overview

The current configuration deploys to:
- **Server IP**: 188.166.249.118
- **Frontend Domain**: icu.darwishathibi.com (Port 5173)
- **Backend Domain**: api-icu.darwishathibi.com (Port 5141)

To customize for your environment, you need to modify **2 main files** and configure **DNS settings**.

### Step-by-Step Customization

#### 1. Update Inventory (Server IP and SSH Settings)

**File**: `inventory/hosts.yml`

**What to change**:
```yaml
all:
  hosts:
    icu_server:
      ansible_host: 188.166.249.118        # ← CHANGE THIS to your server IP
      ansible_user: root                    # ← Change if using different user
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519  # ← Update SSH key path
      ansible_python_interpreter: /usr/bin/python3
```

**Example for your own server**:
```yaml
all:
  hosts:
    icu_server:
      ansible_host: 203.0.113.50           # Your DigitalOcean/AWS/VPS IP
      ansible_user: ubuntu                  # or root, admin, etc.
      ansible_ssh_private_key_file: ~/.ssh/my_key
      ansible_python_interpreter: /usr/bin/python3
```

#### 2. Update Variables (Domains and Ports)

**File**: `group_vars/all.yml`

**What to change**:
```yaml
# Domain Configuration
frontend_domain: "icu.darwishathibi.com"     # ← CHANGE THIS to your domain
backend_domain: "api-icu.darwishathibi.com"  # ← CHANGE THIS to your API domain

# Port Configuration (optional - only change if your app uses different ports)
frontend_port: 5173    # ← Change if your frontend uses different port
backend_port: 5141     # ← Change if your backend uses different port

# Database Configuration
mysql_database: "icu_db"        # ← Change database name if desired
mysql_user: "mandur_gede"       # ← Change database user if desired

# Application Configuration
app_directory: "/opt/icu_application"  # ← Change deployment path if desired
github_org: "Deta-Technologies"        # ← CHANGE THIS to your GitHub org/username
```

**Example for your own setup**:
```yaml
# Domain Configuration
frontend_domain: "myapp.example.com"
backend_domain: "api.myapp.example.com"

# Port Configuration
frontend_port: 3000    # If using Next.js or different port
backend_port: 8080     # If using different backend port

# Database Configuration
mysql_database: "myapp_production"
mysql_user: "myapp_user"

# Application Configuration
app_directory: "/opt/myapp"
github_org: "your-github-username"
```

#### 3. Configure DNS Records

Before running the playbook, ensure your domains point to your server:

**Required DNS A Records**:
```
myapp.example.com       →  203.0.113.50 (your server IP)
api.myapp.example.com   →  203.0.113.50 (your server IP)
```

**How to verify DNS**:
```bash
# Check if DNS is properly configured
dig myapp.example.com +short
# Should return: 203.0.113.50

dig api.myapp.example.com +short
# Should return: 203.0.113.50
```

**Note**: DNS propagation can take 5 minutes to 48 hours. Let's Encrypt SSL will only work after DNS is properly configured.

#### 4. Update Docker Image References (if using different images)

If you're using your own Docker images (not Deta-Technologies images), you'll need to update the image references:

**File**: `roles/application/templates/docker-compose.yml.j2`

Look for these lines and update:
```yaml
services:
  backend:
    image: ghcr.io/{{ github_org }}/icu-backend:latest  # Auto-uses your github_org
    # This becomes: ghcr.io/your-username/icu-backend:latest

  frontend:
    image: ghcr.io/{{ github_org }}/icu-frontend:latest  # Auto-uses your github_org
    # This becomes: ghcr.io/your-username/icu-frontend:latest
```

The images will automatically use your `github_org` variable, so you only need to ensure:
1. Your images are pushed to GitHub Container Registry
2. Your images are named `icu-backend` and `icu-frontend` (or update the template)

**If using different image names**:
```yaml
# Change from:
image: ghcr.io/{{ github_org }}/icu-backend:latest

# To (example):
image: ghcr.io/{{ github_org }}/my-api-service:latest
```

### Quick Customization Checklist

Use this checklist to ensure you've customized everything:

- [ ] **Updated `inventory/hosts.yml`** with your server IP
- [ ] **Updated `group_vars/all.yml`** with your domains
- [ ] **Updated `group_vars/all.yml`** with your GitHub organization
- [ ] **Configured DNS** A records for both domains
- [ ] **Verified DNS** propagation with `dig` command
- [ ] **Set up SSH** key-based authentication to your server
- [ ] **Set environment variables** with your credentials
- [ ] **Updated ports** in `group_vars/all.yml` (if different)
- [ ] **Updated app_directory** path (if desired)
- [ ] **Updated GitHub image names** (if using different names)

### Multi-Environment Setup (Optional)

If you want to manage multiple environments (staging, production), create separate inventory files:

**Directory structure**:
```
inventory/
├── production.yml
└── staging.yml
```

**Example `inventory/staging.yml`**:
```yaml
all:
  hosts:
    staging_server:
      ansible_host: 198.51.100.10
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_python_interpreter: /usr/bin/python3
```

**Example `inventory/production.yml`**:
```yaml
all:
  hosts:
    production_server:
      ansible_host: 203.0.113.50
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_python_interpreter: /usr/bin/python3
```

**Then use different variable files**:
```
group_vars/
├── all.yml          # Shared variables
├── staging.yml      # Staging-specific variables
└── production.yml   # Production-specific variables
```

**Deploy to specific environment**:
```bash
# Deploy to staging
ansible-playbook -i inventory/staging.yml playbook.yml

# Deploy to production
ansible-playbook -i inventory/production.yml playbook.yml
```

### Common Customization Scenarios

#### Scenario 1: Different Cloud Provider (AWS, Azure, etc.)

No changes needed to the playbook! Just update:
- `inventory/hosts.yml` with your cloud VM's IP
- DNS records to point to that IP
- SSH key to match your cloud provider's key

#### Scenario 2: Different Application Ports

Your app runs on port 8080 instead of 5141?

Update `group_vars/all.yml`:
```yaml
backend_port: 8080
```

Ensure your Docker image exposes this port.

#### Scenario 3: Subdomain vs Separate Domains

Want to use `app.example.com` and `app.example.com/api`?

This requires additional Caddy configuration. Update `roles/caddy/templates/Caddyfile.j2`:
```
app.example.com {
    # Proxy /api to backend
    reverse_proxy /api/* localhost:{{ backend_port }}

    # Everything else to frontend
    reverse_proxy localhost:{{ frontend_port }}
}
```

#### Scenario 4: Different Database

Using PostgreSQL instead of MySQL?

You'll need to:
1. Modify `roles/application/templates/docker-compose.yml.j2`
2. Replace MySQL service with PostgreSQL
3. Update connection strings in environment variables

### Troubleshooting Customization

**Issue**: Playbook fails with "Host unreachable"
- Verify server IP is correct in `inventory/hosts.yml`
- Test SSH manually: `ssh -i ~/.ssh/id_ed25519 user@your-server-ip`

**Issue**: SSL certificate fails
- Ensure DNS is properly configured and propagated
- Check DNS: `dig your-domain.com +short`
- Wait 10-15 minutes after DNS changes

**Issue**: Wrong ports exposed
- Check `group_vars/all.yml` port configuration
- Verify your Docker images expose the correct ports
- Restart Caddy: `ssh root@server "systemctl restart caddy"`

## Quick Start

> **First time user?** Make sure to complete the [Customization Guide](#customization-guide) above before proceeding!

### Before You Begin

Ensure you have customized these files for your environment:
1. `inventory/hosts.yml` - Your server IP and SSH settings
2. `group_vars/all.yml` - Your domains, ports, and GitHub organization
3. DNS records configured for your domains

### 1. Clone and Navigate to Ansible Directory

```bash
cd /path/to/infrastructure/ansible
```

### 2. Set Required Environment Variables

```bash
export MYSQL_ROOT_PASSWORD="your_secure_root_password"
export MYSQL_PASSWORD="your_database_password"
export GITHUB_TOKEN="your_github_token"
export GITHUB_USERNAME="your_github_username"
```

### 3. Verify SSH Access

```bash
ssh -i ~/.ssh/id_ed25519 root@188.166.249.118
```

### 4. Run the Playbook

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml
```

### 5. Verify Deployment

After successful deployment, verify the services are running:

```bash
# SSH into server
ssh root@188.166.249.118

# Check running containers
docker ps

# Check container logs
cd /opt/icu_application
docker-compose logs -f

# Access the applications
# Frontend: https://icu.darwishathibi.com
# Backend: https://api-icu.darwishathibi.com
```

## Project Structure

```
ansible/
├── inventory/
│   └── hosts.yml                    # Server inventory configuration
├── group_vars/
│   └── all.yml                      # Global variables for all hosts
├── playbook.yml                     # Main orchestration playbook
└── roles/
    ├── common/                      # System setup and firewall
    │   └── tasks/
    │       └── main.yml
    ├── docker/                      # Docker installation
    │   └── tasks/
    │       └── main.yml
    ├── caddy/                       # Reverse proxy configuration
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       └── Caddyfile.j2
    └── application/                 # Application deployment
        ├── tasks/
        │   └── main.yml
        └── templates/
            └── docker-compose.yml.j2
```

### Directory Breakdown

- **inventory/**: Defines target servers and connection details
- **group_vars/**: Variables applied to all hosts (domains, ports, credentials)
- **playbook.yml**: Main playbook that orchestrates all roles
- **roles/**: Modular task collections for different infrastructure components

## Configuration

### Inventory Configuration

Edit `inventory/hosts.yml` to configure target server:

```yaml
all:
  hosts:
    icu_server:
      ansible_host: 188.166.249.118  # CUSTOMIZE: Your server IP address
      ansible_user: root              # CUSTOMIZE: SSH user (root, ubuntu, etc.)
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519  # CUSTOMIZE: Path to SSH key
      ansible_python_interpreter: /usr/bin/python3
```

> **Important**: See the [Customization Guide](#customization-guide) for step-by-step instructions on customizing for your server.

### Variables Configuration

Edit `group_vars/all.yml` to customize deployment:

```yaml
# Domain Configuration (CUSTOMIZE THESE)
frontend_domain: "icu.darwishathibi.com"     # Your frontend domain
backend_domain: "api-icu.darwishathibi.com"  # Your backend API domain

# Port Configuration (CUSTOMIZE IF NEEDED)
frontend_port: 5173    # Port your frontend container exposes
backend_port: 5141     # Port your backend container exposes

# Docker Configuration
docker_compose_version: "3.8"

# Database Configuration (values from environment)
mysql_root_password: "{{ lookup('env', 'MYSQL_ROOT_PASSWORD') }}"
mysql_database: "icu_db"              # CUSTOMIZE database name
mysql_user: "mandur_gede"             # CUSTOMIZE database user
mysql_password: "{{ lookup('env', 'MYSQL_PASSWORD') }}"

# Application Configuration (CUSTOMIZE THESE)
app_directory: "/opt/icu_application"  # Where to deploy on server
github_org: "Deta-Technologies"        # Your GitHub org/username
```

> **Important**: See the [Customization Guide](#customization-guide) for detailed instructions on customizing for your infrastructure.

### Environment Variables

Create a `.env` file or export these variables before running:

```bash
# Database credentials
export MYSQL_ROOT_PASSWORD="your_secure_root_password"
export MYSQL_PASSWORD="your_database_password"

# GitHub Container Registry access
export GITHUB_TOKEN="ghp_xxxxxxxxxxxx"
export GITHUB_USERNAME="your_github_username"
```

## Usage & Commands

### Full Deployment

Deploy complete infrastructure from scratch:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml
```

### Dry Run (Check Mode)

Preview changes without applying them:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml --check
```

### Run Specific Roles

Execute only specific roles using tags:

```bash
# Only install Docker
ansible-playbook -i inventory/hosts.yml playbook.yml --tags docker

# Only update Caddy configuration
ansible-playbook -i inventory/hosts.yml playbook.yml --tags caddy

# Only deploy application
ansible-playbook -i inventory/hosts.yml playbook.yml --tags application
```

### Verbose Output

Get detailed execution information:

```bash
# Verbose mode
ansible-playbook -i inventory/hosts.yml playbook.yml -v

# Very verbose (includes task details)
ansible-playbook -i inventory/hosts.yml playbook.yml -vv

# Debug level (shows everything)
ansible-playbook -i inventory/hosts.yml playbook.yml -vvv
```

### Update Application Only

To update just the application (pull new images and restart):

```bash
# SSH into server
ssh root@188.166.249.118

# Navigate to application directory
cd /opt/icu_application

# Pull latest images
docker-compose pull

# Restart services
docker-compose up -d

# View logs
docker-compose logs -f
```

## CI/CD Integration

### GitHub Actions Workflow

Integrate this Ansible playbook with GitHub Actions for automated deployments.

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Deploy ICU Application

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build-and-push:
    name: Build and Push Docker Images
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push backend image
        run: |
          docker build -t ghcr.io/deta-technologies/icu-backend:latest ./backend
          docker push ghcr.io/deta-technologies/icu-backend:latest

      - name: Build and push frontend image
        run: |
          docker build -t ghcr.io/deta-technologies/icu-frontend:latest ./frontend
          docker push ghcr.io/deta-technologies/icu-frontend:latest

  deploy:
    name: Deploy to Production
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - name: Checkout infrastructure repo
        uses: actions/checkout@v4
        with:
          repository: your-org/infrastructure
          token: ${{ secrets.GH_PAT }}

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install Ansible
        run: |
          pip install ansible

      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan 188.166.249.118 >> ~/.ssh/known_hosts

      - name: Run Ansible playbook
        env:
          MYSQL_ROOT_PASSWORD: ${{ secrets.MYSQL_ROOT_PASSWORD }}
          MYSQL_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
          GITHUB_TOKEN: ${{ secrets.GH_PAT }}
          GITHUB_USERNAME: ${{ github.actor }}
        run: |
          cd ansible
          ansible-playbook -i inventory/hosts.yml playbook.yml
```

### Required GitHub Secrets

Add these secrets to your GitHub repository (Settings → Secrets and variables → Actions):

- `SSH_PRIVATE_KEY`: Your SSH private key for server access
- `MYSQL_ROOT_PASSWORD`: MySQL root password
- `MYSQL_PASSWORD`: MySQL application user password
- `GH_PAT`: GitHub Personal Access Token with `read:packages` and `repo` permissions

### Simplified Update Workflow

For faster deployments after initial setup, create `.github/workflows/update.yml`:

```yaml
name: Update Application

on:
  workflow_dispatch:

jobs:
  update:
    name: Pull and Restart Containers
    runs-on: ubuntu-latest

    steps:
      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan 188.166.249.118 >> ~/.ssh/known_hosts

      - name: Deploy latest images
        run: |
          ssh root@188.166.249.118 << 'EOF'
            cd /opt/icu_application
            docker-compose pull
            docker-compose up -d
            docker-compose logs -f --tail=50
          EOF
```

## Roles Documentation

### Common Role

**Purpose**: System foundation setup

**Tasks**:
- Updates all system packages
- Installs essential dependencies (git, curl, python3-pip)
- Configures UFW firewall:
  - Allows SSH (22), HTTP (80), HTTPS (443)
  - Denies all other incoming traffic

**Idempotency**: Safe to run multiple times

**Location**: `roles/common/tasks/main.yml`

---

### Docker Role

**Purpose**: Install Docker container runtime

**Tasks**:
- Adds Docker official GPG key and repository
- Installs Docker CE, Docker Compose plugin, and Buildx
- Configures Docker service to start on boot
- Adds ansible user to docker group
- Installs Docker Compose v2.24.0 standalone

**Idempotency**: Only installs if not already present

**Verification**: Displays Docker version after installation

**Location**: `roles/docker/tasks/main.yml`

---

### Caddy Role

**Purpose**: Reverse proxy with automatic HTTPS

**Tasks**:
- Installs Caddy web server from official repository
- Deploys Caddyfile configuration from template
- Configures reverse proxy for frontend and backend
- Enables automatic Let's Encrypt SSL certificates
- Sets up CORS headers for API

**Template**: `roles/caddy/templates/Caddyfile.j2`

**Features**:
- Automatic HTTPS with certificate renewal
- Gzip compression
- HTTP/2 support
- CORS configuration

**Handler**: Reloads Caddy when configuration changes

**Location**: `roles/caddy/tasks/main.yml`

---

### Application Role

**Purpose**: Deploy ICU application stack

**Tasks**:
- Creates application directory at `/opt/icu_application`
- Creates persistent storage for MySQL data and logs
- Deploys docker-compose.yml from template
- Creates secure .env file with credentials (mode 0600)
- Logs into GitHub Container Registry
- Pulls and starts all containers

**Template**: `roles/application/templates/docker-compose.yml.j2`

**Services Deployed**:
- MySQL 8.0 database
- ASP.NET Core backend API
- SvelteKit frontend application

**Location**: `roles/application/tasks/main.yml`

## Security Best Practices

### 1. Credential Management

- **Never commit credentials**: Use environment variables or Ansible Vault
- **Use Ansible Vault for secrets** (optional but recommended):
  ```bash
  # Create encrypted variables file
  ansible-vault create group_vars/secrets.yml

  # Run playbook with vault password
  ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass
  ```

### 2. SSH Security

- **Use SSH keys, not passwords**: Key-based authentication is enforced
- **Restrict SSH key permissions**:
  ```bash
  chmod 600 ~/.ssh/id_ed25519
  chmod 644 ~/.ssh/id_ed25519.pub
  ```

### 3. Firewall Configuration

UFW firewall is configured to:
- Allow only SSH (22), HTTP (80), and HTTPS (443)
- Deny all other incoming traffic
- Allow all outgoing traffic

### 4. Docker Security

- **Non-root containers**: Consider running containers as non-root users
- **Regular updates**: Keep Docker images updated
- **Image scanning**: Scan images for vulnerabilities

### 5. Database Security

- **.env file permissions**: Set to 0600 (owner read/write only)
- **Strong passwords**: Use complex passwords for MySQL
- **Network isolation**: MySQL only accessible within Docker network

### 6. HTTPS/SSL

- **Automatic certificates**: Caddy handles Let's Encrypt automatically
- **Certificate renewal**: Automatic, no manual intervention needed
- **Force HTTPS**: Caddy automatically redirects HTTP to HTTPS

## What Gets Deployed

### Services Overview

After successful deployment, the following services will be running:

#### 1. MySQL Database
- **Container**: `icu-mysql`
- **Port**: 3306 (internal only)
- **Version**: MySQL 8.0
- **Database**: `icu_db`
- **User**: `mandur_gede`
- **Data**: Persisted in `/opt/icu_application/mysql-data`
- **Health Check**: Automated with retry logic

#### 2. Backend API
- **Container**: `icu-backend`
- **Port**: 5141 (proxied via Caddy)
- **Runtime**: ASP.NET Core
- **Domain**: `https://api-icu.darwishathibi.com`
- **Environment**: Production mode
- **Logs**: Stored in `/opt/icu_application/logs`
- **Dependencies**: Waits for healthy MySQL

#### 3. Frontend Application
- **Container**: `icu-frontend`
- **Port**: 5173 (proxied via Caddy)
- **Framework**: SvelteKit
- **Domain**: `https://icu.darwishathibi.com`
- **Environment**: Production mode
- **API Connection**: `https://api-icu.darwishathibi.com`
- **Dependencies**: Waits for backend

#### 4. Caddy Reverse Proxy
- **Service**: System service (systemd)
- **Ports**: 80 (HTTP), 443 (HTTPS)
- **Configuration**: `/etc/caddy/Caddyfile`
- **SSL Certificates**: Automatic via Let's Encrypt
- **Features**: Gzip compression, CORS, automatic HTTPS

### Port Mappings

| Service  | Internal Port | External Access           |
|----------|---------------|---------------------------|
| Frontend | 5173          | icu.darwishathibi.com:443 |
| Backend  | 5141          | api-icu.darwishathibi.com:443 |
| MySQL    | 3306          | Internal only (Docker network) |
| Caddy    | 80, 443       | Public |

### Data Persistence

Persistent data is stored at `/opt/icu_application/`:

```
/opt/icu_application/
├── mysql-data/              # MySQL database files (persistent)
├── logs/                    # Application logs
├── docker-compose.yml       # Service orchestration
└── .env                     # Environment variables (secure)
```

### Automatic Restart

All containers are configured with restart policies:
- Containers restart automatically on failure
- Containers start automatically on server reboot
- Docker service starts on system boot

---

## Troubleshooting

### Check Service Status

```bash
# SSH into server
ssh root@188.166.249.118

# Check all containers
docker ps

# Check specific container logs
docker logs icu-frontend
docker logs icu-backend
docker logs icu-mysql

# Check Caddy status
systemctl status caddy
```

### Common Issues

**Issue**: Containers not starting
```bash
# Check docker-compose logs
cd /opt/icu_application
docker-compose logs

# Restart services
docker-compose down
docker-compose up -d
```

**Issue**: SSL certificate not working
- Verify DNS records point to server IP
- Check Caddy logs: `journalctl -u caddy -f`
- Ensure ports 80 and 443 are open

**Issue**: Database connection failed
```bash
# Verify MySQL is healthy
docker exec icu-mysql mysql -u root -p -e "SELECT 1;"

# Check MySQL logs
docker logs icu-mysql
```

---

**Maintained by**: Deta Technologies
**Last Updated**: January 2025
**Ansible Version**: 2.9+
