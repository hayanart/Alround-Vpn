# RHEL Porting Notes: `rhel.sh` (Alround-Vpn)

**Source:** `premi.sh` (1014 lines, Ubuntu/Debian) → **`rhel.sh`** (991 lines)
**Target:** AlmaLinux 8/9, Rocky Linux 8/9, CentOS Stream 8/9
**Date:** 2026-10-20
**Status:** Static port complete, `bash -n` syntax OK. **Not yet tested on a live RHEL VM.**

Install command (once repo is created):
`wget -q https://raw.githubusercontent.com/hayanart/Alround-Vpn/main/rhel.sh && chmod +x rhel.sh && ./rhel.sh`

---

## 1. Package Manager (`apt` → `dnf`)

All `apt`/`apt-get` invocations replaced with `dnf` equivalents:
- `apt update -y` → `dnf makecache -y`
- `apt upgrade -y` + `apt dist-upgrade -y` → `dnf upgrade -y` (dnf covers both)
- `apt install -y` → `dnf install -y`
- `apt-get clean all` / `apt autoclean` → `dnf clean all`
- `apt-get autoremove` / `apt autoremove` → `dnf autoremove -y`
- `apt-get install --no-install-recommends software-properties-common` → **deleted** (replaced by `dnf-plugins-core`, installed at header)

## 2. Repository Setup (new block at header)

```bash
dnf install -y epel-release
dnf install -y dnf-plugins-core
dnf makecache -y
```
EPEL is required for: `fail2ban`, `vnstat`, `rclone`, `openvpn`, `easy-rsa`, `htop`, `figlet`, `pwgen`, `p7zip`, `screen`.

## 3. Package Name Mapping (Debian → RHEL)

| Debian | RHEL | Notes |
|--------|------|-------|
| `cron` | `cronie` | Service is `crond` |
| `netcat` | `nmap-ncat` | Provides `nc` |
| `libsqlite3-dev` | `sqlite-devel` | |
| `libssl-dev` | `openssl-devel` | |
| `zlib1g-dev` | `zlib-devel` | |
| `libevent-dev` | `libevent-devel` | |
| `libnss3-dev` | `nss-devel` | |
| `libnss3-tools` | `nss-tools` | |
| `libnspr4-dev` | `nspr-devel` | |
| `libpam0g-dev` | `pam-devel` | |
| `libcap-ng-dev` | `libcap-ng-devel` | |
| `libselinux1-dev` | `libselinux-devel` | |
| `libxml-parser-perl` | `perl-XML-Parser` | |
| `pkg-config` | `pkgconf-pkg-config` | |
| `build-essential` | `gcc gcc-c++ make` | |
| `g++` | `gcc-c++` | |
| `p7zip-full` | `p7zip` | EPEL |
| `xz-utils` | `xz` | |
| `dnsutils` | `bind-utils` | Provides `dig` |
| `msmtp-mta` | `msmtp` | |
| `bsd-mailx` | `mailx` | |
| `iptables-persistent` | `iptables-services` | |
| `gnupg` / `gnupg1` | `gnupg2` only | `gnupg1` dropped |
| `lsb-release` | `redhat-lsb-core` | |
| `libc6` | `glibc` | |

**Deleted (not available on RHEL):** `ntpdate` (use chrony), `shc`, `dirmngr` (in gnupg2), `apt-transport-https`, `software-properties-common`, `debconf-utils`, `netfilter-persistent`, `wondershaper` (apt line; GitHub install in `ins_backup` kept).

## 4. Deleted Debian-Only Blocks

- `debconf-set-selections` (all): keyboard-configuration (18 lines), iptables-persistent autosave (4 lines)
- `DEBIAN_FRONTEND=noninteractive dpkg-reconfigure keyboard-configuration`
- `wget -O /etc/pam.d/common-password` — RHEL uses `/etc/pam.d/system-auth`; Debian PAM config skipped to avoid breaking auth
- `dpkg -i /tmp/gotop.deb` → `rpm -i /tmp/gotop.rpm` (downloads `.rpm` URL)

## 5. Dropbear: SKIPPED Entirely

Dropbear is not in RHEL BaseOS/AppStream/EPEL. `ins_dropbear()` is now a stub that prints a skip notice; the call is commented out in `instal()`. **OpenSSH (`sshd`) is used on all SSH ports instead.** The Dropbear banner `sed` in `ins_Fail2ban()` was also removed.

## 6. Service Name Changes

| Debian | RHEL |
|--------|------|
| `ssh` | `sshd` |
| `cron` | `crond` |
| `netfilter-persistent` | `iptables` (from `iptables-services`) |
| `openvpn` | `openvpn@server` (templated unit) |

All `/etc/init.d/<service>` calls replaced with `systemctl`. All `systemctl enable --now` lines updated.

## 7. User/Group: `www-data` → `nginx`

4 places changed:
- `chown nginx:nginx /var/log/xray` (was `www-data.www-data`)
- `chown nginx:nginx /run/xray` (was `www-data.www-data $domainSock_dir`)
- `User=nginx` in `xray.service` (was `User=www-data`)
- `chown -R root:root /etc/msmtprc` (was `www-data:www-data`)

## 8. SELinux Handling

Added at header (v1 = permissive):
```bash
setenforce 0 || true
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```
Documented future path: enforcing + `httpd_can_network_connect` and `haproxy_connect_any` booleans.

## 9. Firewall Handling

- `firewalld` is **NOT removed** (only disabled): `systemctl disable --now firewalld`
- `iptables-services` installed and enabled instead (faithful port — script manages raw iptables rules for BitTorrent blocking and SlowDNS NAT)
- Rules saved to `/etc/sysconfig/iptables` (was `/etc/iptables.up.rules`)
- `netfilter-persistent save/reload` → `systemctl restart iptables`

## 10. OS Detection

Replaced ubuntu/debian check with:
```bash
OS_ID=$( ... )  # from /etc/os-release
if [[ "$OS_ID" == "almalinux" || "$OS_ID" == "rocky" || "$OS_ID" == "centos" || "$OS_ID" == "rhel" ]]; then ...
```
Exits with error listing supported OS on mismatch.

## 11. Kept As-Is (Portable)

- `pasang_domain()` — interactive domain selection (1/2/random), prompts unchanged
- `pasang_ssl()` — acme.sh works on RHEL; `lsof` available
- `make_folder_xray()` — pure mkdir/touch
- `install_xray()` — Xray install script is cross-platform; config symlink (`/usr/local/etc/xray/config.json` → `/etc/xray/config.json`) already present from prior fix
- `udp_mini()`, `ssh_slow()`, `menu()`, `profile()` — download/config logic portable (systemd units identical)
- `restart_system()` — Telegram notify; `curl` available
- Final `read -p ... For Reboot` + `reboot` — unchanged
- `REPO` updated to `https://raw.githubusercontent.com/hayanart/Alround-Vpn/main/`
- haproxy.cfg: no `bind-process` (already fixed in repo); nginx `xray.conf` case-insensitive WebSocket check (already in repo)

## 12. Known Limitations / TODO Before Live Test

1. **Not tested on live RHEL** — static port only. Needs AlmaLinux 9 VM test.
2. **`limit/openvpn` installer** — not audited for `apt` usage; may need its own port.
3. **`limit/bbr.sh`** — not audited; BBR sysctl is kernel-generic but check for Debian paths.
4. **`limit/menu.zip` scripts** — `addssh` etc. may use Debian paths; menu was ported for Ubuntu/Debian, RHEL testing needed.
5. **`kyt.sh`** — pip install flow; needs `--break-system-packages` on RHEL 9 (same as Debian 12 fix).
6. **SlowDNS** — `limit/nameserver` missing from repo (pre-existing issue, not RHEL-specific).
7. **HAProxy versions** — RHEL 8 ships 1.8, RHEL 9 ships 2.4. Config avoids `bind-process` so should work on both, but 1.8 lacks some newer features; verify.
8. **PAM** — Debian password-quality config skipped; RHEL default PAM used. Password login still enabled via `sshd_config`.
9. **`speedtest-cli`** — kept in package list; assumed available in EPEL. If missing, remove from list.
10. **SELinux permissive** — acceptable for v1 personal VPN; revisit for production.

## 13. Change Summary (28 replacements)

Header (dnf/EPEL/SELinux/firewall), OS check, REPO URL, 4× www-data→nginx/root, first_setup, nginx_install, base_package (full list), ssh() PAM+keyboard, ins_SSHD, ins_dropbear (stub), ins_vnstat, ins_openvpn, ins_backup, ins_swab (rpm), ins_Fail2ban, ins_epro, ins_restart, profile, enable_services, instal (dropbear call removed).
