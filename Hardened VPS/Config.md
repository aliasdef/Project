# Setup

## 1. SSH key

Generate an Ed25519 key on the client:

```bash
ssh-keygen -t ed25519 -C "example"
```

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Create the SSH directory on the server:

```bash
mkdir -p /home/scaramouche/.ssh
```

Add the public key:

```bash
nano /home/scaramouche/.ssh/authorized_keys
```

Set correct ownership and permissions:

```bash
chown -R scaramouche:scaramouche /home/scaramouche/.ssh
chmod 700 /home/scaramouche/.ssh
chmod 600 /home/scaramouche/.ssh/authorized_keys
```

---

## 2. Create an administrative user

Create the user with Bash:

```bash
useradd -m -s /bin/bash scaramouche
```

Set a password:

```bash
passwd scaramouche
```

Add the user to the sudo group:

```bash
usermod -aG sudo scaramouche
```

---

## 3. SSH hardening

Edit the SSH configuration:

```bash
nano /etc/ssh/sshd_config
```

Example configuration:

```text
Port 2222
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
UsePAM no
```

Check the configuration before restarting SSH:

```bash
sshd -t
```

Restart SSH:

```bash
systemctl restart ssh
```

### Test the connection

Open a second terminal and test the new configuration before closing the current SSH session:

```bash
ssh -p 2222 scaramouche@SERVER_IP
```

Expected result:

```text
scaramouche + SSH key       → allowed
scaramouche + password      → denied
root + SSH key              → denied
root + password             → denied
```

---

# 4. UFW

Install UFW:

```bash
apt install ufw
```

Set default policies:

```bash
ufw default deny incoming
ufw default allow outgoing
```

Allow the required ports:

```bash
ufw limit 2222/tcp
ufw allow 2000/tcp
ufw allow 51820/udp
ufw allow 51821/udp
```

Enable the firewall:

```bash
ufw enable
```

Check the configuration:

```bash
ufw status verbose
```

The SSH rule is limited with `ufw limit` to reduce repeated connection attempts.

---

# 5. Fail2ban

Install Fail2ban:

```bash
apt install fail2ban
```

Create the local configuration:

```bash
nano /etc/fail2ban/jail.local
```

Example configuration:

```ini
[DEFAULT]
banaction = ufw
bantime = 3600
findtime = 600
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = 2222
filter = sshd
maxretry = 3
```

Enable and start Fail2ban:

```bash
systemctl enable --now fail2ban
```

Check the service:

```bash
fail2ban-client status
```

Check the SSH jail:

```bash
fail2ban-client status sshd
```

If an IP needs to be unbanned:

```bash
fail2ban-client set sshd unbanip <IP>
```

---

# 6. Disable root password

Lock the root password:

```bash
passwd -l root
```

Combined with:

```text
PermitRootLogin no
PasswordAuthentication no
```

this prevents direct SSH access to the root account.

---

# 7. Automatic updates

Create the update script:

```bash
nano /usr/local/bin/update-vps.sh
```

Example:

```bash
#!/bin/bash

LOG_DIR="/var/log/vps-update"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
LOG_FILE="$LOG_DIR/$TIMESTAMP.log"

mkdir -p "$LOG_DIR"

echo "=== Update started: $TIMESTAMP ===" >> "$LOG_FILE"

if apt update >> "$LOG_FILE" 2>&1 && apt upgrade -y >> "$LOG_FILE" 2>&1; then
    echo "Update completed successfully" >> "$LOG_FILE"
else
    echo "Update FAILED" >> "$LOG_FILE"
    exit 1
fi
```

Make it executable:

```bash
chmod +x /usr/local/bin/update-vps.sh
```

Test it manually:

```bash
/usr/local/bin/update-vps.sh
```

Check the logs:

```bash
ls -la /var/log/vps-update/
```

---

# 8. Cron

Edit the root crontab:

```bash
crontab -e
```

Example: run the update every Sunday at midnight:

```cron
0 0 * * 0 /usr/local/bin/update-vps.sh
```

---

# Verification

After configuring the VPS, I check the actual state of the system instead of assuming the configuration works.

### Listening ports

```bash
ss -tulnp
```

### Firewall

```bash
ufw status verbose
```

### Fail2ban

```bash
fail2ban-client status
fail2ban-client status sshd
```

### SSH configuration

```bash
sshd -t
```

Check the effective SSH configuration:

```bash
sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|pubkeyauthentication'
```

### SSH service

```bash
systemctl status ssh
```

### SSH logs

```bash
journalctl -u ssh
```

---

# What I learned

This project helped me get more comfortable with:

* Linux server administration
* SSH hardening
* SSH key authentication
* Linux permissions
* UFW
* Fail2ban
* Bash scripting
* cron
* systemd
* troubleshooting SSH authentication
* basic server hardening
* verifying security controls

One of the most useful lessons was that security configuration is not just about changing settings.

**You also need to test what you changed and make sure you haven't locked yourself out of the server.**

During the project I ran into several issues myself, including SSH key permissions and getting blocked by Fail2ban while testing. Fixing these problems was part of the learning process.

---

# Final Result

The VPS now uses:

```text
SSH keys instead of passwords
        +
No root SSH login
        +
UFW firewall
        +
Fail2ban
        +
Separate sudo user
        +
Automatic system updates
        +
Basic logging and verification
```

This project is primarily for learning and does not represent a complete production security configuration.

Sensitive information such as private keys, passwords, real IP addresses and credentials is not included in this repository.
