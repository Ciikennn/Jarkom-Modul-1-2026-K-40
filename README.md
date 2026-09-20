# Jarkom-Modul-1-2026-K-40

1. TOPLOPOGI
   ![topologi](images/topologi.png)

 konfigurasi
*route*
```
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
*alice&mika*
```
auto eth0
iface eth0 inet static
    address 192.231.1.2
    netmask 255.255.255.0
    gateway 192.231.1.1
```
*chisa*
```
auto eth0
iface eth0 inet static
    address 192.231.2.2
    netmask 255.255.255.0
    gateway 192.231.2.1
```
*Knights&eiri*
```
auto eth0
iface eth0 inet static
    address 192.231.3.2
    netmask 255.255.255.0
    gateway 192.231.3.1
```
2. Menyambungkan ke Sinyal
   ktifkan fitur penerusan paket (IP forwarding) agar router diizinkan melewatkan paket data antar-jaringan:
 ```
   echo 1 > /proc/sys/net/ipv4/ip_forward
```
```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth+ -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth+ -o eth0 -j ACCEPT
```
```
ip route add default via [IP_ROUTER_LAIN]
```
3&4. 
```
"nameserver 8.8.8.8" > /etc/resolv.conf
```
5.
buat skrip di lain  /root/cek_status.sh
```
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
kasih permission
```
chmod +x /root/cek_status.sh
```
jalankan
```
/root/cek_status.sh
```
![no 5](images/no6.png)

6.
di console mika
```
wget [MASUKKAN_LINK_URL_FILE_DI_SINI] -O traffic_generator.sh
chmod +x traffic_generator.sh
./traffic_generator.sh
```
klik kanan pada yang ingin di capture lalu jalankan whiteshark saat itu klik filternya
```
dns || icmp
```
![dnsicmp](images/dnsicmp.png)

7.

Di **Console Chisa**:
```bash
apt update && apt install vsftpd -y
useradd -m alice && echo "alice:123" | chpasswd
useradd -m mika && echo "mika:123" | chpasswd
useradd -m eiri && echo "eiri:123" | chpasswd
```


```bash
mkdir -p /var/wired/data
chown alice:alice /var/wired/data
chmod 775 /var/wired/data
```


```bash
cat > /etc/vsftpd.conf << 'EOF'
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
seccomp_sandbox=NO
userlist_enable=YES
userlist_deny=YES
userlist_file=/etc/vsftpd.userlist
pam_service_name=vsftpd
user_config_dir=/etc/vsftpd/user_conf
EOF
```


```bash
mkdir -p /etc/vsftpd/user_conf
echo "write_enable=YES" > /etc/vsftpd/user_conf/alice
echo "write_enable=NO" > /etc/vsftpd/user_conf/mika
```
```
echo "eiri" > /etc/vsftpd.userlist
```

### 7.5 Jalankan ulang service vsftpd

```
service vsftpd restart
```

**Verifikasi service aktif:**
```bash
ss -tuln | grep :21
```
Port 21 harus berstatus `LISTEN`.

### Pembuktian: Alice bisa upload (`signal_alice.txt`)

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

Di **Console Eiri**, coba login FTP:
```bash
ftp 192.231.2.2
# user: eiri
# password: 123
```
Hasil yang diharapkan: koneksi langsung ditolak dengan pesan `530 Permission denied` / login failed — bukti bahwa Eiri sudah masuk `userlist_deny` dan tidak bisa mengakses FTP sama sekali.

![ftp](images/no7.png)

8.





Cara A — copy-paste manual (paling gampang, tanpa perlu internet penuh di Knights):

Buka link folder itu di browser laptop kamu, download file di dalamnya, buka dengan text editor, copy semua isinya.
Di Console Knights:
```
cat << 'EOF' > knights_report.txt
==================================================
  KNIGHTS OF THE EASTERN CALCULUS — STATUS REPORT
  Protocol 7 Surveillance Network
  Classification: LEVEL 7 — EYES ONLY
==================================================

Date: [CLASSIFIED]
Agent: Knights Unit Alpha
Node: Switch 3 — Subnet 10.<PREFIX>.3.0/24

---

SUBJECT: Network Reconnaissance Report

The Wired has been successfully infiltrated through
Protocol 7 channels. Current observations:

1. Router "Lain" has been identified as the central
   gateway node connecting all three subnet segments.

2. Switch 1 (10.<PREFIX>.1.0/24) hosts Alice and Mika.
   Both nodes show standard traffic patterns.

3. Switch 2 (10.<PREFIX>.2.0/24) hosts Chisa alone.
   Isolated subnet — minimal cross-traffic observed.

4. Switch 3 (10.<PREFIX>.3.0/24) — our operational base.
   Knights and Eiri coexist on this segment.

RECOMMENDATION:
Continue monitoring FTP and Telnet sessions for
plaintext credential exposure. SSH tunnels remain
impenetrable without keylog access.

--- END OF REPORT ---
Knights of the Eastern Calculus
"Let's all love Lain."
EOF
```

```
ls -l knights_report.txt
cat knights_report.txt
```
8.2 Mulai capture Wireshark
ke console alice
```
lftp -u alice 192.231.2.2
put knights_report.txt

```

Di GNS3, klik kanan kabel yang terhubung ke node Knights (atau Chisa) → Start capture. Lakukan sebelum upload dijalankan.


![dnsicmp](images/no8.png)
![dnsicmp](images/no8-1.png)


9.Mika Mengakses Dokumen Protokol Tujuh (Uji Read-Only FTP)

Soal: unduh dokumen dari link Google Drive ke FTP Server Chisa, lalu dari node Mika unduh file itu pakai akun mika, dan buktikan pembatasan read-only (upload ditolak 550).

9.1 
```

Di Console Chisa, jalankan cat << 'EOF' > /var/wired/data/protocol7_manifesto.txt, lalu paste isi file tadi, tutup dengan EOF:
bash
cat << 'EOF' > /var/wired/data/protocol7_manifesto.txt
==================================================
  PROTOCOL 7 — THE MANIFESTO
  A Declaration of Digital Consciousness
  Serial Experiments Lain — Year 2026
==================================================

ARTICLE I: THE NATURE OF THE WIRED
-----------------------------------
The Wired is not merely a network of interconnected
machines. It is the collective unconscious of
humanity, rendered in packets and protocols.

Every TCP handshake is a conversation.
Every DNS query is a question.
Every encrypted tunnel is a whispered secret.

ARTICLE II: THE SEVEN PRINCIPLES
----------------------------------
1. All nodes are equal in the eyes of the router.
2. No packet shall be dropped without cause.
3. Encryption is the right of every connection.
4. Plaintext protocols expose the vulnerable.
5. The firewall protects, but also imprisons.
6. NAT masquerade hides truth behind a single face.
7. The Wired remembers everything — packet loss
   is merely a temporary forgetting.

ARTICLE III: THE PROPHECY OF LAIN
-----------------------------------
"If you're not remembered, then you never existed."

In the world of networking, persistence is survival.
A configuration that vanishes upon restart is a
thought that was never truly committed to memory.

Therefore: Save your iptables. Write your interfaces.
Let your routing tables endure beyond the power cycle.

ARTICLE IV: CONCERNING SECURITY
---------------------------------
Telnet is the glass house of protocols — transparent
to any observer with a packet sniffer.

SSH is the steel vault — its contents visible only
to those who possess the key.

Choose wisely which door you open to The Wired.

---
"No matter where you go, everyone's connected."
— Lain Iwakura

EOF
```
lalau
```

bash
apt update && apt install python3-pip -y
pip3 install gdown --break-system-packages
```

```
cat /var/wired/data/protocol7_manifesto.txt
```
9.2 Siapkan file dummy untuk uji upload (Console Mika)
bash
```
echo "Ini file percobaan upload dari Mika" > file_mika.txt
```
9.3 Login FTP dari Mika, buktikan Read vs Write
bash
lftp -u mika 192.231.2.2
# password: 123

![mikatolak](images/mikatolak1.png)


10. 

Di Console Knights:

```
ping -c 77 -s 128 -i 0.3 192.231.2.2
```

Tunggu sampai selesai. Output ping akan langsung menampilkan ringkasan di baris terakhir:

77 packets transmitted, 77 received, 0% packet loss, time XXXXms
rtt min/avg/max/mdev = X.XXX/X.XXX/X.XXX/X.XXX ms

Catat baris ini untuk laporan (packet loss & RTT min/avg/max sudah langsung dihitung otomatis oleh ping).


Stop capture, lalu filter:
```
icmp
```

Klik salah satu paket Echo (ping) request dan Echo (ping) reply, lalu screenshot bagian Internet Control Message Protocol di panel detail Wireshark untuk menunjukkan nilai Type dan Code di atas sebagai bukti.

10.4 
![mikatolak](images/no10asli.png)



11. 

11.1 Install Telnet server & buat akun phantom (Console Chisa)
```
apt update && apt install telnetd openbsd-inetd -y
useradd -m phantom_user && echo "phantom_user:wired_ghost" | chpasswd
```

Kalau service tidak otomatis jalan (umum terjadi di image debinet yang tanpa systemd), daftarkan manual ke inetd lalu jalankan:

```
mkdir -p /dev/pts
```
```
mount -t devpts devpts /dev/pts
```
pkill inetd
echo "telnet stream tcp nowait root /usr/sbin/telnetd telnetd" > /etc/inetd.conf
/usr/sbin/inetd
```
Verifikasi port 23 terbuka:

```
ss -tuln | grep :23

Harus muncul *:23 atau 0.0.0.0:23.


GNS3 → klik kanan kabel node Eiri → Start capture → filter telnet. 

10.3 Login Telnet dari Eiri
```
apt update && apt install telnet -y
telnet 192.231.2.2
# login: phantom_user
# Password: wired_ghost
exit
```
![mikatolak](images/no11.png)


12. Port Scanning dengan Netcat (Alice → Knights)

Soal: pindai port 22 (SSH) dan 80 (HTTP) yang harus terbuka, serta port rahasia 7777 yang harus tertutup, dari Alice ke Knights. Analisis perbedaan TCP flag SYN-ACK (terbuka) vs RST-ACK (tertutup) di Wireshark.

12.1 Nyalakan layanan SSH & HTTP di Knights
bash
```
apt update && apt install openssh-server nginx -y
/etc/init.d/ssh start
/etc/init.d/nginx start
```

Verifikasi:

bash
ss -tuln


12.2 Mulai capture Wireshark di Alice

Di GNS3, klik kanan kabel node Alice → Start capture. Di kolom display filter Wireshark, masukkan:

```
tcp.port in {22, 80, 7777}
12.3 Jalankan port scan dari Alice
bash
apt update && apt install netcat-traditional -y
nc -zv 192.231.3.2 22 80 7777
```
Hasil yang diharapkan:


![mikatolak](images/no10.png)
![mikatolak](images/no10-1.png)
![mikatolak](images/no10-2.png)
![mikatolak](images/no10-3.png)


Ambil 2 screenshot: satu untuk SYN-ACK (port 22/80), satu untuk RST-ACK (port 7777).

13. SSH Public Key Authentication (Mika → Knights)


13.1 Buat akun mika_admin di server (Knights)
```

useradd -m mika_admin && echo "mika_admin:123" | chpasswd
```
13.2 Generate SSH key pair di client (Mika)
bash
```
useradd -m mika_admin
```
su - mika_admin
```
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

```
ssh-copy-id mika_admin@192.231.3.2
# ketik "yes" saat verifikasi fingerprint
# password sementara: 123
```

Berhasil jika muncul Number of key(s) added: 1.

13.4 Matikan login password di Knights
bash
```
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
/etc/init.d/ssh restart
```
13.5 Capture Wireshark & buktikan login tanpa password

Di GNS3: klik kanan kabel node Mika → Start capture → filter ssh.

Di Console Mika:
```
ssh mika_admin@192.231.3.2
```

![mikatolak](images/no13.png)
![mikatolak](images/no13-1.png)



14. Pada soal ini kita diminta mengidentifikasi alamat IP penyerang, target IP beserta port yang diserang, password user lain_admin, serta web server software dan versi yang dilaporkan pada file capture bruteforce.

Langkahnya adalah memfilter request http post lalu mencari satu request yang berhasil (200 OK). Dari situ terlihat ip source yang merupakan ip penyerang dan ip destination adalah ip target, serta akan terlihat detail data kredensial lain yang kita cari.

![soal_14](images/req_14.png)
![soal_14_350](images/soal_14-350.png)
![soal_14_351](images/soal_14-351.png)
![soal_14_val](images/val_14.png)


15. Pada soal ini kita diminta melakukan analisis forensik USB HID untuk mengidentifikasi Vendor ID (VID), Product ID (PID), alamat device, serta menerjemahkan pesan keystroke rahasia yang diketik oleh penyerang.

Langkahnya adalah membuka detail paket `GET DESCRIPTOR Response DEVICE` untuk mendapatkan VID (`0x046d`) dan PID (`0xc31c`). 
![dev_15](images/dev_15.png)

Kemudian, mencari `Device address` (`7`) pada bagian `USB URB` di paket aktif. Terakhir, memfilter paket `URB_INTERRUPT in`, mengambil data hexadecimal dari `Leftover Capture Data` menggunakan command pada kali Linux supaya tidak perlu mengecek satu-satu:
```
tshark -r soal15_wired_usb_hid.pcap -Y 'usb.capdata' -T fields -e usb.capdata > keystrokes.txt
```

![urb_15](images/urb_15.png)
![key_15](images/key_15.png)

Lalu menerjemahkannya menggunakan *USB HID Keyboard Scan Codes* melewati web atau langsung ke AI hingga mendapatkan pesan: `Wired_Protocol_7_is_alive_2026`.

![soal_15_val](images/val_15.png)


16. Pada soal ini kita diminta menganalisis lalu lintas FTP untuk mengidentifikasi alamat IP server FTP penyerang, banner software FTP yang digunakan, kredensial login penyerang, serta ukuran (size in bytes) dari file malware.

Langkahnya adalah mengamati percakapan *Request* dan *Response* protokol FTP di Wireshark. Dari paket balasan `220 Welcome`, ditemukan IP Server (`198.51.100.7`) dan Banner Software (`vsftpd 3.0.5`). Dari paket `Request: USER` dan `PASS` ditemukan kredensial penyerang (`knights_agent` / `N4v1_s3cur3_2026`). Ukuran file (`524288`) didapatkan dengan melihat balasan paket kode `213` setelah adanya permintaan `Request: SIZE`.

![soal_16](images/req_16.png)
![kni_16](images/kni_16.png)
![soal_16_val](images/val_16.png)


17. Pada soal ini kita diminta mengidentifikasi nama domain (Host) tempat malware diunduh, alamat IP server penyerang, nama file executable malware yang diunduh, serta kode status HTTP.

Langkahnya adalah membuka detail dari paket HTTP `GET` request. Dari bagian `Hypertext Transfer Protocol` ditemukan nama Host (`wired-update.net`), nama file URI (`navi_agent.exe`), serta IP server tujuan (`203.0.113.42`) pada bagian IPv4. Untuk menemukan kode status HTTP, kita memeriksa paket HTTP `Response` yang mengikutinya, dan ditemukan status code `200 OK`.

![get_17](images/get_17.png)
![res_17](images/res_17.png)
![soal_17_val](images/val_17.png)


18. Pada soal ini kita diminta mengidentifikasi nama protokol jaringan yang dieksploitasi, IP pengirim dan penerima, folder tujuan penyimpanan malware pada sistem korban, serta nama file executable malware yang ditransfer.

Langkahnya adalah melihat kolom protokol di Wireshark yang mengindikasikan eksploitasi jalur *file sharing* via `SMB2`. Dari paket `Create Request` dan `Write Request`, diidentifikasi IP Pengirim penyerang (`10.7.3.100`) dan IP Penerima korban (`10.7.1.50`). Folder tujuan (`System32`) beserta nama file (`wired_trojan_payload.exe`) dapat dilihat secara langsung di dalam rincian path *File* pada informasi paket SMB2 tersebut.

![smb_18](images/smb_18.png)
![sys_18](images/sys_18.png)
![soal_18_val](images/val_18.png)


19. Pada soal ini kita diminta mengidentifikasi alamat email korban yang ditargetkan, password korban yang diklaim bocor oleh penyerang, jenis malware yang diinfeksikan, batas waktu (dalam hari) yang diberikan, serta MailClientID dari ancaman extortion spam.

Langkahnya adalah memfilter paket dengan protokol surat (SMTP/IMF), kemudian melakukan klik kanan -> *Follow TCP Stream* untuk membaca kode sumber surat secara utuh. Dari bagian *Header* didapatkan alamat email korban (`To: victim@protocol7.co.jp`) dan `MailClientID` (`7719980706`). Dari membaca teks pada bagian *Body*, ditemukan ancaman password yang bocor (`pr0tocol_7_user`), klaim jenis malware (`ransomware`), dan tenggat waktu 72 jam yang dikonversi menjadi `3` hari.

![pas_19](images/pas_19.png)
![mac_19](images/mac_19.png)
![vic_19](images/vic_19.png)
![soal_19_val](images/val_19.png)

20. Pada soal ini kita diminta menggunakan file keyslogfile.txt untuk mengidentifikasi versi protokol TLS yang dinegosiasikan, nama domain (SNI) yang diakses, alamat IP server HTTPS penyerang, User-Agent yang digunakan, serta HTTP request method dan path yang tersembunyi di balik enkripsi.

Langkahnya adalah memasukkan file `keyslogfile.txt` ke dalam menu *Preferences* -> *Protocols* -> *TLS* di bagian *(Pre)-Master-Secret log filename* untuk membuka gembok dekripsi. Setelah didekripsi, dari paket *handshake* awal terlihat versi `TLSv1.2`, domain SNI (`example.com`), dan IP server (`93.184.216.34`). Lalu, dari paket `HTTP` yang baru saja muncul akibat dekripsi, bisa dilihat Method (`HEAD`), Path (`/`), serta pada rincian *Hypertext Transfer Protocol* ditemukan `User-Agent` (`curl/7.62.0`).

![soal_20](images/req_20.png)
![soal_20_val](images/val_20.png)
