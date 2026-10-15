# README-BUILD — LibreNMS Development VM

> **Build log / reproducible rebuild guide** — updated 2026-10-15.
>
> **Current build status (2026-10-15):** Ubuntu 26.04.1, prerequisites, Git fork and upstream remotes, MariaDB permissions, PHP 8.5 timezones, and PHP-FPM pool verified. **Phase 4 is complete.** Composer downloaded and installed 202 dependencies including require-dev and finished Laravel package discovery/caching, but logged nonfatal missing-directory ACL warnings because runtime folders were absent. **Phase 5 remains in progress** pending filesystem repair, Composer recheck, `.env`, web access and validation. Earlier Ubuntu 24.04 and Cloud-Init attempts were discarded.

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
- [x] 3. LibreNMS fork cloned, correct service account/permissions and upstream Git remote verified (2026-10-15; clean checkout on `master`).
- [x] 4. MariaDB and PHP configuration completed (timezone CLI/FPM verified, local `librenms` database granted ALL PRIVILEGES, PHP-FPM pool active; 2026-10-15).
- [ ] 5. LibreNMS development dependencies, application settings and web access configured (Composer dependencies installed, but runtime directories/ACLs, `.env` and web setup pending).
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

APT reported 176 packages installed, 0 upgraded and 0 removed by the prerequisite-install command, with 3 packages left 'Not Upgrading'. Package triggers completed and `needrestart` reported some deferred restarts and the then-current SSH session holding outdated binaries; a controlled reboot was recommended before continuing. No fatal package errors were visible. Reverify PHP, MariaDB, NGINX and PHP-FPM after reboot. `snmpd` was installed as a prerequisite but **SNMPv3 configuration has not yet been completed**.

**SNMPv3-only policy:** Do not leave the package's sample SNMP agent configuration serving an older protocol. Until its SNMPv3-only listener is explicitly reviewed and configured, run `systemctl disable --now snmpd`. This does not prevent LibreNMS from polling other devices using SNMPv3. Restore this service only after a compliant SHA/AES authPriv configuration is in place. This corrective command is **recommended, not yet verified executed** on the new VM.

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

**Confirmed on the replacement Ubuntu 26.04.1 VM, 2026-10-15:**

- `systemctl disable --now snmpd` succeeded; systemd removed the service enablement symlink. This satisfies the precaution to keep the unconfigured SNMP agent off.
- `librenms` service account creation, fork clone and ownership command completed without reported errors.
- Fork checkout: 17,264 files, receiving/resolving objects 100%.
- Fetch from the official `librenms/librenms` upstream completed successfully, including `upstream/master`.
- `origin`: `https://github.com/Null0Engineer/LibreNMS-Development.git` for fetch and push.
- `upstream`: `https://github.com/librenms/librenms.git` for fetch and push.
- `git status -sb`: `## master...origin/master`, with no file changes listed.
- `git branch -vv`: `* master 43bfcc953a [origin/master] docs: note SNMPv3-only precaution on new LibreNMS VM`.
- **Phase 3 finished.** This SHA is the *checkout observed at verification time*. The build README itself may receive newer documentation commits; fetch/pull updates only after confirming the local worktree is clean.

## 4. MariaDB and PHP configuration — IN PROGRESS

Apply only on the **new Ubuntu 26.04.1** VM as root. **Operator reported completion of MariaDB configuration and database/user creation on 2026-10-15**, but command output has not yet been submitted for validation. Timezone and PHP-FPM tasks are still pending. Source: https://docs.librenms.org/Installation/Install-LibreNMS/ with Ubuntu 26.04 selected.

### 4A. Set server and PHP timezones

```bash
timedatectl set-timezone America/New_York
timedatectl
vi /etc/php/8.5/cli/php.ini
vi /etc/php/8.5/fpm/php.ini
```

In each PHP ini file ensure:

```ini
date.timezone = America/New_York
```

Check:

```bash
php -r 'echo date_default_timezone_get(), PHP_EOL;'
```

### 4B. Configure MariaDB — VERIFIED 2026-10-15

```bash
vi /etc/mysql/mariadb.conf.d/50-server.cnf
```

Under the existing `[mariadbd]` section for Ubuntu 26.04, ensure these settings occur once:

```ini
innodb_file_per_table=1
lower_case_table_names=0
```

Restart and check:

```bash
systemctl enable mariadb
systemctl restart mariadb
systemctl is-active mariadb
mariadb -e "SHOW VARIABLES WHERE Variable_name IN ('innodb_file_per_table','lower_case_table_names');"
```

Expected: MariaDB active; `innodb_file_per_table=ON`, `lower_case_table_names=0`.

### 4C. Create a **local dev-only** MariaDB database and account — database/login verified, grants pending

Enter `mariadb` as root. Substitute a generated **unique** password locally for the placeholder; do not commit the password or paste it into the build log.

```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'REPLACE_WITH_UNIQUE_PASSWORD';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
EXIT;
```

Check without exposing secrets:

```bash
mariadb -e "SHOW DATABASES LIKE 'librenms';"
mariadb -e "SHOW GRANTS FOR 'librenms'@'localhost';"
```

Do not use a production database or shared production credentials.

**Post-configuration verification (still pending on this VM):**

```bash
systemctl is-active mariadb
mariadb -e "SHOW VARIABLES WHERE Variable_name IN ('innodb_file_per_table','lower_case_table_names');"
mariadb -e "SHOW DATABASES LIKE 'librenms';"
mariadb -e "SHOW GRANTS FOR 'librenms'@'localhost';"
```

Confirm the application user can authenticate with its own password, without displaying it on screen or saving it in the shell history:

```bash
mariadb -u librenms -p -e "SELECT CURRENT_USER();"
```

**2026-10-15 progress:** user said **“done”** after being instructed to configure MariaDB and create the dev database/user. Treat 4B and 4C as *operator-reported complete*, not instrument-verified, until these commands pass. Do **not** commit any credentials.

### 4D. PHP-FPM pool — FIXED and VERIFIED 2026-10-15

LibreNMS upstream calls for copying the stock `www.conf` and changing the pool name, user/group and unique listen socket. **Keep the original process-manager directives** from that file: PHP-FPM will not start if the `pm` directive is absent. This was missed in earlier rebuild guidance, and it produced an actual failure during setup.

First-time setup (**do not overwrite an existing edited pool on a running VM**):

```bash
cp /etc/php/8.5/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/librenms.conf
vi /etc/php/8.5/fpm/pool.d/librenms.conf
```

Under the `[librenms]` section, ensure **exactly one active entry per setting** (keep other needed defaults in the copied pool):

```ini
[librenms]
user = librenms
group = librenms
listen = /run/php-fpm-librenms.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

These modest worker counts are initial values for this 8 GiB development VM, not benchmarked production tuning. `pm` and `pm.max_children` are mandatory; dynamic also requires the `pm.min_spare_servers` and `pm.max_spare_servers` settings.

**Observed failure (2026-10-15):**

```text
ALERT: [pool librenms] the process manager is missing (static, dynamic or ondemand)
ERROR: failed to post process the configuration
ERROR: FPM initialization failed
Job for php8.5-fpm.service failed because the control process exited with error code.
systemctl is-active php8.5-fpm -> activating (not active)
ls: cannot access '/run/php-fpm-librenms.sock': No such file or directory
```

**Recovery from this exact failure** — edit the existing file, do not repeat `cp` (that would erase it). After saving:

```bash
grep -nE '^\[|^[[:space:]]*pm([[:space:]]*=|\.)' /etc/php/8.5/fpm/pool.d/librenms.conf
php-fpm8.5 -t
```

**Only when the syntax check succeeds:**

```bash
systemctl restart php8.5-fpm
systemctl is-active php8.5-fpm
ls -l /run/php-fpm-librenms.sock
```

Expected: successful `php-fpm8.5 -t`, service `active`, and the socket present. If failed, use:

```bash
systemctl status php8.5-fpm --no-pager -l
journalctl -xeu php8.5-fpm --no-pager -n 60
```

Keep the default `www.conf` pool until the new pool is verified. The PHP-FPM socket must match NGINX's later `fastcgi_pass` value. Refer to [official PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php) and [LibreNMS installation](https://docs.librenms.org/Installation/Install-LibreNMS/) for authoritative requirements.

**Verified recovery on 2026-10-15 10:22 EDT:**

```text
php-fpm8.5 -t -> configuration file /etc/php/8.5/fpm/php-fpm.conf test is successful
systemctl restart php8.5-fpm -> succeeded
systemctl is-active php8.5-fpm -> active
/run/php-fpm-librenms.sock -> srw-rw---- www-data:www-data
systemctl status php8.5-fpm -> active (running), enabled
Processes:
  php-fpm: pool www (2 workers)
  php-fpm: pool librenms (2 workers)
```

The journal still contains the **10:20** failed attempts but shows a successful **10:22:11** start. Do not treat past log errors as current service failures after verification.

**MariaDB verified on 2026-10-15:**

```text
systemctl is-active mariadb -> active
innodb_file_per_table -> ON
lower_case_table_names -> 0
SHOW DATABASES LIKE 'librenms' -> librenms
mariadb -u librenms -p -e "SELECT CURRENT_USER();" -> librenms@localhost
```

Two initial password-login attempts returned `ERROR 1045`; the third succeeded without database changes. No password was recorded in Git. **The specific database grants remain to be verified**; a successful login alone does not prove write permission.

**Remaining Phase 4 validation:**

```bash
timedatectl show -p Timezone --value
php -r 'echo date_default_timezone_get(), PHP_EOL;'
grep -nE '^[[:space:]]*date\.timezone[[:space:]]*=' /etc/php/8.5/cli/php.ini /etc/php/8.5/fpm/php.ini
mariadb -e "SHOW GRANTS FOR 'librenms'@'localhost';"
```

**Verified on rebuilt VM, 2026-10-15:**

```text
timedatectl -> America/New_York
PHP CLI date_default_timezone_get() -> America/New_York
/etc/php/8.5/cli/php.ini -> date.timezone = America/New_York
/etc/php/8.5/fpm/php.ini -> date.timezone = America/New_York
MariaDB SHOW GRANTS -> GRANT ALL PRIVILEGES ON librenms.* TO librenms@localhost
```

The MariaDB response also contained a password verifier/hash; **never copy that value into Git, support tickets or the build log**. **Phase 4 verified complete.**

## 5. LibreNMS development dependencies and runtime — IN PROGRESS

### 5A. Runtime directories and ACLs — required before Composer on rebuilds

Official installation guide: https://docs.librenms.org/Installation/Install-LibreNMS/ . On the first attempt (2026-10-15), this directory/ACL phase was missed, so the Composer install ended with these nonfatal messages:

```text
Could not scan for classes inside "vendor/dapphp/radius" which does not appear to be a file nor a folder
...
> LibreNMS\\ComposerHelper::postInstall
setfacl: rrd/: No such file or directory
setfacl: logs/: No such file or directory
setfacl: storage/: No such file or directory
setfacl: bootstrap/cache/: No such file or directory
```

The `dapphp/radius` missing-path scan was an initial autoload warning: later Composer output shows the library successfully extracted. Do not conclude the install failed from that first line alone. For future clean rebuilds, **create runtime directories and set permissions before the Composer install**.

Run as `root` (or `sudo` each privileged command) from any working directory:

```bash
install -d -o librenms -g librenms -m 0775 \
  /opt/librenms/rrd \
  /opt/librenms/logs \
  /opt/librenms/storage \
  /opt/librenms/storage/framework/cache/data \
  /opt/librenms/storage/framework/sessions \
  /opt/librenms/storage/framework/views \
  /opt/librenms/bootstrap/cache

chown -R librenms:librenms \
  /opt/librenms/rrd \
  /opt/librenms/logs \
  /opt/librenms/storage \
  /opt/librenms/bootstrap/cache

chmod 771 /opt/librenms
setfacl -d -m g::rwx \
  /opt/librenms/rrd \
  /opt/librenms/logs \
  /opt/librenms/storage \
  /opt/librenms/bootstrap/cache

setfacl -R -m g::rwx \
  /opt/librenms/rrd \
  /opt/librenms/logs \
  /opt/librenms/storage \
  /opt/librenms/bootstrap/cache
```

Verify the `librenms` service account can write to the runtime directories:

```bash
for d in rrd logs storage bootstrap/cache; do
  if sudo -u librenms test -w "/opt/librenms/$d"; then
    echo "OK: $d writable"
  else
    echo "FAIL: $d not writable"
  fi
done

getfacl -p /opt/librenms/storage
```

**Status:** commands proposed as corrective action; execution/result **not yet supplied**.

### 5B. Composer development dependencies — packages installed, recheck pending

From the LibreNMS development instructions: https://docs.librenms.org/Developing/Getting-Started/ . Run as the `librenms` service user (without `--no-dev`):

```bash
su - librenms
cd /opt/librenms
./scripts/composer_wrapper.php install
```

**Observed 2026-10-15:**

```text
Installing dependencies from lock file (including require-dev)
Package operations: 202 installs, 0 updates, 0 removals
Laravel package discovery: DONE
Blade templates cached successfully
Laravel optimize: DONE (config/events/routes/views/laravel-data)
Configuration cache cleared successfully
Returned to librenms shell prompt
```

Composer warned `influxdb/influxdb-php` is abandoned with no replacement suggested. This is an upstream dependency warning; do **not** remove or replace locked packages arbitrarily in this downstream fork. The initial `dapphp/radius` class-scan warning did not prevent extraction.

**After fixing runtime directories**, rerun `./scripts/composer_wrapper.php install` as `librenms` and verify the `setfacl: ... No such file or directory` warnings are gone. Capture results. Do not mark this recheck complete until observed.

### 5C. Local development environment — PENDING

Set `.env` with database details (not committed) and `APP_ENV=local`, `APP_DEBUG=true` **only for an isolated development VM**. Never expose Laravel debug mode publicly. Install/setup of local key, migrations and remaining application configuration should follow the official upstream developer instructions and any current install prerequisites.

### 5D. Development web interface — PENDING

Official contributor quick-start: `./lnms serve` (local port 8000). For persistent NGINX with PHP-FPM, configure a proper `/opt/librenms/html` web root and use `/run/php-fpm-librenms.sock` as the FastCGI socket per official LibreNMS instructions. Restrict network access while local debugging is enabled.

### 5E. Validation, snapshot, feature branch — PENDING

Run `./validate.php`, `./lnms dev:check`, applicable tests/lint, and document the output and fixes. After a validated baseline, take a Proxmox VM 107 snapshot before beginning NetBox IPAM development on a new feature branch.

**Documentation rule:** Record exactly what was executed and verified, distinguish warnings from failures, do not copy credential hashes or other secrets, and do not overwrite existing user-created configuration during a rebuild.

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
| Git remotes | `origin`: `Null0Engineer/LibreNMS-Development`; `upstream`: `librenms/librenms` | Verified 2026-10-15; `master` tracking `origin/master` |
| PHP-FPM pool | `php8.5-fpm` running; `librenms` workers; socket `www-data:www-data` | Verified 2026-10-15 10:22 EDT |
| MariaDB login | `librenms@localhost` | Verified 2026-10-15 (privilege grant detail pending) |
| PHP timezone | `America/New_York` (system, CLI, FPM) | Verified 2026-10-15 |
| MariaDB privileges | `GRANT ALL PRIVILEGES ON librenms.*` | Verified 2026-10-15 |
| Composer dependencies | 202 including require-dev; postinstall warning for 4 missing dirs | Download/install completed; ACL fix pending |
| LibreNMS validation | Pending | Not verified |
| Snapshot name | Pending | Not created |

**Documentation rule:** Update this file during the build; keep completed steps, failed attempts (briefly, with correction), exact commands and verification checks clearly distinguished. Recheck official LibreNMS development guidance before adding or changing project code.
