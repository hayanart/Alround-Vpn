# Alround-Vpn

All-round tunneling script — multi-OS support.

### INSTALL - Ubuntu / Debian
<pre><code>wget -q https://raw.githubusercontent.com/hayanart/Alround-Vpn/main/premi.sh && chmod +x premi.sh && ./premi.sh
</code></pre>

### INSTALL - AlmaLinux / Rocky Linux / CentOS
<pre><code>wget -q https://raw.githubusercontent.com/hayanart/Alround-Vpn/main/rhel.sh && chmod +x rhel.sh && ./rhel.sh
</code></pre>

### PERINTAH UPDATE
<pre><code>wget https://raw.githubusercontent.com/hayanart/Alround-Vpn/main/update.sh && chmod +x update.sh && ./update.sh</code></pre>

### TESTED ON OS
- UBUNTU 20.04 / 22.04 / 24.04
- DEBIAN 10 / 11 / 12
- ALMALINUX 8 / 9 (rhel.sh)
- ROCKY LINUX 8 / 9 (rhel.sh)
- CENTOS STREAM 8 / 9 (rhel.sh)

### CATATAN RHEL
- Dropbear tidak tersedia di RHEL — hanya OpenSSH
- SELinux di-set permissive otomatis
- Dokumentasi porting: RHEL_ANALYSIS.md, RHEL_PORTING_NOTES.md
