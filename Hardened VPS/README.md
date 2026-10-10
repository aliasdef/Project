# Hardened VPS

A hands-on Linux server hardening project focused on securing SSH access, configuring a firewall, and implementing basic intrusion prevention and system maintenance.

The goal was to make my VPS more secure while understanding how each security measure works and learning how to verify the actual system configuration.

## What I implemented

### SSH Hardening

- Generated an Ed25519 SSH key pair.
- Configured public-key authentication.
- Disabled SSH password authentication.
- Changed the default SSH port to `2222`.
- Disabled direct SSH login for `root`.
- Created a separate administrative user.
- Configured `authorized_keys` and SSH file permissions.
- Validated the SSH configuration before applying changes.
- Verified the effective SSH settings and tested a new connection before closing the existing session.

During troubleshooting, I discovered that a cloud-init configuration file was overriding the password authentication setting in the main SSH configuration. I investigated the included configuration files and corrected the conflicting setting.

### Firewall — UFW

Configured UFW to restrict incoming network traffic.

- Set the default incoming policy to `deny`.
- Allowed outgoing connections.
- Allowed only the ports required for SSH and the VPN.
- Applied connection limiting to SSH.
- Verified the active firewall rules and listening ports.

### Fail2ban

Configured Fail2ban to detect repeated failed SSH authentication attempts and temporarily ban offending IP addresses.

The configuration included:

- An SSH jail.
- A maximum number of failed attempts.
- A detection time window.
- A ban duration.
- Firewall integration.

During testing, I accidentally banned my own IP address while troubleshooting. This helped me understand the importance of maintaining an alternative recovery path and verifying firewall actions.

### User and Privilege Management

Created a separate administrative user: `scaramouche`.

The account uses:

- SSH public-key authentication.
- `sudo` privileges.
- `/bin/bash` as the default shell.

Direct root SSH login is disabled, and SSH password authentication is disabled.

### Automatic Security Updates

Configured automated package maintenance for security updates.

The setup uses the system's package management and automatic update mechanisms, with logging to help verify update activity and troubleshoot failures.

Update logs and service status are checked to confirm that the configuration is working as intended.

### Verification and Troubleshooting

Rather than relying solely on configuration files, I checked the effective system state.

Tools and commands used included:

- `sshd -t` — validate SSH configuration syntax.
- `sshd -T` — inspect effective SSH settings.
- `ss` — inspect listening sockets.
- `ufw status` — verify firewall rules.
- `fail2ban-client` — inspect Fail2ban jails and status.
- `systemctl` — check service status.
- `journalctl` — inspect service logs.

## Architecture

```text
                  Internet
                     |
                     v
              +--------------+
              |     UFW      |
              |   Firewall   |
              +--------------+
                     |
                     v
              +--------------+
              |   Fail2ban   |
              | SSH monitoring|
              +--------------+
                     |
                     v
              +--------------+
              |  SSH :2222   |
              | Key auth only|
              +--------------+
                     |
                     v
              +--------------+
              | scaramouche  |
              | Admin user   |
              +--------------+
                     |
                     v
                   sudo
                     |
                     v
                Linux VPS
                     |
                     v
                Amnezia VPN
```

*Note: This is a conceptual overview, not a literal network packet-processing sequence. UFW filters network traffic, while Fail2ban monitors authentication activity and applies configured bans through firewall actions. Amnezia VPN uses its own configured ports and protocols.*

## What I learned

This project helped me gain practical experience with:

- Linux server administration.
- SSH authentication and hardening.
- SSH configuration precedence and troubleshooting.
- Linux users, groups, permissions, and privilege escalation through `sudo`.
- Firewall configuration and network exposure.
- Fail2ban jails and ban actions.
- Systemd services and journal logs.
- Automated security updates.
- Verifying security controls instead of assuming they work.

One of the most useful lessons was that changing a configuration file does not necessarily change the effective behavior of a service. I had to investigate an SSH configuration override, correct it, and verify the result.

I also learned to keep an existing administrative session open while testing SSH changes to avoid accidentally locking myself out of the server.

## Configuration Guide

The detailed setup, commands, and verification steps are documented here:

[**Hardened VPS — Configuration Guide**](https://github.com/aliasdef/Project/blob/main/Hardened%20VPS/Config.md)

## Scope and Limitations

This is a personal learning project demonstrating basic VPS hardening. It is not a complete production security baseline.

The configuration should be reviewed against the actual operating system, SSH settings, firewall rules, and Amnezia VPN deployment before being reused on another server.

Sensitive information, including private keys, passwords, VPN credentials, and real infrastructure secrets, must not be committed to the repository.
