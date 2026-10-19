# PERUBAHAN — Porting wiz4rd ke Ubuntu 22/24 & Debian 11/12

Dikerjakan oleh Luna, 2026-10-19. Semua klaim paket/URL di bawah ini diverifikasi langsung
ke packages.ubuntu.com, packages.debian.org, dan HTTP status check.

## File yang diubah
- `premi.sh` (installer utama/Ubuntu)
- `debian.sh` (installer Debian)
- `kyt.sh`, `update.sh`, `limit/bbr.sh`
- `README.md` (daftar OS)

## 🔴 Perbaikan yang bikin install jalan

1. **Typo `apt install curls` → `apt install curl -y`** (premi.sh & debian.sh baris 5)
   Paket `curls` tidak pernah ada — gagal di semua OS.

2. **HAProxy Ubuntu: hapus PPA `vbernat/haproxy-2.0` + pin `haproxy=2.0.*`**
   PPA itu tidak punya build untuk jammy/noble → install gagal total di 22.04/24.04.
   Sekarang pakai haproxy bawaan distro (22.04→2.4, 24.04→2.8). Config `limit/haproxy.cfg`
   sudah dicek: tidak pakai directive yang dihapus di HAProxy 2.x.

3. **HAProxy Debian: hapus repo `haproxy.debian.net buster-backports-1.8` + pin `haproxy=1.8.*`**
   Build 1.8 itu butuh `libssl1.1` yang tidak ada di Debian 12 (bookworm pakai libssl3)
   → dependency gagal di Debian 12. Sekarang pakai haproxy bawaan distro (11→2.2, 12→2.6).

4. **Hapus `python` (Python 2) dari daftar ~80 paket** → diganti `python3`
   Paket `python` sudah dibuang total di Ubuntu 22.04+ dan Debian 11+. Satu nama paket
   yang tidak ketemu saja sudah cukup untuk menggagalkan SELURUH transaksi apt —
   ini baris paling merusak untuk porting.

5. **Hapus `libcurl4-nss-dev` dari daftar paket**
   Tidak tersedia di Ubuntu 24.04 (efeknya sama: meracuni seluruh transaksi apt).

6. **URL `limit.sh` yang mati (404) diperbaiki**
   - premi.sh: `hayanart/vip/...` (repo tidak ada) → `${REPO}limit/limit.sh` (repo sendiri)
   - debian.sh: `arifkenza09/vip/...` → `${REPO}limit/limit.sh` (repo sendiri)
   - Bonus temuan: `REPO` di debian.sh dulunya menunjuk ke `nishikazekazata/vip`
     yang SUDAH 404 — artinya SEMUA download config debian.sh gagal. Sekarang menunjuk
     ke `hayanart/wiz4rd` (terverifikasi HTTP 200).

7. **SlowDNS (`limit/nameserver`) tidak ada di repo (404)** → tidak lagi silent
   File-nya memang tidak ada di repo, jadi modul SlowDNS tidak pernah terinstall
   (sebelumnya gagal diam-diam). Sekarang script kasih peringatan jelas.
   → **Perlu kamu upload `limit/nameserver` ke repo kalau mau SlowDNS jalan.**

## 🟡 Perbaikan error & ketahanan

8. **Hapus baris `systemctl enable/restart chronyd`** — `chronyd` itu nama unit RHEL;
   di Debian/Ubuntu namanya `chrony` (baris chrony yang benar sudah ada).

9. **Hapus pemanggilan `password_default`** — fungsinya tidak pernah didefinisikan
   (sebelumnya cuma mencetak `command not found`).

10. **Xray & gotop: tahan terhadap rate-limit API GitHub**
    Pakai endpoint `/releases/latest` (1 request, bukan daftar panjang) + fallback:
    kalau API gagal, Xray install tanpa pin versi, gotop dilewati dengan peringatan
    (tidak lagi bikin URL rusak diam-diam).

11. **Fail2ban sekarang beneran di-install** — sebelumnya baris install-nya dikomen
    tapi script tetap memanggil `fail2ban` di bagian restart (error di semua OS).

12. **vnstat: build dari source (2.6) hanya kalau perlu**
    Distro baru sudah bawa vnstat ≥ 2.9 — build source 2.6 sebelumnya malah downgrade.
    Sekarang dicek dulu versinya, build cuma jalan di distro lama.

13. **kyt.sh: `pip3 install --break-system-packages`** — wajib di Debian 12/Ubuntu 24.04 (PEP 668).

14. **update.sh: URL `epro` diperbaiki** — `MONSTER/epro/epro` (404) → `MONSTER/limit/epro/epro` (200).

15. **limit/bbr.sh: `modprobe ip_conntrack` → `modprobe nf_conntrack`**
    (nama modul lama, gagal di kernel modern).

16. **Hilangkan `sudo` di dalam script** — script sudah wajib jalan sebagai root,
    `sudo` belum tentu terinstall saat `first_setup` berjalan (image minimal).

## ⚠️ Catatan / PR untuk kamu

- **SlowDNS**: upload file `limit/nameserver` (installer SlowDNS) ke repo, atau modul
  ini akan selalu dilewati dengan peringatan.
- **Notifikasi backup email (msmtp/Gmail)**: Gmail sudah tidak menerima password biasa
  untuk SMTP — perlu App Password atau fitur ini mati. Kredensialnya tetap tidak diubah.
- **Token bot Telegram & Cloudflare API key** masih hardcoded di repo public.
  Sangat disarankan: regenerate token via @BotFather dan pindahkan secret keluar repo.
- **Belum di-test di VPS beneran.** Analisa ini statis + verifikasi paket/URL.
  Disarankan: test di satu VPS percobaan (Ubuntu 22.04 atau Debian 12) sebelum dipakai produksi.
  Kalau mau, aku bisa siapkan VPS test — tinggal kamu konfirmasi dulu.
- `comp-lzo` di config OpenVPN: masih jalan di OpenVPN 2.6 (22.04/24.04, Debian 11/12)
  tapi mencetak warning. Bisa dibersihkan nanti, tidak mendesak.
