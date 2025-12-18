# Installing NGINX Plus on Ubuntu (Jammy)

This document describes how to install **NGINX Plus** on an Ubuntu server.  
It is based on the official [NGINX Docs](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/#install_debian_ubuntu) and includes troubleshooting for common issues encountered during setup.

---

## Prerequisites

1. **Ubuntu Jammy (22.04)** or newer.
2. A valid **NGINX Plus trial certificate** (`nginx-repo.crt`) and **private key** (`nginx-repo.key`).  
   These files are provided by F5/NGINX after [registering for a free trial](https://www.nginx.com/free-trial-request/).
3. Root or `sudo` privileges.
4. Stable internet connection to access NGINX package repositories.

---

## Installation – Step by Step

### 1. Copy Certificate and Key Files
```bash
# Create SSL directory for NGINX repository certificates
sudo mkdir -p /etc/ssl/nginx

# Move your downloaded certificate and key files
# Replace the paths with your actual file locations
sudo cp /path/to/your-SFA-XXXXXX-F5-trial.crt /etc/ssl/nginx/nginx-repo.crt
sudo cp /path/to/your-SFA-XXXXXX-F5-trial.key /etc/ssl/nginx/nginx-repo.key

# Set appropriate permissions
sudo chmod 644 /etc/ssl/nginx/nginx-repo.crt
sudo chmod 600 /etc/ssl/nginx/nginx-repo.key

# Verify files are in place
ls -l /etc/ssl/nginx/
```

### 2. Install Required Dependencies
```bash
sudo apt update
sudo apt install -y apt-transport-https lsb-release ca-certificates wget gnupg2 ubuntu-keyring software-properties-common
```

### 3. Add NGINX Signing Key
```bash
# Download and add the official NGINX signing key
wget -qO - https://cs.nginx.com/static/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# Verify the key was added
gpg --list-keys --keyring /usr/share/keyrings/nginx-archive-keyring.gpg
```

### 4. Configure NGINX Plus Repository
```bash
# Add the NGINX Plus repository for your Ubuntu version
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://pkgs.nginx.com/plus/ubuntu $(lsb_release -cs) nginx-plus" | sudo tee /etc/apt/sources.list.d/nginx-plus.list
```

### 5. Configure APT HTTPS Certificate Authentication
Create the configuration file `/etc/apt/apt.conf.d/90pkgs-nginx`:

```bash
sudo tee /etc/apt/apt.conf.d/90pkgs-nginx > /dev/null << 'EOF'
Acquire::https::pkgs.nginx.com::Verify-Peer "true";
Acquire::https::pkgs.nginx.com::Verify-Host "true";
Acquire::https::pkgs.nginx.com::SslCert "/etc/ssl/nginx/nginx-repo.crt";
Acquire::https::pkgs.nginx.com::SslKey  "/etc/ssl/nginx/nginx-repo.key";
EOF

# Verify the file was created
cat /etc/apt/apt.conf.d/90pkgs-nginx
```

### 6. Update Package Lists and Install NGINX Plus
```bash
# Update package lists with the new repository
sudo apt update

# Install NGINX Plus
sudo apt install -y nginx-plus

# Alternatively, install specific modules
# sudo apt install -y nginx-plus nginx-plus-module-modsecurity nginx-plus-module-geoip
```

### 7. Verify Installation
```bash
# Check NGINX version (should show "nginx-plus")
nginx -v

# Check certificate validity
openssl x509 -in /etc/ssl/nginx/nginx-repo.crt -noout -dates

# Check NGINX Plus status
sudo systemctl status nginx

# Test configuration
sudo nginx -t
```

### 8. Start and Enable NGINX Plus
```bash
# Start NGINX Plus if not already running
sudo systemctl start nginx

# Enable auto-start on boot
sudo systemctl enable nginx

# Verify it's listening on expected ports
sudo ss -tlnp | grep nginx
```

---

## Common Errors & Solutions

### ❌ Error: Repository Added with Wrong Format
**Problem:**
```bash
echo "deb [signed-by=/etc/ssl/nginx/nginx-repo.crt,sslkey=/etc/ssl/nginx/nginx-repo.key] https://pkgs.nginx.com/plus/ubuntu jammy nginx-plus"
```
**Issue:** `deb` sources do not support `sslkey` and `sslcert` parameters directly in the repository line.

**Solution:** Use the separate `/etc/apt/apt.conf.d/90pkgs-nginx` file as shown in Step 5.

---

### ❌ Error: `90pkgs-nginx` Placed in Wrong Directory
**Problem:**
```bash
sudo mv /etc/apt/apt.conf.d/90pkgs-nginx /etc/apt/sources.list.d/
```
**Issue:** The APT configuration file belongs in `/etc/apt/apt.conf.d/`, not in the sources list directory.

**Solution:**
```bash
# Move it back if accidentally moved
sudo mv /etc/apt/sources.list.d/90pkgs-nginx /etc/apt/apt.conf.d/
```

---

### ❌ Error: Multiple/Duplicate Configuration Files
**Problem:** Files like `90pkgs-nginx.1`, `90pkgs-nginx.2` appear after multiple download attempts.

**Solution:**
```bash
# Remove all duplicate files, keeping only the main one
sudo rm -f /etc/apt/apt.conf.d/90pkgs-nginx.*
sudo rm -f /etc/apt/sources.list.d/nginx-plus.list.*

# Recreate the correct files as per steps 4-5
```

---

### ❌ Error: GPG Key Verification Failed
**Problem:** `apt update` fails with GPG errors.

**Solution:**
```bash
# Re-add the GPG key
curl -fsSL https://cs.nginx.com/static/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg

# Update again
sudo apt update
```

---

### ❌ Error: Certificate Authentication Failed
**Problem:** `apt update` fails with 403 or certificate errors.

**Solution:**
1. Verify certificate and key files are valid:
   ```bash
   openssl x509 -in /etc/ssl/nginx/nginx-repo.crt -text -noout | head -20
   openssl rsa -in /etc/ssl/nginx/nginx-repo.key -check -noout
   ```

2. Check certificate expiration:
   ```bash
   openssl x509 -in /etc/ssl/nginx/nginx-repo.crt -noout -dates
   ```

3. Ensure correct permissions:
   ```bash
   sudo chown root:root /etc/ssl/nginx/nginx-repo.*
   sudo chmod 644 /etc/ssl/nginx/nginx-repo.crt
   sudo chmod 600 /etc/ssl/nginx/nginx-repo.key
   ```

---

### ❌ Error: Repository Not Found for Release
**Problem:** `apt update` shows "Release file not found" for the NGINX repository.

**Solution:** Check your Ubuntu version and repository URL:
```bash
# Check your Ubuntu release
lsb_release -cs

# Verify the repository URL matches your release
cat /etc/apt/sources.list.d/nginx-plus.list
```

For Ubuntu 22.04 (Jammy), the file should contain:
```
deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://pkgs.nginx.com/plus/ubuntu jammy nginx-plus
```

---

### ❌ Error: NGINX Starts but Plus Features Unavailable
**Problem:** NGINX runs but shows as open source version.

**Solution:**
1. Verify NGINX Plus installation:
   ```bash
   nginx -v  # Should show "nginx-plus"
   ```

2. Check if the license file exists:
   ```bash
   sudo ls -la /etc/nginx/nginx-plus-license.key
   ```

3. If missing, apply your trial license:
   ```bash
   # Copy your trial license key
   sudo cp /path/to/your-trial-license.key /etc/nginx/nginx-plus-license.key
   sudo chmod 644 /etc/nginx/nginx-plus-license.key
   sudo nginx -s reload
   ```

---

## Post-Installation Steps

### 1. Configure NGINX Plus
```bash
# Backup original configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Enable the status module for monitoring (optional)
sudo tee -a /etc/nginx/conf.d/status.conf > /dev/null << 'EOF'
server {
    listen 8080;
    
    location /api/ {
        api write=on;
    }
    
    location = /dashboard.html {
        root /usr/share/nginx/html;
    }
}
EOF

# Test and reload configuration
sudo nginx -t
sudo nginx -s reload
```

### 2. Verify NGINX Plus Features
```bash
# Check if NGINX Plus API is accessible
curl http://localhost:8080/api/ 2>/dev/null | head -5

# Check active connections
curl http://localhost:8080/api/1/connections 2>/dev/null
```

### 3. Set Up Firewall Rules (if needed)
```bash
# Allow HTTP, HTTPS, and status port
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp comment 'NGINX Plus status'
sudo ufw reload
```

---

## Troubleshooting Commands

```bash
# Check installation logs
sudo journalctl -u nginx --no-pager -n 50

# Verify repository configuration
grep -r "nginx.com" /etc/apt/

# Test package download manually
apt download nginx-plus --print-uris

# Check for conflicting NGINX installations
dpkg -l | grep nginx

# Remove any conflicting open source NGINX
sudo apt remove nginx nginx-common nginx-core --purge
sudo apt autoremove
```

---

## Summary

A successful **NGINX Plus** installation on Ubuntu requires:

1. **Certificate Setup**: Place trial certificate and key in `/etc/ssl/nginx/`
2. **Repository Configuration**: Add NGINX Plus repository to `sources.list.d/`
3. **APT Authentication**: Configure certificate access via `90pkgs-nginx`
4. **Installation**: Use `apt install nginx-plus`
5. **Verification**: Check version, status, and features
6. **Configuration**: Enable desired NGINX Plus modules and features

Once installed, NGINX Plus can be updated regularly via:
```bash
sudo apt update
sudo apt upgrade nginx-plus
```

---

## Additional Resources

- [Official NGINX Plus Documentation](https://docs.nginx.com/nginx-plus/)
- [NGINX Plus Admin Guide](https://docs.nginx.com/nginx/admin-guide/)
- [F5 Support Portal](https://support.f5.com)
- [NGINX Community Forum](https://forum.nginx.org)

---

**Note**: This installation guide is for evaluation purposes using a trial license. For production deployments, ensure you have the appropriate commercial licenses and follow your organization's security and compliance requirements.

