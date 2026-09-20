# Jarkom-Modul-1-2026-K-20

Laporan konfigurasi jaringan **The Wired** menggunakan GNS3, dengan Router **Lain** dan lima Entitas (**Alice, Mika, Chisa, Knights, Eiri**) berbasis node **Debian (debinet)**.

Prefix IP yang digunakan: `192.231.x.x`

| Switch/Gateway | Subnet | Entitas |
|---|---|---|
| Switch 1 (eth1) | 192.231.1.0/24 | Alice, Mika |
| Switch 2 (eth2) | 192.231.2.0/24 | Chisa |
| Switch 3 (eth3) | 192.231.3.0/24 | Knights, Eiri |
| eth0 (Lain) | DHCP | Internet Publik (NAT GNS3) |

---

## 1. Topologi

![topologi](images/topologi.png)

### Konfigurasi IP (`/etc/network/interfaces`)

**Router Lain**
```bash
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.231.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.231.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.231.3.1
    netmask 255.255.255.0
```

**Alice**
```bash
auto eth0
iface eth0 inet static
    address 192.231.1.2
    netmask 255.255.255.0
    gateway 192.231.1.1
```

**Mika**
```bash
auto eth0
iface eth0 inet static
    address 192.231.1.3
    netmask 255.255.255.0
    gateway 192.231.1.1
```

**Chisa**
```bash
auto eth0
iface eth0 inet static
    address 192.231.2.2
    netmask 255.255.255.0
    gateway 192.231.2.1
```

**Knights**
```bash
auto eth0
iface eth0 inet static
    address 192.231.3.2
    netmask 255.255.255.0
    gateway 192.231.3.1
```

**Eiri**
```bash
auto eth0
iface eth0 inet static
    address 192.231.3.3
    netmask 255.255.255.0
    gateway 192.231.3.1
```

Aktifkan interface di setiap node:
```bash
systemctl restart networking
# atau
ifup -a
```

**Verifikasi:** jalankan `ip -br a` di setiap node, pastikan IP sudah terpasang sesuai tabel di atas.

---

## 2. Menyambungkan Router Lain ke Internet (NAT/DHCP di eth0)

Aktifkan IP forwarding secara permanen di Router Lain:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Tambahkan aturan NAT Masquerade agar seluruh trafik keluar lewat eth0 (interface publik):
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth+ -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth+ -o eth0 -j ACCEPT
```

**Verifikasi:** dari Router Lain jalankan `ping 8.8.8.8`, harus dapat balasan (reply).

---

## 3. Routing Antar-Entitas (Switch 1, 2, 3 Saling Terhubung)

Karena semua client sudah mengarahkan **default gateway** ke interface Router Lain (lihat konfigurasi di poin 1), dan IP forwarding di Router Lain sudah aktif (poin 2), maka secara otomatis seluruh Entitas di bawah Switch 1, 2, dan 3 sudah bisa saling berkomunikasi melalui Router Lain sebagai penghubung antar-subnet.

**Verifikasi antar subnet, contoh dari Alice ke Chisa dan Knights:**
```bash
ping -c 4 192.231.2.2
ping -c 4 192.231.3.2
```

Jika hasilnya `0% packet loss`, berarti routing antar Switch sudah berhasil.

---

## 4. Firewall/Iptables (NAT Masquerade) & DNS Resolver Mandiri

Rule NAT Masquerade sudah dibuat di poin 2 (`iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`), sehingga seluruh Client di Switch 1/2/3 otomatis bisa keluar ke internet melalui Router Lain.

Tambahkan DNS resolver di **setiap Client** (Alice, Mika, Chisa, Knights, Eiri):
```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

**Verifikasi mandiri dari tiap Client:**
```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

Jika keduanya berhasil (dapat balasan/reply), berarti Client sudah bisa mengakses internet secara mandiri.

---

## 5. Persistensi Konfigurasi & Script Verifikasi Reboot

Buat script `/root/cek_status.sh` di **Router Lain**:
```bash
cat << 'EOF' > /root/cek_status.sh
#!/bin/bash
echo "========================================="
echo "   RINGKASAN STATUS INTERFACE & NAT     "
echo "========================================="
echo ""
echo "[+] Status Interface Jaringan:"
ip -br a
echo ""
echo "[+] Status Tabel NAT (Iptables):"
iptables -t nat -L -v -n
echo "========================================="
EOF
```

Beri izin eksekusi:
```bash
chmod +x /root/cek_status.sh
```

Jalankan untuk pengetesan awal:
```bash
/root/cek_status.sh
```

**Verifikasi persistensi:** lakukan `reboot` pada Router Lain, tunggu sampai menyala kembali, lalu jalankan ulang `/root/cek_status.sh`. Pastikan IP interface (`ip -br a`) dan rule NAT (`iptables -t nat -L -v -n`) masih sama seperti sebelum reboot — artinya konfigurasi tidak hilang.

![no 5](images/no6.png)

---

## 6. Packet Sniffing (DNS/ICMP) di Node Mika

Di **Console Mika**, buat script traffic generator, beri izin eksekusi, lalu jalankan:
```bash
cat << 'EOF' > /root/traffic_generator.sh
#!/bin/bash
# ============================================
# Traffic Generator — Protocol 7 Network
# Serial Experiments Lain — Modul 1 Jarkom 2026
# Jalankan di node MIKA untuk generate traffic DNS & ICMP
# ============================================

echo "============================================"
echo "  Protocol 7 Traffic Generator v2026"
echo "  Node: Mika Iwakura"
echo "============================================"
echo "[*] Generating DNS & ICMP traffic..."

# ICMP Traffic
ping -c 5 8.8.8.8 &
ping -c 5 1.1.1.1 &
ping -c 3 its.ac.id &

# DNS Queries
nslookup google.com 8.8.8.8 &
nslookup its.ac.id 8.8.8.8 &
nslookup github.com 1.1.1.1 &
dig @8.8.8.8 example.com A &
dig @1.1.1.1 cloudflare.com AAAA &

wait
echo "[*] Traffic generation complete."
echo "[*] Check Wireshark for captured packets."
EOF

chmod +x /root/traffic_generator.sh
/root/traffic_generator.sh
```
> Perlu `dnsutils` terpasang agar `nslookup`/`dig` tersedia: `apt update && apt install dnsutils -y`

Di **GNS3**, klik kanan pada kabel/interface yang terhubung ke node Mika → pilih **Start capture** untuk membuka Wireshark.

Pada kolom display filter Wireshark (hijau), masukkan:
```
dns || icmp
```

**Verifikasi:** amati daftar paket yang lolos filter — akan terlihat paket `Standard query`/`Standard query response` (DNS) dan `Echo (ping) request`/`Echo (ping) reply` (ICMP). Ambil screenshot hasil filter beserta ringkasan paketnya.

![dnsicmp](images/dnsicmp.png)

---

## 7. FTP Server di Node Chisa (`/var/wired/data`)

### 7.1 Install vsftpd & buat user

Di **Console Chisa**:
```bash
apt update && apt install vsftpd -y
useradd -m alice && echo "alice:123" | chpasswd
useradd -m mika && echo "mika:123" | chpasswd
useradd -m eiri && echo "eiri:123" | chpasswd
```

### 7.2 Buat shared folder

```bash
mkdir -p /var/wired/data
chown -R ftp:ftp /var/wired/data
chmod 777 /var/wired/data
```
> **Penting:** pakai `777` (bukan `755`), karena `alice` bukan owner folder ini (yang owner-nya `ftp`). Kalau cuma `755`, `alice` akan kena error `553 Could not create file` saat upload walau `write_enable=YES` sudah diset — permission folder di level Linux tetap menang duluan sebelum aturan vsftpd dicek. Mika tetap aman terblokir lewat `write_enable=NO` di vsftpd (bukan lewat permission folder), dan Eiri tetap diblokir total lewat `userlist_deny` sebelum sempat menyentuh folder ini.

### 7.3 Konfigurasi `/etc/vsftpd.conf`

```bash
nano /etc/vsftpd.conf
```

Pastikan baris-baris berikut ada/disesuaikan:
```
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
local_root=/var/wired/data
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=50000
user_config_dir=/etc/vsftpd_user_conf
userlist_enable=YES
userlist_file=/etc/vsftpd.user_list
userlist_deny=YES
```
Simpan dengan `Ctrl+O`, Enter, lalu keluar dengan `Ctrl+X`.

### 7.4 Hak akses per-user (Alice R/W, Mika Read-only, Eiri blacklist)

```bash
mkdir -p /etc/vsftpd_user_conf

# Alice -> Read & Write
echo "write_enable=YES" > /etc/vsftpd_user_conf/alice

# Mika -> Read-only
echo "write_enable=NO" > /etc/vsftpd_user_conf/mika

# Eiri -> Blacklist (tidak boleh login sama sekali)
echo "eiri" >> /etc/vsftpd.user_list
```

### 7.5 Jalankan ulang service vsftpd

```bash
/etc/init.d/vsftpd restart
```

**Verifikasi service aktif:**
```bash
ss -tuln | grep :21
```
Port 21 harus berstatus `LISTEN`.

### 7.6 Pembuktian: Alice bisa upload (`signal_alice.txt`)

Di **Console Alice**, buat file yang akan diupload:
```bash
echo "Ini pesan dari Alice" > signal_alice.txt
```

Login FTP ke Chisa lalu upload:
```bash
ftp 192.231.2.2
# user: alice
# password: 123
put signal_alice.txt
exit
```

**Verifikasi** di Console Chisa:
```bash
ls -l /var/wired/data
```
File `signal_alice.txt` harus muncul di dalam folder tersebut.

### 7.7 Pembuktian: Mika hanya read-only (upload ditolak)

Di **Console Mika**, coba login dan upload file apapun:
```bash
ftp 192.231.2.2
# user: mika
# password: 123
put test_mika.txt
```
Hasil yang diharapkan: server membalas `550 Permission denied` — bukti bahwa `write_enable=NO` untuk Mika sudah berjalan.

### 7.8 Pembuktian: Eiri ditolak login (blacklist)

Di **Console Eiri**, coba login FTP:
```bash
ftp 192.231.2.2
# user: eiri
# password: 123
```
Hasil yang diharapkan: koneksi langsung ditolak dengan pesan `530 Permission denied` / login failed — bukti bahwa Eiri sudah masuk `userlist_deny` dan tidak bisa mengakses FTP sama sekali.

![ftp](images/ftp.png)

---

## 8. Knights Upload Laporan Intelijen ke FTP Chisa (via akun `alice`)

**Soal:** Knights melakukan koneksi FTP client ke Chisa memakai akun `alice`, upload [file laporan intelijen ini](https://drive.google.com/drive/folders/1tvZpueSH9E3GWwXM6KNnM64Y5wNoIAYP?usp=sharing), lalu analisis di Wireshark: perintah `STOR`, kode status `226`, dan port data TCP yang dinegosiasikan mode PASV.

### 8.1 Ambil file dari link Drive ke node Knights

Link soal berupa **folder** Google Drive, jadi tidak bisa langsung di-`wget`. Dua cara:

**Cara A — copy-paste manual (paling gampang, tanpa perlu internet penuh di Knights):**
1. Buka link folder itu di browser laptop kamu, download file di dalamnya, buka dengan text editor, copy semua isinya.
2. Di **Console Knights**:
```bash
cat << 'EOF' > knights_report.txt
[PASTE ISI FILE YANG SUDAH KAMU DOWNLOAD DI SINI]
EOF
```

**Cara B — pakai `gdown` (kalau file-nya tunggal di dalam folder dan Knights punya akses internet):**
```bash
apt update && apt install python3-pip -y
pip3 install gdown --break-system-packages
gdown "https://drive.google.com/uc?id=FILE_ID" -O knights_report.txt
```
> Ganti `FILE_ID` dengan ID file asli (klik kanan file di Drive → **Get link**, ambil bagian setelah `/d/` atau `id=` di URL-nya).

**Verifikasi:**
```bash
ls -l knights_report.txt
cat knights_report.txt
```

### 8.2 Mulai capture Wireshark

Di **GNS3**, klik kanan kabel yang terhubung ke node Knights (atau Chisa) → **Start capture**. Lakukan **sebelum** upload dijalankan.

### 8.3 Login FTP dari Knights memakai akun `alice`

Knights tidak punya akun FTP sendiri di Chisa, jadi ia "menumpang" kredensial `alice` (yang punya hak read/write) untuk upload:
```bash
lftp -u alice 192.231.2.2
# password: 123
put knights_report.txt
exit
```

### 8.4 Analisis di Wireshark (jawaban laporan)

Stop capture, filter `ftp`, lalu temukan tiga hal berikut:

| Yang dicari | Info di Wireshark |
|---|---|
| Perintah upload | `Request: STOR knights_report.txt` |
| Status sukses server | `Response: 226 Transfer complete` |
| Port data PASV | `Response: 227 Entering Passive Mode (192,231,2,2,X,Y)` → port = `(X × 256) + Y` |

---

## 9. Mika Mengakses Dokumen Protokol Tujuh (Uji Read-Only FTP)

**Soal:** unduh dokumen dari [link Google Drive](https://drive.google.com/drive/folders/1S3hG0dnZBTkCta4uILWwKVc6dSYYGRJ6?usp=sharing) ke FTP Server Chisa, lalu dari node Mika unduh file itu pakai akun `mika`, dan buktikan pembatasan read-only (upload ditolak `550`).

### 9.1 Ambil file asli dari link Drive ke server Chisa

Link di soal adalah **folder** Google Drive, jadi `wget` langsung ke link itu tidak akan berhasil (Google Drive folder bukan file langsung). Cara paling gampang:

1. Buka link folder itu di **browser laptop/PC kamu** (bukan di console GNS3), lalu buka file di dalamnya dan klik **Download** untuk menyimpannya ke laptop.
2. Cek nama & format filenya (biasanya `.txt` atau `.pdf`).
3. Salin isinya ke node Chisa. Karena node GNS3 tidak punya akses ke file lokal laptop secara langsung, cara termudah:
   - Buka file yang sudah kamu download tadi pakai Notepad/text editor di laptop, **copy semua isinya**.
   - Di **Console Chisa**, jalankan `cat << 'EOF' > /var/wired/data/protocol7_manifesto.txt`, lalu **paste** isi file tadi, tutup dengan `EOF`:
```bash
cat << 'EOF' > /var/wired/data/protocol7_manifesto.txt
[PASTE ISI FILE YANG SUDAH KAMU DOWNLOAD DI SINI]
EOF
```

> **Alternatif kalau Chisa punya akses internet penuh:** kalau filenya kecil dan berupa file tunggal (bukan folder), klik kanan file itu di Drive → **Get link** → copy `FILE_ID` dari URL-nya (bagian setelah `/d/` atau `id=`), lalu di Chisa:
> ```bash
> apt update && apt install python3-pip -y
> pip3 install gdown --break-system-packages
> gdown "https://drive.google.com/uc?id=FILE_ID" -O /var/wired/data/protocol7_manifesto.txt
> ```

**Verifikasi file sudah lengkap (tidak terpotong):**
```bash
ls -l /var/wired/data/
cat /var/wired/data/protocol7_manifesto.txt
```

### 9.2 Siapkan file dummy untuk uji upload (Console Mika)

```bash
echo "Ini file percobaan upload dari Mika" > file_mika.txt
```

### 9.3 Login FTP dari Mika, buktikan Read vs Write

```bash
lftp -u mika 192.231.2.2
# password: 123
get protocol7_manifesto.txt   # berhasil -> bukti hak READ
put file_mika.txt             # ditolak  -> bukti TIDAK ADA hak WRITE
exit
```

**Hasil yang diharapkan** saat `put`:
```
put: Access failed: 550 Permission denied. (file_mika.txt)
```
Screenshot pesan `550 Permission denied` ini menjadi bukti `write_enable=NO` untuk Mika sudah bekerja.

---

## 10. Uji Ketahanan Koneksi (Ping Latency) Knights → Chisa

**Soal:** kirim ping dari Knights ke Chisa dengan payload 128 byte, interval 0.3 detik, sebanyak 77 paket. Analisis ICMP Type/Code di Wireshark, serta packet loss dan RTT (min/avg/max).

### 10.1 Mulai capture Wireshark

Di GNS3, klik kanan kabel yang terhubung ke node Knights atau Chisa → **Start capture**. Lakukan **sebelum** menjalankan ping.

### 10.2 Jalankan ping dengan parameter sesuai soal

Di **Console Knights**:
```bash
ping -c 77 -s 128 -i 0.3 192.231.2.2
```
- `-c 77` → kirim 77 paket
- `-s 128` → payload data 128 byte (di luar header ICMP/IP)
- `-i 0.3` → jeda 0.3 detik antar paket

Tunggu sampai selesai. Output ping akan langsung menampilkan ringkasan di baris terakhir:
```
77 packets transmitted, 77 received, 0% packet loss, time XXXXms
rtt min/avg/max/mdev = X.XXX/X.XXX/X.XXX/X.XXX ms
```
Catat baris ini untuk laporan (packet loss & RTT min/avg/max sudah langsung dihitung otomatis oleh `ping`).

### 10.3 Analisis ICMP Type/Code di Wireshark

Stop capture, lalu filter:
```
icmp
```

| Arah paket | ICMP Type | ICMP Code | Keterangan |
|---|---|---|---|
| Knights → Chisa (`Echo (ping) request`) | Type 8 | Code 0 | Permintaan echo dari pengirim |
| Chisa → Knights (`Echo (ping) reply`) | Type 0 | Code 0 | Balasan echo dari penerima, menandakan host hidup & reachable |

Klik salah satu paket `Echo (ping) request` dan `Echo (ping) reply`, lalu screenshot bagian **Internet Control Message Protocol** di panel detail Wireshark untuk menunjukkan nilai `Type` dan `Code` di atas sebagai bukti.

### 10.4 Ringkasan analisis untuk laporan

- **Packet loss 0%** menandakan tidak ada paket yang hilang di jalur Knights → Chisa, berarti routing dan link antar Switch 3 dan Switch 2 (lewat Router Lain) stabil.
- **RTT (Round Trip Time)** menunjukkan waktu tempuh pulang-pergi satu paket ICMP; nilai `avg` yang kecil dan konsisten (selisih `min` dan `max` tidak jauh) menandakan latensi jaringan stabil, sedangkan `mdev` (mean deviation) yang besar menandakan jitter/ketidakstabilan.

---

## 11. Buktikan Kelemahan Protokol Telnet (`phantom_user` login dari Eiri)

**Soal:** buktikan kelemahan Telnet dengan akun `phantom_user`/`wired_ghost` di Chisa, login dari Eiri, tangkap sesi di Wireshark, tunjukkan kredensial plaintext lewat *Follow TCP Stream*, dan jelaskan kenapa tiap karakter terkirim di paket TCP terpisah.

### 10.1 Install Telnet server & buat akun phantom (Console Chisa)
```bash
apt update && apt install telnetd openbsd-inetd -y
useradd -m phantom_user && echo "phantom_user:wired_ghost" | chpasswd
```

**Kalau service tidak otomatis jalan** (umum terjadi di image debinet yang tanpa systemd), daftarkan manual ke `inetd` lalu jalankan:
```bash
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
pkill inetd
echo "telnet stream tcp nowait root /usr/sbin/telnetd telnetd" > /etc/inetd.conf
/usr/sbin/inetd
```

**Verifikasi port 23 terbuka:**
```bash
ss -tuln | grep :23
```
Harus muncul `*:23` atau `0.0.0.0:23`.

### 10.2 Mulai capture Wireshark di Eiri

GNS3 → klik kanan kabel node Eiri → **Start capture** → filter `telnet`. Lakukan ini **sebelum** login.

### 10.3 Login Telnet dari Eiri
```bash
apt update && apt install telnet -y
telnet 192.231.2.2
# login: phantom_user
# Password: wired_ghost
exit
```

### 10.4 Analisis di Wireshark

| Yang dibuktikan | Caranya |
|---|---|
| Kredensial plaintext | Klik kanan salah satu paket Telnet → **Follow → TCP Stream**. Username `phantom_user` & password `wired_ghost` akan terlihat jelas tanpa enkripsi. |
| Kenapa tiap karakter jadi paket TCP terpisah | Telnet beroperasi dalam mode **Character-at-a-time (Remote Echo)** — tiap tombol ditekan langsung dibungkus satu paket TCP dan dikirim real-time ke server, lalu server meng-*echo*-kan kembali karakter itu ke klien agar muncul di layar. Ini yang menyebabkan banyak paket kecil hanya untuk mengirim satu kata. |

Screenshot jendela *Follow TCP Stream* ini jadi bukti bahwa Telnet tidak aman — bandingkan nanti dengan hasil SSH di bagian 13 yang sudah terenkripsi.

---

## 12. Port Scanning dengan Netcat (Alice → Knights)

**Soal:** pindai port 22 (SSH) dan 80 (HTTP) yang harus terbuka, serta port rahasia 7777 yang harus tertutup, dari Alice ke Knights. Analisis perbedaan TCP flag `SYN-ACK` (terbuka) vs `RST-ACK` (tertutup) di Wireshark.

### 12.1 Nyalakan layanan SSH & HTTP di Knights

```bash
apt update && apt install openssh-server nginx -y
/etc/init.d/ssh start
/etc/init.d/nginx start
```

**Verifikasi:**
```bash
ss -tuln
```
Pastikan muncul `*:22` dan `*:80` yang `LISTEN`. Jangan nyalakan apa pun di port 7777.

### 12.2 Mulai capture Wireshark di Alice

Di GNS3, klik kanan kabel node Alice → **Start capture**. Di kolom display filter Wireshark, masukkan:
```
tcp.port in {22, 80, 7777}
```

### 12.3 Jalankan port scan dari Alice

```bash
apt update && apt install netcat-traditional -y
nc -zv 192.231.3.2 22 80 7777
```

**Hasil yang diharapkan:**
- Port 22 & 80 → `succeeded!` / `open`
- Port 7777 → `Connection refused` (tertutup)

### 12.4 Analisis TCP Flag di Wireshark

| Kondisi Port | Flag yang dikirim Alice | Flag balasan Knights | Arti |
|---|---|---|---|
| Terbuka (22, 80) | `[SYN]` | `[SYN, ACK]` | Server menyetujui, three-way handshake lanjut |
| Tertutup (7777) | `[SYN]` | `[RST, ACK]` | Tidak ada layanan yang listen, koneksi ditolak instan |

Ambil 2 screenshot: satu untuk `SYN-ACK` (port 22/80), satu untuk `RST-ACK` (port 7777).

---

## 13. SSH Public Key Authentication (Mika → Knights)

**Soal:** admin jarak jauh tanpa password. Install OpenSSH di Knights, buat key pair di Mika untuk user `mika_admin`, aktifkan `PasswordAuthentication no`, lalu analisis Protocol Version Exchange & Key Exchange di Wireshark.

### 13.1 Buat akun `mika_admin` di server (Knights)

```bash
useradd -m mika_admin && echo "mika_admin:123" | chpasswd
```

### 13.2 Generate SSH key pair di client (Mika)

```bash
useradd -m mika_admin
su - mika_admin
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```
Tunggu sampai muncul *randomart* (kotak-kotak) sebagai tanda key berhasil dibuat.

### 13.3 Kirim public key ke Knights

```bash
ssh-copy-id mika_admin@192.231.3.2
# ketik "yes" saat verifikasi fingerprint
# password sementara: 123
```
Berhasil jika muncul `Number of key(s) added: 1`.

### 13.4 Matikan login password di Knights

```bash
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
/etc/init.d/ssh restart
```

### 13.5 Capture Wireshark & buktikan login tanpa password

Di GNS3: klik kanan kabel node Mika → **Start capture** → filter `ssh`.

Di Console Mika:
```bash
ssh mika_admin@192.231.3.2
```
Harus langsung masuk **tanpa diminta password**. Ketik `exit`, lalu stop capture.

### 13.6 Analisis paket SSH di Wireshark

| Fase | Info paket di Wireshark | Penjelasan |
|---|---|---|
| Protocol Version Exchange | `Client: Protocol (SSH-2.0-OpenSSH...)` & `Server: Protocol (...)` | Mika & Knights saling memberi tahu versi SSH yang didukung |
| Key Exchange Init | `Key Exchange Init` | Diffie-Hellman: bertukar materi kriptografi publik untuk membentuk shared secret session key tanpa mengirim kunci rahasia lewat jaringan |
| Setelah KEX | `Encrypted packet` | Seluruh sisa komunikasi (termasuk autentikasi & data) dienkripsi AES, sehingga sniffer tidak bisa membaca isinya |

**Jawaban analisis (kenapa kredensial tidak terlihat, beda dari Telnet):**
Begitu fase Key Exchange selesai, Mika dan Knights sudah sepakat memakai kunci enkripsi simetris. Sejak titik itu seluruh paket — termasuk proses autentikasi berbasis key dan pertukaran data — tampil di Wireshark hanya sebagai `Encrypted packet`, berbeda dengan Telnet yang mengirim setiap karakter secara plaintext.

---

## File yang Perlu Disimpan di `/root` Tiap Node

Semua file di bawah ini disimpan di `/root` masing-masing node (kecuali disebutkan lain) supaya persisten dan tidak perlu diketik ulang saat GNS3 dibuka lagi.

### Router Lain
```bash
/root/cek_status.sh      # sudah dibuat di bagian 5 — cek IP + tabel NAT
```
`/etc/rc.local` (bukan di `/root`, tapi wajib untuk auto-start NAT/DHCP retry — lihat bagian sebelumnya di chat).

### Alice (`192.231.1.2`)
```bash
cat << 'EOF' > /root/signal_alice.txt
Ini pesan dari Alice
EOF

cat << 'EOF' > /root/cek_alice.sh
#!/bin/bash
ip -br a
cat /etc/resolv.conf
ping -c2 8.8.8.8
EOF
chmod +x /root/cek_alice.sh
```

### Mika (`192.231.1.3`)
```bash
cat << 'EOF' > /root/traffic_generator.sh
#!/bin/bash
echo "============================================"
echo "  Protocol 7 Traffic Generator v2026"
echo "  Node: Mika Iwakura"
echo "============================================"
echo "[*] Generating DNS & ICMP traffic..."

ping -c 5 8.8.8.8 &
ping -c 5 1.1.1.1 &
ping -c 3 its.ac.id &

nslookup google.com 8.8.8.8 &
nslookup its.ac.id 8.8.8.8 &
nslookup github.com 1.1.1.1 &
dig @8.8.8.8 example.com A &
dig @1.1.1.1 cloudflare.com AAAA &

wait
echo "[*] Traffic generation complete."
echo "[*] Check Wireshark for captured packets."
EOF
chmod +x /root/traffic_generator.sh

echo "Ini file percobaan upload dari Mika" > /root/file_mika.txt
```
> `id_rsa` / `id_rsa.pub` untuk SSH (bagian 13) otomatis tersimpan persisten di `/home/mika_admin/.ssh/`, bukan di `/root` — tidak perlu dipindah.

### Chisa (`192.231.2.2`)
Data FTP tetap di `/var/wired/data` (bukan `/root`), tapi simpan script setup-nya di `/root` sebagai cadangan kalau perlu setup ulang:
```bash
cat << 'EOF' > /root/setup_ftp.sh
#!/bin/bash
apt update && apt install vsftpd -y
useradd -m alice && echo "alice:123" | chpasswd
useradd -m mika && echo "mika:123" | chpasswd
useradd -m eiri && echo "eiri:123" | chpasswd
mkdir -p /var/wired/data
chown -R ftp:ftp /var/wired/data
chmod 777 /var/wired/data
mkdir -p /etc/vsftpd_user_conf
echo "write_enable=YES" > /etc/vsftpd_user_conf/alice
echo "write_enable=NO" > /etc/vsftpd_user_conf/mika
echo "eiri" >> /etc/vsftpd.user_list
/etc/init.d/vsftpd restart
EOF
chmod +x /root/setup_ftp.sh
```

Untuk demo Telnet (bagian 11), simpan juga script setup-nya:
```bash
cat << 'EOF' > /root/setup_telnet.sh
#!/bin/bash
apt update && apt install telnetd openbsd-inetd -y
useradd -m phantom_user && echo "phantom_user:wired_ghost" | chpasswd
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
pkill inetd
echo "telnet stream tcp nowait root /usr/sbin/telnetd telnetd" > /etc/inetd.conf
/usr/sbin/inetd
EOF
chmod +x /root/setup_telnet.sh
```
> File `protocol7_manifesto.txt` (bagian 9) disimpan di `/var/wired/data/`, bukan `/root`, karena harus bisa diakses lewat FTP.

### Knights (`192.231.3.2`)
```bash
# File laporan intelijen untuk soal 8 (isi asli diambil dari link Drive, lihat bagian 8.1)
cat << 'EOF' > /root/knights_report.txt
[PASTE ISI FILE DARI LINK DRIVE — lihat bagian 8.1]
EOF

cat << 'EOF' > /root/setup_services.sh
#!/bin/bash
apt update && apt install openssh-server nginx -y
/etc/init.d/ssh start
/etc/init.d/nginx start
EOF
chmod +x /root/setup_services.sh
```
> `authorized_keys` dari `mika_admin` (bagian 13) tersimpan otomatis di `/home/mika_admin/.ssh/`, bukan `/root`.

### Eiri (`192.231.3.3`)
Tidak ada file wajib di `/root` — perannya cuma membuktikan penolakan akses (FTP `530`/blacklist) dan sebagai client Telnet untuk demo kelemahan protokol (bagian 10, login `phantom_user`/`wired_ghost` ke Chisa). Cukup pastikan paket client terpasang:
```bash
apt update && apt install telnet -y
```
Screenshot hasil percobaan (gagal FTP / sukses Telnet + Follow TCP Stream) sudah cukup, tidak perlu file tambahan.

---

## Ringkasan Alur Pengerjaan

1. Setting IP tiap node sesuai tabel subnet → cek dengan `ip -br a`.
2. Aktifkan `ip_forward` + NAT Masquerade di Lain → cek `ping 8.8.8.8` dari Lain.
3. Pastikan gateway tiap Client mengarah ke Lain → cek ping antar subnet.
4. Set `resolv.conf` di semua Client → cek `ping 8.8.8.8` & `ping google.com`.
5. Buat `/root/cek_status.sh` di Lain → uji setelah `reboot`.
6. Jalankan traffic generator di Mika → capture Wireshark dengan filter `dns || icmp`.
7. Install & konfigurasi `vsftpd` di Chisa → buktikan Alice R/W, Mika read-only, Eiri diblokir.
8. Knights upload laporan intelijen ke FTP Chisa via akun `alice` (file dari link Drive) → cek `STOR`/`226`/PASV port di Wireshark.
9. Mika unduh dokumen Protokol Tujuh dari link Drive ke FTP Chisa → uji read-only (download OK, upload `550`).
10. Ping latency Knights → Chisa (`-c 77 -s 128 -i 0.3`) → catat ICMP Type/Code (8/0 request, 0/0 reply), packet loss & RTT.
11. Kelemahan Telnet `phantom_user`/`wired_ghost` dari Eiri → *Follow TCP Stream* + jelaskan character-at-a-time.
12. Port scan Netcat dari Alice ke Knights → bandingkan `SYN-ACK` (terbuka) vs `RST-ACK` (tertutup) di Wireshark.
13. Setup SSH key-based auth Mika → Knights (`PasswordAuthentication no`) → analisis Protocol Version Exchange, Key Exchange Init, dan paket terenkripsi di Wireshark.
