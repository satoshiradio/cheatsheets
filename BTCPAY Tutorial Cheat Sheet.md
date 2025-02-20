# Linux Server Hardening Guide for BTCPay Server

## Introduction

This guide will help you secure your Virtual Private Server (VPS) for running a BTCPay Server. Before starting, note that you should replace the following placeholders with your actual values throughout this guide:

- `<VPS_IP>` - Your server's IP address
- `<USERNAME>` - The non-root username you'll create
- `<SSH_KEY_NAME>` - Name for your SSH key (e.g., "id_rsa_DigitalOcean" or "id_ed25519_BTCPAY")
- `<KEY_LABEL>` - A descriptor for your SSH key (e.g., "btcpay-server-key")

## Generic Commands
```bash
# List all public keys in your .ssh directory (directory of current user)
ls -la ~/.ssh/

# Remove a key
rm ~/.ssh/id_rsa_DigitalOcean

# Display content of all public keys
cat ~/.ssh/*.pub

# For a specific key
cat ~/.ssh/id_rsa_DigitalOcean.pub
```

## 0. Generate SSH Key for VPS Provider

```bash
# Create SSH key
ssh-keygen

# Choose key location and name (id_rsa_DigitalOcean)
~/.ssh/<SSH_KEY_NAME>

# Add a strong passphrase when prompted

# List all public keys in your .ssh directory
ls -la ~/.ssh/

# Display content of your public key
cat ~/.ssh/<SSH_KEY_NAME>.pub

# Add this key to your VPS provider's dashboard before creating the server
```

## 1. Initial System Update and Security Patches

```bash
# Login as root initially
ssh root@<VPS_IP>
# Or with specific key:
ssh -i ~/.ssh/<SSH_KEY_NAME> root@<VPS_IP>

# Update package list and upgrade all packages
apt update && apt upgrade -y

# Install unattended-upgrades for automatic security updates
apt install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades

# Configure unattended-upgrades
nano /etc/apt/apt.conf.d/50unattended-upgrades

# Recommended settings:
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}-security";
        // "${distro_id}:${distro_codename}-updates";
};

# Auto-reboot settings:
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";

# Verify configuration
unattended-upgrades --dry-run --debug
```

## 2. Create Non-Root User

```bash
# Create new user with sudo privileges
adduser <USERNAME>

# Add user to sudo group
usermod -aG sudo <USERNAME>

# Verify sudo access
su - <USERNAME>
sudo whoami  # Should return "root"
```

## 3. Configure SSH Key Authentication

```bash
# On your LOCAL machine:
ssh-keygen -t ed25519 -C "<KEY_LABEL>"

# On the SERVER as your non-root user:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Copy your LOCAL public key content
cat ~/.ssh/id_ed25519.pub  # On LOCAL machine

# Add it to authorized_keys on SERVER
echo "paste-your-public-key-here" >> ~/.ssh/authorized_keys
nano ~/.ssh/authorized_keys  # Verify key was added correctly

# Optional: Create SSH config on LOCAL machine
nano ~/.ssh/config

Host btcpay-server
    HostName <VPS_IP>
    User <USERNAME>
    IdentityFile ~/.ssh/id_ed25519

# Disable password authentication and root login
sudo nano /etc/ssh/sshd_config

# Set these values:
PermitRootLogin no
PasswordAuthentication no
UsePAM no
ChallengeResponseAuthentication no

# Check for overriding settings:
sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo cat /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

# Edit if necessary to disable password authentication
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo nano /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

# Restart SSH service
sudo systemctl restart ssh
```

## 4. Configure Firewall (UFW)

```bash
# Install UFW
sudo apt install ufw

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow ssh

# Allow BTCPay Server required ports
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 9735/tcp  # Lightning Network
sudo ufw allow 8333/tcp  # Bitcoin mainnet

# Enable firewall
sudo ufw enable

# Verify status
sudo ufw status verbose
```

## 5. Install and Configure Fail2ban

```bash
# Install fail2ban
sudo apt install fail2ban

# Create local configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# Recommended configuration:
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600

# Enable and start fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

## 6. Monitor BTCPay Server Resources (OOM KILLER)

```bash
# Check Bitcoin node resource usage
sudo docker stats btcpayserver_bitcoind

# Check for out-of-memory errors
sudo journalctl | grep -i "out of memory"

# Verify memory settings
sudo docker inspect btcpayserver_bitcoind | grep -i "memory"

# Check Bitcoin configuration
sudo docker exec btcpayserver_bitcoind cat /data/bitcoin.conf

# Check node status
sudo docker exec btcpayserver_bitcoind bitcoin-cli -rpccookiefile=/data/.cookie -rpcport=43782 getblockchaininfo

# Create swap file if needed (recommended for systems with less than 4GB RAM)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap
free -h
```

## Verification Steps

```bash
# Verify SSH configuration
sudo sshd -T

# Check listening ports
sudo netstat -tulpn

# Review UFW rules
sudo ufw status verbose

# Check fail2ban status
sudo fail2ban-client status

# Monitor SSH logs
sudo journalctl -fu sshd
```

## Regular Maintenance Tasks

1. Weekly system updates:
   ```bash
   sudo apt update && sudo apt upgrade
   ```

2. Monitor authentication logs:
   ```bash
   sudo tail -f /var/log/auth.log
   ```

3. Check fail2ban logs:
   ```bash
   sudo tail -f /var/log/fail2ban.log
   ```

4. Review firewall logs:
   ```bash
   sudo tail -f /var/log/ufw.log
   ```

## Security Best Practices

1. Store SSH keys securely and maintain backups
2. Document any custom port changes
3. Maintain a list of authorized users
4. Regularly review and update firewall rules
5. Monitor system resources and disk space
6. Consider setting up automated log monitoring and alerts
7. Regularly rotate SSH keys and update passwords
8. Keep software versions up to date
9. Review system logs periodically for suspicious activity
