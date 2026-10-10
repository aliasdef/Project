# VPS Hardening — Linux Server Security

## Overview

This project documents the basic hardening of a personal VPS running Linux and Amnezia VPN.

The goal is to reduce unnecessary exposure, secure SSH access, configure a firewall, add basic intrusion prevention, and maintain the system through security updates.

This is a learning project, not a complete production security baseline.

## Security goals

- SSH public-key authentication
- Disable direct root SSH access
- Disable SSH password authentication
- Create a separate administrative user
- Restrict incoming network connections
- Configure Fail2ban for SSH
- Apply security updates automatically
- Verify the effective system configuration

---

# 1. SSH key authentication

## Generate a key on the client

Generate the key on the local computer, not on the server:

```bash
ssh-keygen -t ed25519 -a 100 -C "vps-admin"
```

Use a strong passphrase.

The default key files are:

- `~/.ssh/id_ed25519` — private key
- `~/.ssh/id_ed25519.pub` — public key

The private key must never be uploaded to the VPS or committed to a repository.

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

## Create the administrative user

On a Debian/Ubuntu server:

```bash
sudo adduser USERNAME
sudo usermod -aG sudo USERNAME
```

Choose a strong account password. It can be retained for local administrative operations and sudo authentication even after SSH password authentication is disabled.

## Install the public key

Create the SSH directory:

```bash
sudo install -d -m 700 -o USERNAME -g USERNAME \
  /home/USERNAME/.ssh
```

Add the public key:

```bash
sudo nano /home/USERNAME/.ssh/authorized_keys
```

Paste the complete public key on a single line.

Set ownership and permissions:

```bash
sudo chown USERNAME:USERNAME \
  /home/USERNAME/.ssh/authorized_keys

sudo chmod 600 \
  /home/USERNAME/.ssh/authorized_keys
```

Verify the directory and file:

```bash
sudo ls -ld /home/USERNAME/.ssh
sudo ls -l /home/USERNAME/.ssh/authorized_keys
```

Alternatively, from the client, use `ssh-copy-id` if password-based SSH access is still enabled:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub USERNAME@SERVER_IP
```

---

# 2. Verify key-based access

Before disabling password authentication, open a new terminal on the client:

```bash
ssh -i ~/.ssh/id_ed25519 USERNAME@SERVER_IP
```

Verify administrative access:

```bash
sudo -v
sudo whoami
```

Expected output:

```text
root
```

Do not continue until the new SSH connection and sudo access both work.

Keep the original administrative session open throughout the hardening process.

---

# 3. SSH hardening

## Configure SSH

On Debian/Ubuntu, SSH configuration may be loaded from multiple files:

```text
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/*.conf
```

Ubuntu's SSH configuration commonly includes `sshd_config.d` near the beginning of the main configuration file. OpenSSH generally uses the first obtained value for a directive, so an included file may override a setting placed later in the main file.

Inspect the active configuration:

```bash
sudo grep -RniE '^[[:space:]]*(Include|Match|Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|KbdInteractiveAuthentication|UsePAM)' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Edit the appropriate configuration file. On Ubuntu, a dedicated file such as `/etc/ssh/sshd_config.d/00-local-hardening.conf` can be used, provided no earlier file defines conflicting values.

Example:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Keep PAM at its system default unless you have a specific reason to change it.

Do not change the SSH port unless you have confirmed the new port is configured and reachable.

## Troubleshooting: cloud-init overrides

During this project, password authentication remained enabled despite setting `PasswordAuthentication no` in the main configuration.

The cause was:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf:
PasswordAuthentication yes
```

To resolve this, inspect the effective settings and the included files. Modify the configuration source responsible for the override rather than assuming the main configuration is authoritative.

On cloud-init-managed systems, consider whether cloud-init may regenerate its SSH configuration during provisioning or reboot.

## Validate and reload SSH

Check syntax:

```bash
sudo sshd -t
```

No output means the configuration passed the syntax check.

Check effective settings:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|usepam) '
```

Expected key settings:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
kbdinteractiveauthentication no
```

If the configuration is valid, reload SSH:

```bash
sudo systemctl reload ssh
```

If the service uses another name, identify it with:

```bash
systemctl list-units --type=service | grep -i ssh
```

Open a second terminal and test key-based login again. Also verify that password authentication is rejected.

Do not close the working session until the new connection succeeds.

## Optional: change the SSH port

A non-standard SSH port may reduce automated scanning noise, but it is not a replacement for key authentication and firewall rules.

If changing the port:

1. Configure the new port in SSH.
2. Allow the new port in the VPS provider's network firewall, if applicable.
3. Allow the new port in UFW.
4. Validate the SSH configuration.
5. Reload SSH.
6. Test a new connection using the new port.
7. Remove the old firewall rule only after successful verification.

Example client command:

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 USERNAME@SERVER_IP
```

---

# 4. UFW firewall

## Identify required ports first

Before enabling the firewall, identify the actual ports used by:

- SSH
- Amnezia VPN
- Any intentionally exposed services

Check listening sockets:

```bash
sudo ss -tulnp
```

Check the Amnezia configuration and your provider's network firewall. Do not assume every Amnezia installation uses the same ports or protocols.

## Install UFW

```bash
sudo apt update
sudo apt install ufw
```

## Configure default policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow SSH on its actual port. For the default port:

```bash
sudo ufw limit 22/tcp
```

If SSH has been deliberately moved to port 2222, use this instead:

```bash
sudo ufw limit 2222/tcp
```

Allow each required Amnezia port using the correct protocol and port number.

For example, only if the actual service configuration confirms these ports are required:

```bash
sudo ufw allow 51820/udp
```

Do not copy example VPN ports blindly. Amnezia protocols and deployments differ.

If the server hosts other public services, allow only those required services.

## Enable the firewall

Before enabling UFW, confirm that the SSH allow rule matches the actual SSH port:

```bash
sudo ufw show added
sudo ufw status verbose
```

Then enable the firewall:

```bash
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
sudo ss -tulnp
```

Test SSH and VPN connectivity after applying the rules.

Remember that a host firewall and the VPS provider's network firewall are separate layers.

---

# 5. Fail2ban

## Purpose

Fail2ban monitors logs for repeated failed authentication attempts and applies temporary bans through configured firewall actions.

It is an additional defensive layer, not a replacement for SSH key authentication, firewall rules, or software updates.

## Install

```bash
sudo apt update
sudo apt install fail2ban
```

## Configure the SSH jail

Create a local configuration file:

```bash
sudo nano /etc/fail2ban/jail.local
```

Example configuration for a system using systemd journals:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = ssh
maxretry = 3
findtime = 10m
bantime = 1h
```

If SSH uses a custom port, replace `port = ssh` with the actual port number.

This example uses Fail2ban's default ban action for the installed distribution. Do not force `banaction = ufw` without confirming that the corresponding action is installed and compatible with the firewall setup.

Keep distribution-provided `.conf` files unchanged when possible; use `.local` files for custom settings.

## Enable and verify

```bash
sudo systemctl enable --now fail2ban
```

Check the service:

```bash
sudo systemctl status fail2ban
```

Check the jail list:

```bash
sudo fail2ban-client status
```

Inspect the SSH jail:

```bash
sudo fail2ban-client status sshd
```

Validate the loaded configuration:

```bash
sudo fail2ban-client -t
```

Review logs:

```bash
sudo journalctl -u fail2ban
```

## Test the firewall action

Confirm that the SSH jail is active and that a ban appears in the expected firewall rules when a controlled test triggers one.

Do not repeatedly test from your only administrative IP without an alternate recovery path. A ban can interrupt your own access.

If an IP needs to be unbanned:

```bash
sudo fail2ban-client set sshd unbanip IP_ADDRESS
```

Fail2ban is useful as an additional layer even when password authentication is disabled, but its practical benefit depends on the exposed services and the configured filters.

---

# 6. Root account protection

Direct root SSH login is disabled with:

```text
PermitRootLogin no
```

SSH password authentication is disabled with:

```text
PasswordAuthentication no
```

These settings prevent direct root SSH access and password-based SSH login when the effective configuration confirms they are active.

## Optional: lock the root password

On systems where the root account is not needed for direct password login, you may lock its password:

```bash
sudo passwd -l root
```

This is an optional additional measure, not a substitute for SSH configuration.

Before locking the account, confirm that your distribution's administrative access and recovery procedures are understood. Keep the separate sudo user working.

---

# 7. Automatic security updates

For Ubuntu, use the supported `unattended-upgrades` mechanism rather than creating a custom script that automatically upgrades every package without additional checks.

## Install

```bash
sudo apt update
sudo apt install unattended-upgrades
```

Enable automatic package-list updates and unattended upgrades:

```bash
sudo dpkg-reconfigure unattended-upgrades
```

Inspect the configuration:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

The usual enabled settings are:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Review the allowed package origins:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

For a personal VPN server, automatic security updates are a sensible baseline. Be aware that packages from third-party repositories may not be included unless their origins are configured.

## Reboot policy

Avoid enabling automatic reboots without a deliberate maintenance and recovery plan.

A reboot or service restart may interrupt VPN and SSH connections.

Check whether a reboot is required:

```bash
test -f /var/run/reboot-required && cat /var/run/reboot-required
```

## Verify update activity

Check the service and timers:

```bash
systemctl status unattended-upgrades
systemctl list-timers 'apt-daily*'
```

Review logs:

```bash
sudo less /var/log/unattended-upgrades/unattended-upgrades.log
```

On a server running Amnezia VPN, periodically verify that the VPN remains functional after updates.

For Debian installations, verify that `unattended-upgrades` and its security origins are configured appropriately for the installed release.

---

# 8. Verification and troubleshooting

The most important principle is to verify the effective system state, not merely the configuration files.

## Listening ports

```bash
sudo ss -tulnp
```

Identify unexpected exposed services and confirm that the intended services are listening on the expected interfaces and ports.

## SSH configuration

```bash
sudo sshd -t
```

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|usepam) '
```

## SSH service

```bash
sudo systemctl status ssh
```

## SSH logs

```bash
sudo journalctl -u ssh --since "1 hour ago"
```

On systems where the unit is named `sshd`, use that service name instead.

## Firewall

```bash
sudo ufw status verbose
sudo ufw show added
```

## Fail2ban

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo journalctl -u fail2ban --since "1 hour ago"
```

## Automatic updates

```bash
systemctl list-timers 'apt-daily*'
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log
```

## Final access test

From a separate client terminal:

```bash
ssh -i ~/.ssh/id_ed25519 USERNAME@SERVER_IP
```

Verify that:

- Key-based SSH login succeeds.
- Password-based SSH login is rejected.
- Direct root SSH login is rejected.
- `sudo` works for the administrative user.
- The firewall allows required VPN and SSH traffic.
- Fail2ban is active and the SSH jail is loaded.
- The VPN still works after configuration changes.

---

# 9. What I learned

This project improved my understanding of:

- Linux server administration
- SSH public-key authentication
- SSH hardening and configuration precedence
- Linux users, groups, and file permissions
- UFW firewall configuration
- Fail2ban jails, filters, and ban actions
- systemd services and logs
- Automatic security updates
- Network service exposure
- Troubleshooting authentication failures
- Verifying security controls in the effective system state

One of the most useful lessons was that changing a configuration file does not guarantee the desired behavior.

During the project, password authentication remained enabled because an included cloud-init configuration set it to `yes`. Investigating the effective configuration helped identify and correct the issue.

I also learned the importance of keeping an active administrative session open while testing SSH changes, and of checking firewall and Fail2ban behavior instead of assuming that the tools are working as intended.

---

# 10. Final result

The intended baseline is:

```text
SSH public-key authentication
        +
No direct root SSH login
        +
No SSH password authentication
        +
Separate administrative user
        +
UFW firewall
        +
Fail2ban SSH jail
        +
Automatic security updates
        +
System and authentication logs
        +
Post-change verification
```

This project is intended for learning and basic VPS hardening. It is not a complete production security configuration.

Sensitive information such as private keys, passwords, real IP addresses, VPN credentials, and other secrets must never be included in the repository.

Before publishing, review all configuration examples and command output to ensure that no actual credentials or infrastructure details are exposed.
