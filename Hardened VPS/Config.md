## Setup
1. SSH key

---

Generate an Ed25519 key on the client:

ssh-keygen -t ed25519 -C "example"

Display the public key:

cat ~/.ssh/id_ed25519.pub

Create the SSH directory on the server:

mkdir -p /home/scaramouche/.ssh

Add the public key:

nano /home/scaramouche/.ssh/authorized_keys

Set correct ownership and permissions:

chown -R scaramouche:scaramouche /home/scaramouche/.ssh
chmod 700 /home/scaramouche/.ssh
chmod 600 /home/scaramouche/.ssh/authorized_keys
2. Create an administrative user

Create the user with Bash:

useradd -m -s /bin/bash scaramouche

Set a password:

passwd scaramouche

Add the user to the sudo group:

usermod -aG sudo scaramouche
3. SSH hardening

Edit the SSH configuration:

nano /etc/ssh/sshd_config

Example configuration:

Port 2222
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
UsePAM no

Check the configuration before restarting SSH:

sshd -t

Restart SSH:

systemctl restart ssh
Test the connection

Open a second terminal and test the new configuration before closing the current SSH session:

ssh -p 2222 scaramouche@SERVER_IP

Expected result:

scaramouche + SSH key       → allowed
scaramouche + password      → denied
root + SSH key              → denied
root + password             → denied
4. UFW

Install UFW:

apt install ufw

Set default policies:

ufw default deny incoming
ufw default allow outgoing

Allow the required ports:

ufw limit 2222/tcp
ufw allow 2000/tcp
ufw allow 51820/udp
ufw allow 51821/udp

Enable the firewall:

ufw enable

Check the configuration:

ufw status verbose

The SSH rule is limited with ufw limit to reduce repeated connection attempts.

5. Fail2ban

Install Fail2ban:

apt install fail2ban

Create the local configuration:

nano /etc/fail2ban/jail.local

Example configuration:

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

Enable and start Fail2ban:

systemctl enable --now fail2ban

Check the service:

fail2ban-client status

Check the SSH jail:

fail2ban-client status sshd

If an IP needs to be unbanned:

fail2ban-client set sshd unbanip <IP>
6. Disable root password

Lock the root password:

passwd -l root

Combined with:

PermitRootLogin no
PasswordAuthentication no

this prevents direct SSH access to the root account.

7. Automatic updates

Create the update script:

nano /usr/local/bin/update-vps.sh

Example:

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

Make it executable:

chmod +x /usr/local/bin/update-vps.sh

Test it manually:

/usr/local/bin/update-vps.sh

Check the logs:

ls -la /var/log/vps-update/
8. Cron

Edit the root crontab:

crontab -e

Example: run the update every Sunday at midnight:

0 0 * * 0 /usr/local/bin/update-vps.sh
Verification

After configuring the VPS, I check the actual state of the system instead of assuming the configuration works.

Listening ports
ss -tulnp
Firewall
ufw status verbose
Fail2ban
fail2ban-client status
fail2ban-client status sshd
SSH configuration
sshd -t

Check the effective SSH configuration:

sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|pubkeyauthentication'
SSH service
systemctl status ssh
SSH logs
journalctl -u ssh
What I learned

This project helped me get more comfortable with:

Linux server administration
SSH hardening
SSH key authentication
Linux permissions
UFW
Fail2ban
Bash scripting
cron
systemd
troubleshooting SSH authentication
basic server hardening
verifying security controls

One of the most useful lessons was that security configuration is not just about changing settings.

You also need to test what you changed and make sure you haven't locked yourself out of the server.

During the project I ran into several issues myself, including SSH key permissions and getting blocked by Fail2ban while testing. Fixing these problems was part of the learning process.

Final Result

The VPS now uses:

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

This project is primarily for learning and does not represent a complete production security configuration.

Sensitive information such as private keys, passwords, real IP addresses and credentials is not included in this repository.
