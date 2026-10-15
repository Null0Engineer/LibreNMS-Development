# README-BUILD — LibreNMS Development VM

> **Build log / reproducible rebuild guide** — updated 2026-10-15.
>
> **Current build status:** Ubuntu 26.04.1 LTS **OS verified on rebuilt VM** (2026-10-15). Git 2.53.0 is installed. PHP is *not yet installed*. Network, SSH, packages, MariaDB, the LibreNMS Git checkout and application setup remain to be verified. The earlier Ubuntu 24.04 VM was discarded; its results are historical only.

## Purpose and guardrails

Dedicated, non-production LibreNMS development instance to work on:
1. NetBox IPAM integration (first project).
2. NETCONF support (subsequent project).
3. Other upstream-compatible LibreNMS features, fixes, tests and PRs.

**Mandatory upstream references (check again before every implementation):**

- [LibreNMS Developing documentation](https://docs.librenms.org/Developing/) — all applicable sections are mandatory.
- [Getting Started](https://docs.librenms.org/Developing/Getting-Started/)
- [Using Git](https://docs.librenms.org/Developing/Using-Git/)
- [Code Guidelines](https://docs.librenms.org/Developing/Code-Guidelines/)
- [Validating Code](https://docs.librenms.org/Developing/Validating-Code/)
- [Installing LibreNMS](https://docs.librenms.org/Installation/Install-LibreNMS/)

Do not make feature changes on production systems. Prefer feature branches, run tests, avoid committing credentials or local `.env` files, and follow upstream style and architecture. SNMP configuration for this environment is **SNMPv3 only**, with SHA authentication and AES privacy when needed.

## Progress checklist — replacement VM

- [x] 0. Standard Ubuntu Server 26.04.1 installation completed; no Cloud-Init (OS release verified).
- [ ] 1. OS version, networking and SSH verified.
- [x] 2. OS updates and LibreNMS package prerequisites installed and verified (2026-10-15; versions and service states confirmed; reboot recommended due to refreshed packages).
- [ ] 3. LibreNMS fork cloned, correct service account/permissions and upstream Git remote verified.
- [ ] 4. MariaDB and PHP configuration completed.
- [ ] 5. LibreNMS development dependencies, application settings and web access configured.
- [ ] 6. Validation/tests completed; a known-good Proxmox snapshot taken.
- [ ] 7. NetBox integration feature branch created.

The checklist reflects the **new** VM only. The old Ubuntu 24.04 VM had finished steps 1–3 before the rebuild decision, but that work no longer counts toward this checklist.

## 0. Proxmox VM and OS installation (rebuild target)

| Setting | Target |
|---|---|
| VM ID / hostname | 107 / `librenms-dev` |
| Proxmox VM name | `LibreNMS-DEV` |
| Operating system | **Ubuntu Server 26.04.1 LTS (Resolute Raccoon)** |
| Install media | Standard `ubuntu-26.04.1-live-server-amd64.iso` (not a cloud `.img`) |
| CPU | 8 vCPUs |
| Memory | 8 GiB (roughly 8192 MiB) |
| Disk | 200 GiB on SCSI / VirtIO SCSI single |
| NIC | VirtIO on `vmbr0` |
| VLAN | Follow the actual Proxmox trunk configuration. Earlier VM had `tag=1`; verify whether VLAN 1 should be tagged or untagged. |
| User | `chanley` |
| SSH | Select **Install OpenSSH Server** during install |
| Cloud-Init | **Not used** |
| QEMU Guest Agent | Enable in Proxmox; install `qemu-guest-agent` inside Ubuntu |

Official ISO location: https://releases.ubuntu.com/26.04/ .

Use the standard Ubuntu installer. After installation, eject the installer ISO and ensure the virtual disk is first in boot order. Do not attach a separate Cloud-Init drive.

**Checkpoint:**

```bash
cat /etc/os-release
hostnamectl
lsblk
df -h /
ip -br addr
ip route
```

Confirm `VERSION_ID="26.04"`, the expected hostname, the root filesystem and a reachable management address **before installing dependencies**.

**Actual rebuilt-VM results, verified 2026-10-15:**

```text
PRETTY_NAME="Ubuntu 26.04.1 LTS"
VERSION_ID="26.04"
VERSION_CODENAME=resolute
git version 2.53.0
php: command not found
Ubuntu apt suggested php-cli (PHP 8.5 family, including php8.5-cli 8.5.4-0ubuntu1.1).
```

These values are baseline evidence, not evidence of package or application installation. Hostname/network/SSH checks from this section have not yet been supplied.

## 1. Baseline tools and SSH (run on the NEW Ubuntu VM)

Log in as `chanley`; use `sudo` for privileged operations:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y openssh-server qemu-guest-agent git vim curl wget ca-certificates
sudo systemctl enable --now ssh
sudo systemctl enable --now qemu-guest-agent
```

**Checkpoint:**

```bash
cat /etc/os-release
git --version
sudo systemctl is-active ssh
sudo systemctl is-active qemu-guest-agent
ip -br addr
```

From the administrator workstation, verify SSH connectivity as `chanley` using the VM's actual IP. Prefer SSH keys for long-term access. Do not add passwords, SSH keys, GitHub access tokens, or database credentials to this README.

## 2. LibreNMS prerequisites on Ubuntu 26.04

LibreNMS's current installation guide requires **PHP >= 8.4**, recommends **PHP 8.5**, and has an Ubuntu 26.04 package list. Re-check upstream instructions before each rebuild: https://docs.librenms.org/Installation/Install-LibreNMS/ .

**Current status:** The full prerequisite package command below completed on the new Ubuntu 26.04 VM on 2026-10-15. The earlier initial OS check showed PHP absent; PHP **8.5.4** is now installed from Ubuntu's official package repository. APT reported installing 176 packages. No installation errors appeared in the supplied log. The next actions are configuring LibreNMS and verifying its runtime.

```bash
sudo apt update
sudo apt install -y \
  acl curl fping git mariadb-client mariadb-server \
  mtr-tiny nginx-full nmap php-cli php-curl php-fpm \
  php-gd php-gmp php-json php-mbstring php-mysql \
  php-snmp php-xml php-zip python3-command-runner \
  python3-dotenv python3-pip python3-psutil \
  python3-pymysql python3-redis python3-setuptools \
  python3-systemd rrdtool snmp snmpd traceroute \
  unzip whois
```

**Checkpoint — do not continue if PHP is below 8.4:**

```bash
php -v
php -m
mariadb --version
nginx -v
git --version
```

**Verified results (2026-10-15, rebuilt Ubuntu 26.04.1):**

```text
PHP 8.5.4 (cli)
mariadb from 11.8.6-MariaDB
nginx version: nginx/1.28.3 (Ubuntu)
git version 2.53.0
systemctl is-active mariadb   -> active
systemctl is-active nginx     -> active
systemctl is-active php8.5-fpm -> active
```

APT reported 176 packages installed, 0 upgraded and 0 removed by the prerequisite-install command, with 3 packages left 'Not Upgrading'. Package triggers completed and `needrestart` reported some deferred restarts and the then-current SSH session holding outdated binaries; a controlled reboot was recommended before continuing. No fatal package errors were visible. Reverify PHP, MariaDB, NGINX and PHP-FPM after reboot. `snmpd` was installed as a dependency but **SNMPv3 configuration has not yet been completed**.

### Phase 2 package-install evidence (2026-10-15)

Executed the prerequisite install command shown above on the replacement Ubuntu 26.04 VM. Verified versions and three active services. The install output is retained in the build conversation; it is not necessary to commit all 1,000+ APT log lines. After reboot, verify with:

```bash
php -v
mariadb --version
nginx -v
git --version
systemctl is-active mariadb nginx php8.5-fpm
```

**Do not proceed to phase 3 if any of the service checks is not active.**

## 3. LibreNMS source, account and Git remotes

This is the personal **LibreNMS development fork** explicitly selected for this environment:

- `origin` = https://github.com/Null0Engineer/LibreNMS-Development.git
- `upstream` = https://github.com/librenms/librenms.git
- Initial default branch = `master`
- Local working directory = `/opt/librenms`

Note: This development repository is an explicit exception for this project to the general preference for writing only to `chanley-home` repositories. Do not push unrelated changes or use the production LibreNMS server.

```bash
sudo useradd -r -M -d /opt/librenms -s /bin/bash librenms
sudo git clone \
  https://github.com/Null0Engineer/LibreNMS-Development.git \
  /opt/librenms
sudo chown -R librenms:librenms /opt/librenms
sudo -u librenms git -C /opt/librenms remote add upstream \
  https://github.com/librenms/librenms.git
sudo -u librenms git -C /opt/librenms fetch upstream
```

If rebuilding onto a VM where the `librenms` account or `upstream` remote already exists, inspect it rather than blindly repeating account/remote creation.

**Checkpoint:**

```bash
sudo -u librenms git -C /opt/librenms remote -v
sudo -u librenms git -C /opt/librenms status -sb
sudo -u librenms git -C /opt/librenms branch -vv
```

Expected: `origin` points to the development fork, `upstream` to LibreNMS, current branch `master` tracks `origin/master`, and the working tree is clean. A fetch is safe; **do not automatically push** to either remote or merge upstream before reviewing changes. Development should proceed on descriptive feature branches.

## 4. Next implementation stages — NOT YET VERIFIED ON REPLACEMENT VM

These are the remaining installation stages, not claims of completion:

1. Configure MariaDB using the **current upstream install guide**, with a separate local `librenms` development database and noncommitted credentials.
2. Apply LibreNMS's documented file permissions/ACLs and configure the PHP timezone and PHP-FPM pool/socket for the installed PHP version.
3. Install Composer **development** dependencies as the `librenms` user. The upstream contributor guide uses `./scripts/composer_wrapper.php install` (without `--no-dev`); do not use production-only dependencies for feature development.
4. Set `.env` with `APP_ENV=local` and `APP_DEBUG=true` **only for this isolated development VM**; never expose debug output publicly or commit `.env`.
5. Configure an appropriate development web interface: `./lnms serve` is the upstream contributor quick-start; NGINX/PHP-FPM can be configured for a persistent VM using the official install guide.
6. Run `./validate.php`, `./lnms dev:check`, relevant unit/integration tests, and upstream format checks where applicable; capture output and correct failures before feature work.
7. Snapshot VM 107 once working, before starting the NetBox IPAM feature branch.
8. Document **each actual command, configuration change, verification output, and fix** in this file as the build proceeds. Do not put secrets in the build log.

## History: discarded Ubuntu 24.04 build (reference only)

On 2026-10-15, the first **manual** Ubuntu install was identified as Ubuntu 24.04 and the VM was being rebuilt for 26.04. Previous output verified:

| Historical item | Previous result (NOT the new VM) |
|---|---|
| PHP | `8.3.6` — below currently documented LibreNMS minimum |
| MariaDB | `10.11.14` |
| NGINX | `1.24.0` |
| Git | `2.43.0` |
| Git checkout | `/opt/librenms` |
| `origin` | `Null0Engineer/LibreNMS-Development` |
| `upstream` | `librenms/librenms` |
| Git branch | `master`, tracking `origin/master` |
| Earlier commit | `3f14909a6b` |

The Git clone, ownership change, upstream fetch and remote/branch checks were completed on **that discarded build only**. The abandoned Proxmox cloud-image attempt was replaced by the conventional ISO installation and is not part of this rebuild procedure.

## New VM verification record (fill in as we go)

| Item | New VM result | Date / status |
|---|---|---|
| Ubuntu release | `26.04.1 LTS` / `resolute` | Verified 2026-10-15 |
| PHP version | `8.5.4` | Verified installed 2026-10-15 |
| MariaDB version | `11.8.6`; service active | Verified 2026-10-15 |
| NGINX version | `1.28.3`; service active | Verified 2026-10-15 |
| Git version | `2.53.0` | Verified 2026-10-15 |
| Git remotes | Pending on rebuilt VM | Not configured/verified |
| LibreNMS validation | Pending | Not verified |
| Snapshot name | Pending | Not created |

**Documentation rule:** Update this file during the build; keep completed steps, failed attempts (briefly, with correction), exact commands and verification checks clearly distinguished. Recheck official LibreNMS development guidance before adding or changing project code.
