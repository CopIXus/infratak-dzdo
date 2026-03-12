# dzdo and RHEL 9 Support

This document describes how to run infra-TAK on **dzdo-only** Red Hat Enterprise Linux 9 systems where `sudo` is not installed or available.

## What is dzdo?

**dzdo** is Centrify/Delinea's replacement for `sudo`. It provides privilege escalation with role-based access from Active Directory instead of local sudoers files. The syntax is similar: `dzdo command` instead of `sudo command`.

## Prerequisites

- **RHEL 9** (or Rocky Linux 9)
- **dzdo** installed and configured (typically via Centrify DirectControl)
- **Root access** via dzdo (or equivalent privileges)
- **Docker** (for Authentik, TAK Portal, Node-RED, CloudTAK)
- **Internet** connection for initial setup

## Quick Start (RHEL 9)

```bash
git clone --depth 1 https://github.com/CopIXus/infratak-dzdo.git
cd infratak-dzdo
chmod +x start.sh
dzdo ./start.sh
```

Use `dzdo` instead of `sudo` for all privileged commands:

| Command | Ubuntu | RHEL 9 (dzdo) |
|---------|--------|---------------|
| Install | `sudo ./start.sh` | `dzdo ./start.sh` |
| Update | `sudo systemctl restart takwerx-console` | `dzdo systemctl restart takwerx-console` |
| Reset password | `sudo ./reset-console-password.sh` | `dzdo ./reset-console-password.sh` |
| Fix after pull | `sudo ./fix-console-after-pull.sh` | `dzdo ./fix-console-after-pull.sh` |

## Required dzdo Policy

Your dzdo configuration must allow the user to:

1. **Run as root** — Full root equivalent for installation and systemctl operations
2. **Run as postgres** — `dzdo -u postgres psql ...` for PostgreSQL (CoT database)
3. **Run as tak** — `dzdo -u tak ./makeCert.sh ...` for TAK Server certificate generation

The infra-TAK console runs as a systemd service with `User=root`. When it needs to run commands as `postgres` or `tak`, it uses `PRIV_CMD -u <user>`. On RHEL, `PRIV_CMD` is set to `dzdo` via the systemd environment.

## How It Works

- **Installation:** You run `dzdo ./start.sh`. dzdo elevates to root; the script proceeds normally.
- **Runtime:** The systemd unit sets `Environment=PRIV_CMD=dzdo` when the system uses dnf (RHEL/Rocky). The Flask app uses `PRIV_CMD` instead of hardcoded `sudo` for all privilege escalation.
- **Detection:** The app defaults to `dzdo` when `/etc/redhat-release` exists; otherwise it uses `sudo`. You can override with `PRIV_CMD=dzdo` or `PRIV_CMD=sudo` in the environment.

## TAK Server on RHEL 9

- Upload the **.rpm** package from [tak.gov](https://tak.gov) (not the .deb)
- The deploy uses `dnf install -y <package.rpm>` and `firewall-cmd` instead of apt and ufw
- PostgreSQL is installed via dnf; the TAK Server RPM handles its configuration

## Known Differences from Ubuntu

| Aspect | Ubuntu | RHEL 9 |
|--------|--------|--------|
| Package manager | apt, dpkg | dnf, rpm |
| TAK Server package | .deb | .rpm |
| Firewall | ufw | firewalld (firewall-cmd) |
| Unattended upgrades | unattended-upgrade | dnf-automatic (not waited on) |
| GPG verification | debsig-verify | Skipped for RPM (use rpm --checksig if needed) |

## FIPS Mode (RHEL 9)

RHEL 9 often has FIPS mode enabled. Password hashing uses `pbkdf2:sha256` (FIPS-approved) instead of `scrypt` to avoid `ValueError: [digital envelope routines] unsupported`.

## Troubleshooting

**"Permission denied" or "command not found: sudo"**  
Ensure you are using `dzdo` instead of `sudo` for all commands.

**dzdo -u postgres fails**  
Your dzdo policy must allow running as the `postgres` user. Check with your Centrify/Delinea administrator.

**dzdo -u tak fails**  
The `tak` user is created by the TAK Server package. Ensure TAK Server is installed and the `tak` user exists.

**PRIV_CMD not set**  
If the console was installed on Ubuntu and later moved to RHEL, re-run `dzdo ./start.sh` to regenerate the systemd unit with `PRIV_CMD=dzdo`. Or manually add `Environment=PRIV_CMD=dzdo` to `/etc/systemd/system/takwerx-console.service` and run `systemctl daemon-reload` and `systemctl restart takwerx-console`.
