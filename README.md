# LXD Deployment Guide

This document details the setup and configuration of LXD with ZFS storage on Ubuntu 22.04.

## 1. Prerequisites

### 1.1 Operating System
- **OS:** Ubuntu 22.04 LTS
- **User:** User with `sudo` privileges

### 1.2 Storage Preparation
LXD requires a dedicated partition or disk for the ZFS storage pool to function optimally.

**Disk Partitioning Strategy:**
- **System Drive:** Standard Linux installation (Ext4/XFS).
- **LXD Drive:** Leave a partition or entire disk **unformatted**.
  - Example: `/dev/sdb` (3.34TB) left unformatted during OS install.

## 2. Installation

### 2.1 Install LXD

LXD is distributed as a snap package. We recommend using the 5.21 LTS release and pinning the version to prevent unexpected automatic updates.

Install LXD 5.21 LTS:

```bash
sudo snap install lxd --channel=5.21/stable
```

Prevent automatic updates for the LXD snap:

```bash
sudo snap refresh --hold lxd
```

### 2.2 Initialize LXD
Run the initialization wizard to configure the daemon and storage.

```bash
sudo lxd init
```

**Configuration Reference:**
- **Clustering:** `no`
- **Storage Pool:**
  - Name: `default`
  - Backend: `zfs`
  - Create new pool: `yes`
  - Source: `/dev/sdb` (or your unformatted device)
- **Network:**
  - Bridge: `lxdbr0`
  - IPv4: `auto`
  - IPv6: `none`
- **Updates:** `yes`

## 3. Configuration

### 3.1 ZFS Optimization

Tune the ZFS pool for performance.

Enable LZ4 compression:

```bash
sudo zfs set compression=lz4 default
```

Disable access time updates to reduce IO:

```bash
sudo zfs set atime=off default
```

### 3.2 Dataset Management

Create datasets to organize virtual machines and data.

Create datasets:

```bash
sudo zfs create default/vms
sudo zfs create default/data
```

Set mountpoints (optional):

```bash
sudo zfs set mountpoint=/data default/data
```

### 3.3 User Permissions
Grant the current user permission to manage LXD.

```bash
sudo usermod -aG lxd $USER
newgrp lxd
```

## 4. Web Interface (LXD-UI)

Enable the built-in web interface for management.

### 4.1 Enable HTTPS
The UI requires the API to be exposed over HTTPS.

```bash
lxc config set core.https_address "[::]:8443"
```

### 4.2 Access
- **URL:** `https://<server-ip>:8443`
- **Authentication:** By default, LXD uses client certificates. You can also configure OIDC or trust passwords.

## 5. MS365 OIDC Authentication

Enable Microsoft 365 (Azure AD) OIDC authentication for centralized identity management and group-based access control.

### 5.1 Register Application in Azure AD

1. Go to [Azure Portal](https://portal.azure.com)
2. Navigate to **Azure Active Directory** → **App registrations** → **New registration**
3. Configure the application:
   - **Name:** `LXD` (or your preferred name)
   - **Supported account types:** `Accounts in this organizational directory only`
   - **Redirect URI:** `https://lxd.yourdomain.com/auth/oidc/callback`
4. Click **Register**

### 5.2 Configure Application Credentials

1. In the app registration, go to **Certificates & secrets**
2. Create a new **Client secret**:
   - **Description:** `LXD OIDC Secret`
   - **Expires:** Select desired expiration (e.g., 24 months)
3. Copy the **Value** (client secret) and **Client ID** from the Overview tab - save these securely

### 5.3 Configure API Permissions

1. In the app registration, go to **API permissions**
2. Add the following permissions:
   - **Microsoft Graph** → **Delegated permissions**:
     - `openid`
     - `profile`
     - `email`
     - `User.Read`
     - `GroupMember.Read.All` (for group-based access)
3. Click **Grant admin consent for [your organization]**

### 5.4 Configure LXD OIDC Settings

Configure OIDC authentication on the LXD server:

```bash
# Set OIDC issuer (Azure AD tenant endpoint)
lxc config set oidc.issuer https://login.microsoftonline.com/<TENANT_ID>/v2.0

# Set OIDC client ID
lxc config set oidc.client.id <CLIENT_ID>

# Set OIDC client secret
lxc config set oidc.client.secret <CLIENT_SECRET>

# Set OIDC redirect URL
lxc config set oidc.redirect.url https://lxd.yourdomain.com/auth/oidc/callback
```

**Variables:**
- `<TENANT_ID>`: Your Azure AD tenant ID (found in Azure Portal)
- `<CLIENT_ID>`: Application client ID
- `<CLIENT_SECRET>`: Client secret value

### 5.5 Add MS365 User Group to LXD Admins

To grant admin access to a specific MS365 group:

1. First, get the Object ID of your MS365 group from Azure AD:
   - Navigate to **Azure Active Directory** → **Groups** → Select your group
   - Copy the **Object ID**

2. Create the identity provider group in LXD:

```bash
lxc auth identity-provider-group create <GROUP_OBJECT_ID>
```

3. Add the group to the admins role:

```bash
lxc auth identity-provider-group group add <GROUP_OBJECT_ID> admins
```

**Example:**

```bash
lxc auth identity-provider-group create aaf5fa57-02bc-4ab2-9229-0dbc26c672b6
lxc auth identity-provider-group group add aaf5fa57-02bc-4ab2-9229-0dbc26c672b6 admins
```

**Variables:**
- `<GROUP_OBJECT_ID>`: The Object ID of your MS365 group from Azure AD

Users in the specified MS365 group will automatically have admin access when authenticating via OIDC.

### 5.6 Verify OIDC Configuration

Test OIDC authentication:

```bash
# View OIDC settings
lxc config get oidc.issuer
lxc config get oidc.client.id

# Check trust entries
lxc config trust list
```

Access LXD WebUI and test login with your MS365 credentials:

```bash
https://lxd.yourdomain.com
```

When prompted, select **OIDC** as the authentication method and sign in with your MS365 account.

## 6. Validation

Launch a test Virtual Machine to verify the stack.

```bash
lxc launch ubuntu:22.04 test-vm --vm
```

Access VM:

```bash
lxc exec test-vm -- bash
```

## 7. Reverse Proxy for LXD WebUI

Configure Nginx Proxy Manager via Docker to provide secure reverse proxy access to the LXD WebUI.

### 7.1 Install Docker CE

Add Docker repository:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker CE:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Add current user to docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker installation:

```bash
docker --version
```

### 7.2 Install Nginx Proxy Manager

Create directory structure:

```bash
sudo mkdir -p /opt/npm
cd /opt/npm
```

Create docker-compose.yml:

```bash
sudo tee /opt/npm/docker-compose.yml > /dev/null <<'EOF'
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF
```

Create data directories:

```bash
sudo mkdir -p /opt/npm/data
sudo mkdir -p /opt/npm/letsencrypt
sudo chown -R $USER:$USER /opt/npm
```

Start Nginx Proxy Manager:

```bash
cd /opt/npm
docker compose up -d
```

Verify services are running:

```bash
docker ps
```

### 7.3 Enable Reverse Proxy for LXD WebUI

Access Nginx Proxy Manager dashboard at `http://<server-ip>:81`

Default credentials:
- **Email:** `admin@example.com`
- **Password:** `changeme`

Log in and configure proxy:

1. Navigate to **Proxy Hosts** → **Add Proxy Host**
2. Configure the following:
   - **Domain Names:** `lxd.yourdomain.com` (or your chosen domain)
   - **Scheme:** `https`
   - **Forward Hostname/IP:** `localhost`
   - **Forward Port:** `8443`
   - **Cache Assets:** `On`
   - **Block Common Exploits:** `On`
   - **Websockets Support:** `On`

3. Navigate to **SSL** tab:
   - **SSL Certificate:** Request a new SSL Certificate
   - **Use Let's Encrypt:** `On`
   - **Email Address for Let's Encrypt:** Your email address
   - **Force SSL:** `On`
   - **HTTP/2 Support:** `On`

4. Save and enable the proxy host

Access LXD WebUI through reverse proxy:

```bash
https://lxd.yourdomain.com
```

Verify proxy configuration:

```bash
curl -I https://lxd.yourdomain.com
```

### 7.4 Docker Management

View logs:

```bash
cd /opt/npm
docker compose logs -f
```

Stop services:

```bash
cd /opt/npm
docker compose down
```

Restart services:

```bash
cd /opt/npm
docker compose restart
```

### 7.5 Advanced Configuration: Non-Standard Port for LXD WebUI

For advanced setups where LXD WebUI needs to be exposed on a non-standard port (e.g., 8843), configure custom nginx settings through the Nginx Proxy Manager web UI.

**Step 1: Update docker-compose.yml to expose the custom port**

Edit `/opt/npm/docker-compose.yml` and add the custom port mapping to the ports section:

```yaml
ports:
  - '80:80'
  - '443:443'
  - '81:81'
  - '8843:8843'
```

Restart NPM to apply changes:

```bash
cd /opt/npm
docker compose restart
```

**Step 2: Configure proxy host in NPM web UI**

1. Log in to NPM dashboard at `http://<server-ip>:81`
2. Navigate to **Proxy Hosts** and select your LXD proxy host
3. Under the **Details** tab:
   - **Scheme:** `https`
   - **Forward Hostname/IP:** `<lxd-server-ip>` (e.g., `10.1.41.101`)
   - **Forward Port:** `8443`
   - **Cache Assets:** `On`
   - **Block Common Exploits:** `On`
   - **Websockets Support:** `On`

4. Navigate to the **Advanced** tab and paste the following custom nginx configuration:

```nginx
# enable listen on non-standard port here
listen 8843 ssl http2;
listen [::]:8843 ssl http2;

# LXD WebUI proxy
location / {
        proxy_pass $forward_scheme://$server:$port;
        proxy_ssl_verify off;
        
        # FIX: Force TLS 1.3 only to match LXD's requirements
        proxy_ssl_protocols TLSv1.3;
        
        # --- COOKIE FIXES ---
        proxy_cookie_domain $server lxd.cn.creekside.network;
        proxy_cookie_path / /;
        proxy_pass_header Set-Cookie;

        # Headers for proper proxying
        proxy_set_header Host $host:$port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port $port;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Buffer sizes for OIDC tokens
        proxy_buffer_size          128k;
        proxy_buffers              4 256k;
        proxy_busy_buffers_size    256k;

        proxy_buffering off;
        proxy_read_timeout 3600s;
    }
```

5. Save and enable the proxy host

**Step 3: Access and verify**

Access LXD WebUI on the custom port:

```bash
https://lxd.yourdomain.com:8843
```

Verify custom port is listening:

```bash
sudo netstat -tlnp | grep 8843
```

Configuration Reference:

| Parameter | Description |
|-----------|-------------|
| `listen 8843 ssl http2` | Listen on port 8843 with SSL and HTTP/2 |
| `proxy_ssl_verify off` | Disable SSL certificate verification for backend |
| `proxy_ssl_protocols TLSv1.3` | Force TLS 1.3 for backend communication |
| `proxy_cookie_domain` | Rewrite cookie domain to match reverse proxy |
| `proxy_set_header Host $host:8843` | Pass correct host header with custom port |
| `proxy_set_header X-Forwarded-*` | Pass original request information to backend |
| `proxy_buffer_size 128k` | Increase buffer for larger OIDC tokens |
| `proxy_read_timeout 3600s` | Allow long-lived WebSocket connections |
## 8. Tips and Troubleshooting

### 8.1 Container Profiles

#### 8.1.1 Rocky Linux Profile

Create a Rocky Linux profile with cloud-init configuration:

```bash
lxc profile create rocky
```

Edit the profile and add the following cloud-init contents:

```bash
lxc profile edit rocky
```

Paste the following configuration:

```yaml
config:
  cloud-init.user-data: |
    #cloud-config
    disable_root: false
    package_update: true

    users:
      - name: root
        ssh_authorized_keys:
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDHrPVbtdHf0aJeRu49fm/lLQPxopvvz6NZZqqGB+bcocZUW3Hw8bflhouTsJ+S4Z3v7L/F6mmZhXU1U3PqUXLVTE4eFMfnDjBlpOl0VDQoy9aT60C1Sreo469FB0XQQYS5CyIWW5C5rQQzgh1Ov8EaoXVGgW07GHUQCg/cmOBIgFvJym/Jmye4j2ALe641jnCE98yE4mPur7AWIs7n7W8DlvfEVp4pnreqKtlnfMqoOSTVl2v81gnp4H3lqGyjjK0Uku72GKUkAwZRD8BIxbA75oBEr3f6Klda2N88uwz4+3muLZpQParYQ+BhOTvldMMXnhqM9kHhvFZb21jTWV7p creeksidenetworks@gmail.com

    bootcmd:
      - dhclient -v || true

    packages:
      - epel-release
      - openssh-server
      - firewalld
      - iputils
      - nano
      - git
      - ncurses
      - net-tools
      - nfs-utils
      - curl
      - wget
      - rsync
      - telnet
      - jq
      - lsof
      - bind-utils
      - tcpdump
      - net-tools
      - util-linux
      - tree
      - traceroute
      - mtr
      - cloud-utils-growpart

    runcmd:
      - systemctl enable --now sshd
      - systemctl enable --now firewalld
      - firewall-cmd --permanent --add-service=ssh
      - firewall-cmd --reload
description: Rocky Linux profile with cloud-init configuration
devices: {}
name: rocky
```

Add the `bootcmd` directive to your Rocky Linux profile's cloud-init configuration to automatically enable DHCP on boot:

```yaml
bootcmd:
  - dhclient -v || true
```

To apply this profile when launching a Rocky Linux container:

```bash
lxc launch images:rocky/9/default rocky-vm --profile rocky
```

### 8.2 macvlan Network Configuration in Rocky Linux Containers

**Issue:** Rocky Linux containers (especially 9.x) do not automatically obtain DHCP addresses when a macvlan profile is applied, unlike Ubuntu containers. This is due to changes in Network Manager.

**Reference:** [Rocky Linux LXD Server - Profiles Documentation](https://docs.rockylinux.org/10/books/lxd_server/06-profiles/)

**Solution - DHCP Fix:**

With the cloud-init bootcmd section, DHCP will be enabled on boot.

**Solution - Static IP Fix:**

For static IP assignment on Rocky Linux containers, use a systemd network service:

1. Access the container shell:

```bash
lxc exec rockylinux-container bash
```

2. Create a systemd service unit at `/etc/systemd/system/eth0-static.service`:

```bash
vi /etc/systemd/system/eth0-static.service
```

3. Add the following content (adjust IP, subnet, and gateway as needed):

```ini
[Unit]
Description=Static IP for eth0
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set eth0 up
ExecStart=/sbin/ip addr add <your-static-ip>/<subnet-mask> dev eth0
ExecStart=/sbin/ip route add default via <your-gateway-ip>
ExecStart=/bin/sh -c 'echo "nameserver <your-dns-server>" > /etc/resolv.conf'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

4. Enable the service:

```bash
systemctl enable eth0-static.service
```

5. Exit and restart the container:

```bash
lxc restart rockylinux-container
```

**Note:** Ubuntu containers with macvlan profiles work seamlessly and automatically obtain DHCP addresses. This workaround is specific to Rocky Linux and other RHEL-based distributions due to NetworkManager implementation differences.

### 8.3 Useful LXD Commands

View all container details:

```bash
lxc list
```

View container configuration:

```bash
lxc config show <container-name>
```

View network configuration:

```bash
lxc network show lxdbr0
```

List all profiles:

```bash
lxc profile list
```

View specific profile:

```bash
lxc profile show <profile-name>
```

Rebuild image cache:

```bash
lxc image list
```
