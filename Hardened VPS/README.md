# Hardened VPS

A small hands-on project where I set up a Linux VPS and went through the basic steps of securing it.
The main goal was simple: make the server harder to attack and understand why each security setting is needed.

What I did:

## SSH

    Generated an Ed25519 SSH key

    Set up key-based authentication

    Disabled password authentication

    Changed the default SSH port

    Disabled SSH login for root

    Created a separate administrative user

    Configured authorized_keys

    Checked SSH configuration before restarting the service

## Firewall

Configured UFW with a default-deny incoming policy:

    Deny incoming connections by default

    Allow outgoing connections

    Allow only required ports

    Added connection limiting for SSH

## Fail2ban

Set up Fail2ban to protect SSH from repeated failed login attempts.

Configured:

    Ban time

    Detection window

    Maximum number of attempts

    UFW integration

    SSH jail

During testing, I also managed to ban my own IP while troubleshooting. :)
Automatic Updates

Created a small Bash script for automatic system updates.

## The script:

    Runs package updates

    Logs the execution time

    Stores update results

    Returns an error if the update fails

The script is scheduled with cron.
## User Security
Created a separate user for administration:

name: s******e

The user has:

    SSH key authentication

    sudo access

    /bin/bash as the default shell

Root SSH access is disabled.


## Architecture

                    Internet
                       │
                       ▼
                      UFW
                  Firewall
                       │
                       ▼
                   Fail2ban
              Brute-force protection
                       │
                       ▼
                  SSH :2222
                       │
                 SSH key only
                       │
                       ▼
                  scaramouche
                       │
                      sudo
                       │
                       ▼
                  Linux VPS


Now we move on to the settings themselves: [Config VPS](https://github.com/aliasdef/Project/blob/main/Hardened%20VPS/Config.md)
