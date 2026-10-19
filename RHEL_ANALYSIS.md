# RHEL Porting Analysis: premi.sh (Alround-Vpn)

**Source:** `~/workspace/alround-vpn/premi.sh` (1014 lines, Ubuntu/Debian tunneling installer)
**Target:** AlmaLinux 8/9, Rocky Linux 8/9, CentOS Stream 8/9
**Date:** 2026-10-20

---

## Executive Summary

The script is **heavily Debian/Ubuntu-specific**. A direct port requires changes in ~15 distinct areas. The recommended approach is a **new `rhel.sh`** (not modifying `premi.sh`), because:

1. The OS detection logic (lines 54-65) is if/elif on `ubuntu`/`debian` — adding RHEL branches to every function would make the script unmaintainable.
2. Package names, service names, and config paths differ in enough places that conditional logic would double the script size.
3. SELinux and firewalld require RHEL-specific setup blocks with no Debian equivalent.

**Estimated effort:** ~60-70% of the script needs RHEL-specific equivalents. The download-and-configure logic (wget configs, sed replacements, systemd units) is portable; the package installation and system setup is not.

---

## 1. Package Manager Commands

| Line(s) | Debian/Ubuntu | RHEL Equivalent | Notes |
|---------|---------------|-----------------|-------|
| 2 | `apt update -y` | `dnf makecache -y` | |
| 3-4 | `apt install curl -y`, `apt install wondershaper -y` | `dnf install -y curl` | `wondershaper` NOT in RHEL repos (see §2) |
| 208 | `apt-get install --no-install-recommends software-properties-common` | **N/A** — no equivalent needed | RHEL uses `dnf config-manager` (from `dnf-plugins-core`) for repo management |
| 207, 224 | `apt update -y` / `apt-get update -y` | `dnf makecache -y` | |
| 215, 225 | `apt-get install -y haproxy` | `dnf install -y haproxy` | |
| 238 | `sudo apt-get install nginx -y` | `dnf install -y nginx` | |
| 243 | `apt -y install nginx` | `dnf install -y nginx` | |
| 252-253 | `apt install zip pwgen openssl netcat socat cron bash-completion -y` | `dnf install -y zip pwgen openssl netcat socat cronie bash-completion` | `cron` → `cronie` on RHEL |
| 254 | `apt install figlet -y` | `dnf install -y figlet` | Requires EPEL |
| 255-258 | `apt update -y`, `apt upgrade -y`, `apt dist-upgrade -y` | `dnf upgrade -y` | `dist-upgrade` has no direct equivalent; `dnf upgrade` covers it |
| 262 | `sudo apt-get clean all` | `dnf clean all` | |
| 263 | `sudo apt-get autoremove -y` | `dnf autoremove -y` | |
| 264 | `sudo apt-get install -y debconf-utils` | **DELETE** — Debian-only | `debconf` does not exist on RHEL |
| 265-266 | `sudo apt-get remove --purge exim4 -y`, `sudo apt-get remove --purge ufw firewalld -y` | `dnf remove -y postfix firewalld` (if desired) | **WARNING:** Removing firewalld on RHEL is discouraged; use firewalld rules instead (see §7) |
| 267 | `sudo apt-get install -y --no-install-recommends software-properties-common` | **DELETE** | No equivalent needed |
| 268 | *(big package list — see §2 table)* | *(see §2 table)* | |
| 460 | `DEBIAN_FRONTEND=noninteractive dpkg-reconfigure keyboard-configuration` | **DELETE** | Debian-only |
| 461-478 | `debconf-set-selections <<<"keyboard-configuration ..."` (18 lines) | **DELETE** | Debian-only, no RHEL equivalent |
| 204-205, 264-265 | `echo iptables-persistent ... \| debconf-set-selections` | **DELETE** | Debian-only |
| 620 | `apt-get install dropbear -y` | **PROBLEM** — see §2 | Dropbear not in RHEL base/EPEL |
| 634, 636 | `apt -y install vnstat`, `apt -y install libsqlite3-dev` | `dnf install -y vnstat sqlite-devel` | |
| 665 | `apt install rclone -y` | `dnf install -y rclone` | Requires EPEL |
| 672 | `apt install msmtp-mta ca-certificates bsd-mailx -y` | `dnf install -y msmtp ca-certificates mailx` | `bsd-mailx` → `mailx` on RHEL |
| 690 | `dpkg -i /tmp/gotop.deb` | `rpm -i /tmp/gotop.rpm` | Must download `.rpm` not `.deb` (see §2) |
| 708 | `apt -y install fail2ban` | `dnf install -y fail2ban` | Requires EPEL |
| 740-741 | `apt autoclean -y`, `apt autoremove -y` | `dnf clean all`, `dnf autoremove -y` | |

---

## 2. Package Name Mapping (Debian → RHEL)

### Core Services

| Debian/Ubuntu | RHEL 8/9 | Source | Notes |
|---------------|----------|--------|-------|
| `nginx` | `nginx` | AppStream | Same name ✅ |
| `haproxy` | `haproxy` | AppStream | Same name ✅ — RHEL 8 has 1.8, RHEL 9 has 2.4. **Note:** `bind-process` removed in 2.5+; script already handles this |
| `openvpn` | `openvpn` | EPEL | Same name ✅ |
| `easy-rsa` | `easy-rsa` | EPEL | Same name ✅ |
| `dropbear` | **NOT AVAILABLE** | — | ❌ Not in BaseOS, AppStream, or EPEL. Options: (a) compile from source, (b) use OpenSSH only, (c) COPR repo. **Recommendation:** Drop Dropbear on RHEL, use OpenSSH with multiple ports |
| `fail2ban` | `fail2ban` | EPEL | Same name ✅ |
| `vnstat` | `vnstat` | EPEL | Same name ✅ |
| `rclone` | `rclone` | EPEL | Same name ✅ |
| `squid` (if used) | `squid` | AppStream | |

### Utilities & Libraries

| Debian/Ubuntu | RHEL 8/9 | Source | Notes |
|---------------|----------|--------|-------|
| `curl` | `curl` | BaseOS | ✅ |
| `wget` | `wget` | AppStream | ✅ |
| `git` | `git` | AppStream | ✅ |
| `python3` | `python3` | AppStream | ✅ |
| `python3-pip` | `python3-pip` | AppStream | ✅ |
| `cron` | `cronie` | BaseOS | ⚠️ **Different name** — service is `crond` |
| `bash-completion` | `bash-completion` | BaseOS | ✅ |
| `zip` / `unzip` | `zip` / `unzip` | BaseOS | ✅ |
| `p7zip-full` | `p7zip` | EPEL | ⚠️ Different name |
| `tar` | `tar` | BaseOS | ✅ |
| `htop` | `htop` | EPEL | ✅ |
| `lsof` | `lsof` | BaseOS | ✅ |
| `net-tools` | `net-tools` | BaseOS | ✅ (provides `netstat`, `ifconfig`) |
| `netcat` | `nmap-ncat` | BaseOS | ⚠️ **Different name** — `netcat` is not a package; use `nmap-ncat` (provides `nc`) |
| `socat` | `socat` | AppStream | ✅ |
| `screen` | `screen` | EPEL | ✅ |
| `jq` | `jq` | AppStream | ✅ |
| `bc` | `bc` | BaseOS | ✅ |
| `figlet` | `figlet` | EPEL | ✅ |
| `pwgen` | `pwgen` | EPEL | ✅ |
| `dos2unix` | `dos2unix` | BaseOS | ✅ |
| `xz-utils` | `xz` | BaseOS | ⚠️ Different name |
| `dnsutils` | `bind-utils` | AppStream | ⚠️ **Different name** — provides `dig`, `nslookup` |
| `ntpdate` | **DEPRECATED** | — | ❌ Not in RHEL 8/9. Use `chrony` only (already in script) |
| `chrony` | `chrony` | BaseOS | ✅ |
| `sudo` | `sudo` | BaseOS | ✅ |
| `openssl` | `openssl` | BaseOS | ✅ |
| `ca-certificates` | `ca-certificates` | BaseOS | ✅ |
| `gnupg` / `gnupg2` | `gnupg2` | AppStream | ⚠️ Just `gnupg2` on RHEL |
| `gnupg1` | **NOT AVAILABLE** | — | ❌ No longer packaged; use `gnupg2` |
| `lsb-release` | `redhat-lsb-core` | AppStream | ⚠️ Different name (only if needed) |
| `rsyslog` | `rsyslog` | AppStream | ✅ |
| `ruby` | `ruby` | AppStream | ✅ |
| `make` / `gcc` / `g++` | `make` / `gcc` / `gcc-c++` | AppStream | ⚠️ `g++` → `gcc-c++` |
| `cmake` | `cmake` | AppStream | ✅ |
| `flex` / `bison` | `flex` / `bison` | AppStream | ✅ |
| `pkg-config` | `pkgconf-pkg-config` | AppStream | ⚠️ Different name (provides `pkg-config`) |
| `shc` | **NOT AVAILABLE** | — | ❌ Not in RHEL/EPEL. Needed? Check usage. |
| `wondershaper` | **NOT AVAILABLE** | — | ❌ Not in RHEL/EPEL. Used for bandwidth shaping in `ins_backup`. Either skip or install from GitHub (script already does `git clone` + `make install` for wondershaper in `ins_backup`, so the `apt install wondershaper` at line 4 is redundant) |

### Development Libraries (for building vnstat from source, etc.)

| Debian/Ubuntu | RHEL 8/9 | Notes |
|---------------|----------|-------|
| `build-essential` | `gcc gcc-c++ make` (or groupinstall "Development Tools") | ⚠️ Use `dnf groupinstall -y "Development Tools"` |
| `libssl-dev` | `openssl-devel` | ⚠️ |
| `libsqlite3-dev` | `sqlite-devel` | ⚠️ |
| `zlib1g-dev` | `zlib-devel` | ⚠️ |
| `libevent-dev` | `libevent-devel` | ⚠️ |
| `libnss3-dev` / `libnss3-tools` | `nss-devel` / `nss-tools` | ⚠️ |
| `libnspr4-dev` | `nspr-devel` | ⚠️ |
| `libpam0g-dev` | `pam-devel` | ⚠️ |
| `libcap-ng-dev` / `libcap-ng-utils` | `libcap-ng-devel` / `libcap-ng-utils` | ⚠️ |
| `libselinux1-dev` | `libselinux-devel` | ⚠️ |
| `libxml-parser-perl` | `perl-XML-Parser` | ⚠️ |
| `libc6` | `glibc` | ⚠️ (usually pre-installed) |
| `util-linux` | `util-linux` | ✅ |
| `dirmngr` | Provided by `gnupg2` | ✅ (no separate package) |
| `apt-transport-https` | **N/A** | DNF handles HTTPS natively |
| `software-properties-common` | **N/A** | Use `dnf config-manager` from `dnf-plugins-core` |
| `debconf-utils` | **N/A** | Debian-only |
| `iptables-persistent` | `iptables-services` | ⚠️ **Different name** — provides `iptables-save`/`iptables-restore` persistence |
| `netfilter-persistent` | **N/A** | Debian wrapper; on RHEL use `iptables-services` directly |
| `msmtp-mta` | `msmtp` | ⚠️ Different name |
| `bsd-mailx` | `mailx` | ⚠️ Different name |

### Special Cases

| Item | Issue | RHEL Solution |
|------|-------|---------------|
| `gotop` .deb (line 688-690) | Downloads `gotop_*_linux_amd64.deb`, installs with `dpkg -i` | Download `.rpm` instead: `https://github.com/xxxserxxx/gotop/releases/download/v$ver/gotop_v${ver}_linux_amd64.rpm`, install with `rpm -i` |
| `iptables-persistent` / `netfilter-persistent` | Used for saving iptables rules (lines 730-732, 758, 766, 772) | Replace with `iptables-services`: `dnf install -y iptables-services`, then `systemctl enable --now iptables`, save with `iptables-save > /etc/sysconfig/iptables` |
| `ufw` / `firewalld` removal (line 266) | Script purges them on Debian | **Do NOT remove firewalld on RHEL.** Instead, configure firewalld rules for required ports (see §7) |

---

## 3. Repositories / PPAs

| Line(s) | Debian/Ubuntu | RHEL Equivalent |
|---------|---------------|-----------------|
| 208, 267 | `software-properties-common` (for `add-apt-repository`) | `dnf-plugins-core` (provides `dnf config-manager`) |
| 237 (commented) | `# sudo add-apt-repository ppa:nginx/stable -y` | Not needed — AppStream nginx is sufficient. For newer nginx, use the [official nginx repo](http://nginx.org/en/linux_packages.html#RHEL) |
| — | *(haproxy PPA removed in port; uses distro package)* | RHEL 8: HAProxy 1.8 (AppStream). RHEL 9: HAProxy 2.4 (AppStream). Both support the script's config (no `bind-process` needed after prior fix) |
| — | — | **EPEL is REQUIRED** for: `fail2ban`, `vnstat`, `rclone`, `openvpn`, `easy-rsa`, `htop`, `figlet`, `pwgen`, `p7zip`, `screen`, `wondershaper` (N/A). Enable with: `dnf install -y epel-release` |
| — | `apt-transport-https` | Not needed — DNF handles HTTPS natively |

**RHEL repo setup block (add at start of `rhel.sh`):**
```bash
dnf install -y epel-release
dnf install -y dnf-plugins-core
dnf makecache -y
```

---

## 4. Service Management

### Service Name Differences

| Debian/Ubuntu Service | RHEL Service | Lines Affected |
|-----------------------|--------------|----------------|
| `ssh` | `sshd` | 616-618 (`/etc/init.d/ssh restart`, `systemctl restart ssh`) |
| `cron` | `crond` | 253 (package), 756 (`/etc/init.d/cron restart`), 774, 779, 897 |
| `netfilter-persistent` | `iptables` (from `iptables-services`) | 732, 758, 766, 772-773, 779 |

### Init Script Usage (Must Be Replaced)

The script uses `/etc/init.d/<service>` in multiple places. RHEL 8/9 still ships some init scripts but they are deprecated. **Replace all with `systemctl`:**

| Line(s) | Current | Replace With |
|---------|---------|--------------|
| 616-618 | `/etc/init.d/ssh restart` + `systemctl restart ssh` + `/etc/init.d/ssh status` | `systemctl restart sshd` |
| 624-625 | `/etc/init.d/dropbear restart` + `/etc/init.d/dropbear status` | `systemctl restart dropbear` *(if dropbear available)* |
| 638, 643-644 | `/etc/init.d/vnstat restart` (×2), `/etc/init.d/vnstat status` | `systemctl restart vnstat` |
| 652 | `/etc/init.d/openvpn restart` | `systemctl restart openvpn@server` *(see note)* |
| 750-756 | `/etc/init.d/nginx restart`, `/etc/init.d/openvpn restart`, `/etc/init.d/ssh restart`, `/etc/init.d/dropbear restart`, `/etc/init.d/fail2ban restart`, `/etc/init.d/vnstat restart`, `/etc/init.d/cron restart` | `systemctl restart nginx`, `systemctl restart openvpn@server`, `systemctl restart sshd`, `systemctl restart dropbear`, `systemctl restart fail2ban`, `systemctl restart vnstat`, `systemctl restart crond` |

**OpenVPN note:** On RHEL, OpenVPN uses templated units: `openvpn@server.service` (for `/etc/openvpn/server.conf`). The script's `limit/openvpn` installer must be checked for config file placement.

### Custom Systemd Units

The script creates many custom units in `/etc/systemd/system/` (`runn.service`, `vmip.service`, `vlip.service`, `trip.service`, `udp-mini-*.service`, `ws.service`, `rc-local.service`, `xray.service`). These are **portable** — systemd unit syntax is identical on RHEL. No changes needed except:
- `xray.service` uses `User=www-data` → change to `User=nginx` (see §8).

---

## 5. File Paths

| Path | Debian/Ubuntu | RHEL | Action |
|------|---------------|------|--------|
| `/etc/nginx/` | ✅ | ✅ Same | No change |
| `/etc/haproxy/` | ✅ | ✅ Same | No change |
| `/etc/ssh/sshd_config` | ✅ | ✅ Same | No change |
| `/etc/xray/` (custom) | ✅ | ✅ Same (script-created) | No change |
| `/etc/default/dropbear` | ✅ | ❌ `/etc/sysconfig/dropbear` | **Change** — but only if Dropbear is used (recommend dropping it) |
| `/etc/pam.d/common-password` (line 457) | ✅ | ❌ | **Change** — RHEL uses `/etc/pam.d/system-auth` and `/etc/pam.d/password-auth`. The script overwrites this file with a downloaded version; the RHEL PAM stack is different. **Needs RHEL-specific PAM config.** |
| `/etc/iptables.up.rules` (line 731) | ✅ (script-created) | ❌ | **Change** — use `/etc/sysconfig/iptables` with `iptables-services` |
| `/etc/rc.local` + `rc-local.service` | ✅ (script-created) | ⚠️ | RHEL 8/9: `rc-local` is deprecated but still works if enabled. Better to convert the iptables rules to a proper systemd unit or firewalld. |
| `/etc/cron.d/` | ✅ | ✅ Same | No change (`crond` reads it) |
| `/etc/shells` | ✅ | ✅ Same | No change |
| `/usr/local/sbin/` (menu scripts) | ✅ | ✅ Same | No change |
| `/var/log/xray/` | ✅ (script-created) | ✅ Same | No change, but `chown www-data:www-data` → `chown nginx:nginx` |
| `/run/xray` | ✅ (script-created) | ✅ Same | No change, but `chown www-data.www-data` → `chown nginx:nginx` |
| `/etc/msmtprc` | ✅ | ✅ Same | `chown www-data:www-data` → `chown nginx:nginx` (or root) |
| `/usr/share/zoneinfo/Asia/Jakarta` | ✅ | ✅ Same | No change |

---

## 6. SELinux

**Current status:** The script does NOT handle SELinux at all. It installs `libselinux1-dev` (build dep) but never configures SELinux.

**RHEL default:** SELinux is **enforcing** on AlmaLinux/Rocky/CentOS Stream.

**What will break without SELinux handling:**
| Component | SELinux Issue | Fix |
|-----------|---------------|-----|
| `nginx` → proxy to `127.0.0.1:10015` (ws), `:10001` (xray) | `httpd_can_network_connect` is off by default | `setsebool -P httpd_can_network_connect 1` |
| `haproxy` binding non-standard ports | May need `haproxy_connect_any` or custom port types | `setsebool -P haproxy_connect_any 1` |
| `xray` binary in `/usr/local/bin` | Needs `bin_t` context (usually OK) | Verify with `ls -Z` |
| Custom systemd services | Generally fine | — |
| `/etc/xray/`, `/var/log/xray/` | Custom dirs may need context | `semanage fcontext` + `restorecon`, or `chcon` |

**Recommended strategy (add to `rhel.sh`):**
```bash
# Option A (simple, less secure): disable SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# Option B (better): keep enforcing, set booleans
setsebool -P httpd_can_network_connect 1
setsebool -P haproxy_connect_any 1
```

**Recommendation:** Start with Option A (disable) for v1 of the RHEL port — the script's threat model (personal VPN) doesn't benefit much from SELinux, and getting the booleans right for all components (xray, ws, udp-mini, etc.) requires testing. Document Option B as a future improvement.

---

## 7. Firewall

**Current status:** The script **removes** `ufw` and `firewalld` (line 266) and uses raw `iptables` + `iptables-persistent`/`netfilter-persistent` with rules saved to `/etc/iptables.up.rules`.

**RHEL approach — two options:**

### Option A: Keep firewalld (RECOMMENDED)
Do NOT remove firewalld. Open required ports:
```bash
for port in 22 80 443 109 143 225-1000 2082 2086 2095 8080 8880 5300/udp 1194; do
    firewall-cmd --permanent --add-port=${port}/tcp  # (adjust udp where needed)
done
firewall-cmd --reload
```
Also handle the NAT redirect for SlowDNS (port 53 → 5300) via firewalld rich rules instead of the `rc.local` iptables hack.

### Option B: Use iptables-services (closer to original script)
```bash
dnf install -y iptables-services
systemctl disable --now firewalld
systemctl enable --now iptables
# ... script's iptables commands work as-is ...
iptables-save > /etc/sysconfig/iptables
```

**Recommendation:** Option B is a more faithful port (the script already manages raw iptables rules for BitTorrent blocking, SlowDNS redirect, etc.). Rewriting all of that for firewalld is error-prone. Use `iptables-services`.

**Lines affected:** 266 (remove firewalld purge), 730-732 (`iptables-save`, `netfilter-persistent save/reload`), 758, 766, 772-773, 779 (`netfilter-persistent` service), 911-915 (`rc.local` iptables rules).

---

## 8. User/Group Differences

The script hardcodes `www-data` in **5 places**. RHEL uses `nginx` as the web server user.

| Line | Current | RHEL |
|------|---------|------|
| 186 | `chown www-data.www-data /var/log/xray` | `chown nginx:nginx /var/log/xray` |
| 390 | `chown www-data.www-data $domainSock_dir` (`/run/xray`) | `chown nginx:nginx /run/xray` |
| 445 | `User=www-data` in `xray.service` | `User=nginx` |
| 678 | `chown -R www-data:www-data /etc/msmtprc` | `chown -R nginx:nginx /etc/msmtprc` (or `root:root`) |
| — | Xray install script's `-u www-data` flag (already removed in prior port) | N/A |

**Note:** The `ws` service and `udp-mini` may also reference users — check `limit/ws.service` and related files during porting.

---

## 9. Other Debian-Specifics

| Item | Lines | Debian | RHEL Action |
|------|-------|--------|-------------|
| `debconf-set-selections` | 204-205, 264-265, 461-478 | Pre-seeds package config | **DELETE ALL** — no equivalent; packages install with defaults |
| `dpkg-reconfigure keyboard-configuration` | 460 | Reconfigures keyboard | **DELETE** |
| `dpkg -i` | 690 | Installs .deb | Change to `rpm -i` with .rpm URL |
| `DEBIAN_FRONTEND=noninteractive` | 460 | Non-interactive apt | **DELETE** — `dnf -y` is already non-interactive |
| `timedatectl set-timezone Asia/Jakarta` | 202 | ✅ | **Same on RHEL** — no change |
| `ln -fs /usr/share/zoneinfo/Asia/Jakarta /etc/localtime` | 505 | ✅ | Same on RHEL (redundant with timedatectl, harmless) |
| `hostnamectl set-hostname` | 1003 | ✅ | Same on RHEL |
| `systemd-detect-virt` | 79 | ✅ | Same on RHEL |
| `/etc/default/` | 623 (dropbear) | Debian sysconfig dir | RHEL uses `/etc/sysconfig/` |
| `chronyd -q 'server ... iburst'` | 698 | ✅ | Same if chrony installed |
| `update-rc.d` | — | Not used in script | N/A |
| `add-apt-repository` | 237 (commented) | N/A | Use `dnf config-manager` if needed |

---

## 10. Code Sections Requiring Changes (by Line Number)

| Lines | Function/Section | Change Required |
|-------|------------------|-----------------|
| 2-4 | Header | `apt` → `dnf`, drop `wondershaper` (or GitHub install) |
| 54-65 | OS check | Add `almalinux`/`rocky`/`centos`/`rhel` ID matching, or new script |
| 186 | `chown www-data.www-data` | → `nginx:nginx` |
| 202-205 | `first_setup()` | `timedatectl` OK; **delete** `debconf-set-selections` lines |
| 207-229 | `first_setup()` OS branches | Add RHEL branch: `dnf install -y epel-release`, `dnf install -y haproxy` |
| 233-247 | `nginx_install()` | Add RHEL branch: `dnf install -y nginx` |
| 251-270 | `base_package()` | **Full rewrite** — see package table §2; `cron`→`cronie`, delete `debconf-utils`, `ntpdate`, `gnupg1`, etc.; `dist-upgrade`→`dnf upgrade` |
| 390 | `chown www-data.www-data` | → `nginx:nginx` |
| 403-410 | `install_xray()` | Add `mkdir -p /usr/local/etc/xray` + symlink (already in repo); Xray install script works on RHEL |
| 433-452 | `xray.service` creation | `User=www-data` → `User=nginx` |
| 456-478 | `ssh()` — PAM + keyboard | **Replace** `/etc/pam.d/common-password` logic with RHEL PAM; **delete** all `debconf-set-selections`/`dpkg-reconfigure` |
| 480-503 | `rc.local` setup | Convert to proper systemd or keep (deprecated but functional) |
| 616-618 | `ins_SSHD()` | `/etc/init.d/ssh` → `systemctl restart sshd` |
| 620-626 | `ins_dropbear()` | **Dropbear not available** — remove or compile from source; `/etc/default/dropbear` → N/A |
| 634-648 | `ins_vnstat()` | `apt` → `dnf`, `libsqlite3-dev` → `sqlite-devel`; `/etc/init.d/vnstat` → `systemctl` |
| 650-653 | `ins_openvpn()` | Check `limit/openvpn` installer for apt usage; service → `openvpn@server` |
| 665-684 | `ins_backup()` | `apt install rclone` → `dnf install -y rclone` (EPEL); `msmtp-mta` → `msmtp`; `bsd-mailx` → `mailx` |
| 686-704 | `ins_swab()` — gotop | `.deb` + `dpkg -i` → `.rpm` + `rpm -i` |
| 708-725 | `ins_Fail2ban()` | `apt -y install fail2ban` → `dnf install -y fail2ban` (EPEL) |
| 727-743 | `ins_epro()` — iptables | `netfilter-persistent save/reload` → `iptables-save > /etc/sysconfig/iptables` + `systemctl restart iptables`; `apt autoclean/autoremove` → `dnf clean/autoremove` |
| 750-762 | `ins_restart()` | All `/etc/init.d/` → `systemctl`; `ssh` → `sshd`; `cron` → `crond`; `netfilter-persistent` → `iptables` |
| 772-786 | `enable_services()` | `netfilter-persistent` → `iptables`; `cron` → `crond` |
| 897 | `service cron restart` | → `systemctl restart crond` |
| 908-915 | `rc.local` iptables | Keep or convert to firewalld/iptables-services |

---

## 11. Recommended Strategy

### Option A: New `rhel.sh` (RECOMMENDED)

Create a separate `rhel.sh` installer rather than adding RHEL branches to `premi.sh`.

**Pros:**
- `premi.sh` stays clean for Debian/Ubuntu.
- RHEL-specific setup (EPEL, SELinux, firewalld/iptables-services, `nginx` user) is contained.
- Easier to test and maintain independently.
- The download-and-configure functions (`make_folder_xray`, `pasang_domain`, `pasang_ssl`, `menu`, `profile`) can be copied verbatim — only the package-install and system-setup functions need rewriting.

**Cons:**
- Some duplication (~30% shared logic).

**Structure of `rhel.sh`:**
```
rhel.sh
├── Header (dnf makecache, epel-release)
├── OS check (almalinux|rocky|centos|rhel, versions 8/9)
├── first_setup_rhel()    # EPEL, haproxy, SELinux disable, firewall
├── nginx_install_rhel()  # dnf install nginx
├── base_package_rhel()   # Full RHEL package list
├── make_folder_xray()    # COPY AS-IS (only chown changes)
├── pasang_domain()       # COPY AS-IS
├── pasang_ssl()          # COPY AS-IS (acme.sh works on RHEL)
├── install_xray()        # COPY AS-IS (with symlink fix already in repo)
├── ssh()                 # REWRITE PAM section, drop debconf
├── ins_SSHD()            # systemctl sshd, no init.d
├── ins_dropbear()        # SKIP or compile from source
├── ins_vnstat()          # dnf version
├── ins_openvpn()         # Check limit/openvpn for apt usage
├── ins_backup()          # dnf packages
├── ins_swab()            # rpm instead of dpkg
├── ins_Fail2ban()        # dnf install fail2ban (EPEL)
├── ins_epro()            # iptables-services instead of netfilter-persistent
├── ins_restart()         # systemctl only
├── menu()                # COPY AS-IS
├── profile()             # COPY AS-IS (crond instead of cron)
└── enable_services()     # systemctl only
```

### Option B: Modify `premi.sh` with RHEL Branches (NOT RECOMMENDED)

Would require adding `elif [[ $ID == "almalinux" || ... ]]` to every OS-specific function (`first_setup`, `nginx_install`, `base_package`, etc.). The script would grow from 1014 to ~1400+ lines and become very hard to maintain.

### Testing Plan for `rhel.sh`

1. AlmaLinux 9 minimal (most common RHEL clone).
2. Verify: `dnf install -y epel-release` → all packages install.
3. Verify each service starts: `nginx`, `haproxy`, `xray`, `sshd`, `openvpn@server`, `fail2ban`, `vnstat`, `crond`, `ws`.
4. Verify SELinux doesn't block (start with disabled, then test enforcing with booleans).
5. Test DarkTunnel SSH WS flow end-to-end (the primary use case).
6. Then test Rocky 9 and AlmaLinux 8 (HAProxy 1.8 — verify config compatibility).

---

## 12. Files Beyond premi.sh That Need RHEL Attention

The `limit/` directory contains configs and helper scripts. These need checking:

| File | Issue |
|------|-------|
| `limit/haproxy.cfg` | ✅ Already portable (no `bind-process` after prior fix) |
| `limit/xray.conf` (nginx) | ✅ Portable, but references `www-data`? Check `proxy_set_header` — no user refs. OK. |
| `limit/nginx.conf` | Check for `user www-data;` directive → change to `user nginx;` |
| `limit/sshd` (sshd_config) | ✅ Portable (OpenSSH config is cross-platform) |
| `limit/dropbear.conf` | Dropbear config — skip if dropping Dropbear |
| `limit/openvpn` (installer) | **Must check** — likely contains `apt` commands |
| `limit/bbr.sh` | **Must check** — BBR sysctl config; check for `apt` or Debian paths |
| `limit/limit.sh` | **Must check** — likely contains `apt` commands |
| `limit/menu.zip` (menu scripts) | **Must check** — `addssh`, `addws`, etc. may use `apt` or Debian paths |
| `limit/ws.service` | ✅ Systemd unit — portable |
| `limit/runn.service` | ✅ Systemd unit — portable |
| `limit/udp-mini*` | Binary + systemd units — check binary is statically linked or has RHEL build |
| `kyt.sh` | **Must check** — likely contains `apt`/`pip` commands |
| `update.sh` | **Must check** — likely contains `apt` commands |

---

## Appendix: Quick Reference — RHEL 8/9 Package Sources

| Repo | How to Enable | Provides |
|------|---------------|----------|
| BaseOS | Pre-enabled | `curl`, `wget`, `cronie`, `sudo`, `openssl`, `firewalld`, `chrony`, `net-tools`, `bind-utils` |
| AppStream | Pre-enabled | `nginx`, `haproxy`, `python3`, `git`, `socat`, `jq`, `ruby`, `gcc`, `iptables-services` |
| EPEL | `dnf install -y epel-release` | `fail2ban`, `vnstat`, `rclone`, `openvpn`, `easy-rsa`, `htop`, `figlet`, `pwgen`, `p7zip`, `screen` |
| Not available | — | `dropbear`, `wondershaper` (use GitHub), `shc`, `gnupg1`, `ntpdate`, `iptables-persistent`, `netfilter-persistent`, `debconf-utils` |

---

*Report generated by static analysis of `premi.sh`. All line numbers refer to `~/workspace/alround-vpn/premi.sh` (1014 lines).*
