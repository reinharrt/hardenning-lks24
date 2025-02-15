# hardenning-seleksi-lks25
Dokumen ini berisi kumpulan penjelasan command-command yang digunakan untuk menyelesaikan soal-soal hardening. Hardening adalah proses mengamankan sistem dengan mengurangi celah keamanan dan membatasi akses yang tidak diperlukan. Command-command di bawah ini mencakup konfigurasi CHROOT, manajemen pengguna, dan isolasi lingkungan.
---
### 1.1 Document Host Information
1. Menulis teks ke dua file (/etc/issue untuk login konsol & /etc/issue.net untuk SSH).
```bash
echo "Unauthorized access prohibited" | sudo tee /etc/issue /etc/issue.net
```
2. MOTD (Opsional)
Menonaktifkan skrip MOTD.
```bash
sudo chmod -x /etc/update-motd.d/*
```
Mengaktifkan skrip MOTD.
```bash
sudo chmod +x /etc/update-motd.d/*
```
---
### 1.2 Hardening Closed Unusual Open Port
1. Menampilkan daftar port yang sedang listening atau aktif.
```bash
ss -tuln
```
2. Close atau menutup koneksi dengan iptables.
Memblokir Input
```bash
sudo iptables -A OUTPUT -p tcp --dport {port} -j DROP
sudo iptables -A OUTPUT -p tcp --dport {port} -j DROP
```
Reject Input
```bash
sudo iptables -A OUTPUT -p tcp --dport {port} -j REJECT
sudo iptables -A OUTPUT -p tcp --dport {port} -j REJECT
```
3. Opsional
Membuka port baru untuk pengujian.
```bash
nc -lvp {port}
```
Melihat apakah port sudah berhasil di tutup atau belum.
```bash
nc -zv {ip address} {port}
```
---
### 1.3 Hardening CHROOT Shell
1. Membuat Direktori CHROOT 
Membuat direktori CHROOT adalah langkah pertama dalam mengisolasi lingkungan pengguna. Direktori ini akan menjadi root filesystem baru bagi pengguna yang dibatasi. Dengan menggunakan perintah `mkdir -p`, kita memastikan bahwa direktori akan dibuat tanpa error meskipun sudah ada. Direktori `/var/chroot` dipilih karena lokasi ini umum digunakan untuk keperluan seperti ini.  

```bash
mkdir -p /var/chroot
```  

2. Menambahkan Pengguna yang Akan Dibatasi ke CHROOT
Pengguna yang akan dibatasi aksesnya perlu dibuat terlebih dahulu. Perintah `adduser` digunakan untuk membuat pengguna baru, dalam hal ini `chrootuser`. Setelah itu, direktori home untuk pengguna tersebut harus dibuat di dalam lingkungan CHROOT. Ini memastikan bahwa pengguna hanya memiliki akses ke direktori home mereka yang terisolasi. Perintah `chown` digunakan untuk mengubah kepemilikan direktori home tersebut agar sesuai dengan pengguna yang bersangkutan.  

```bash
adduser chrootuser
mkdir -p /var/chroot/home/chrootuser
chown chrootuser:chrootuser /var/chroot/home/chrootuser
```  

3. Menyalin Binary yang Diperlukan
Agar pengguna dapat menjalankan perintah dasar di dalam lingkungan CHROOT, binary seperti `bash`, `ls`, `mkdir`, dan `nano` perlu disalin ke dalam direktori CHROOT. Binary ini adalah program inti yang akan digunakan oleh pengguna untuk berinteraksi dengan sistem.  

```bash
cp --parents /bin/bash /var/chroot/
cp --parents /bin/ls /var/chroot/
cp --parents /bin/mkdir /var/chroot/
cp --parents /bin/nano /var/chroot/
cp --parents /bin/cat /var/chroot/
cp --parents /bin/rm /var/chroot/
cp --parents /usr/bin/id /var/chroot/
```  

---

4. Menyalin Library yang Diperlukan 
Setelah binary disalin, library yang diperlukan oleh binary tersebut juga harus disalin agar perintah dapat berfungsi dengan baik. Library seperti `libc`, `libtinfo`, dan lainnya adalah dependensi yang dibutuhkan oleh binary untuk berjalan.  

```bash
cp --parents /lib/x86_64-linux-gnu/libtinfo.so.6 /var/chroot/
cp --parents /lib/x86_64-linux-gnu/libdl.so.2 /var/chroot/
cp --parents /lib/x86_64-linux-gnu/libc.so.6 /var/chroot/
cp --parents /lib64/ld-linux-x86-64.so.2 /var/chroot/
cp --parents /lib/x86_64-linux-gnu/libselinux.so.1 /var/chroot/
cp --parents /lib/x86_64-linux-gnu/libpcre2-8.so.0 /var/chroot/
cp --parents /lib/x86_64-linux-gnu/libncursesw.so.6 /var/chroot/
```

5. Menyalin Terminfo agar Nano Bisa Bekerja
Aplikasi seperti `nano` memerlukan file `terminfo` untuk dapat berinteraksi dengan terminal dengan benar. File-file ini berisi informasi tentang kemampuan terminal, seperti cara menangani warna atau kursor. Tanpa file ini, `nano` mungkin tidak akan berfungsi dengan baik.  

```bash
mkdir -p /var/chroot/usr/share/terminfo
cp -r /usr/share/terminfo/* /var/chroot/usr/share/terminfo/
```  


6. Membuat Struktur Direktori dalam CHROOT
Beberapa direktori seperti `dev`, `proc`, dan `sys` diperlukan agar sistem dapat berfungsi dengan baik di dalam CHROOT. Direktori ini akan di-mount ke dalam CHROOT untuk menyediakan akses ke perangkat, proses, dan sistem file yang diperlukan.  

```bash
mkdir -p /var/chroot/{dev,proc,sys,home,tmp,usr,bin,etc,lib,lib64}
mount -o bind /dev /var/chroot/dev
mount -o bind /proc /var/chroot/proc
mount -o bind /sys /var/chroot/sys
```  


7. Mengonfigurasi SSH agar Menggunakan CHROOT untuk Pengguna Tertentu
Untuk membatasi pengguna tertentu ke lingkungan CHROOT, kita perlu mengedit file konfigurasi SSH (`sshd_config`). Dengan menambahkan blok `Match User`, kita dapat menentukan bahwa pengguna tertentu (dalam hal ini `chrootuser`) hanya akan memiliki akses ke direktori CHROOT.  

```bash
nano /etc/ssh/sshd_config
```  

Tambahkan atau ubah baris berikut:  
```
Match User chrootuser
    ChrootDirectory /var/chroot
    X11Forwarding no
    AllowTcpForwarding no
```  

Setelah itu, restart layanan SSH untuk menerapkan perubahan.  

```bash
systemctl restart ssh
```  

8. Pengujian CHROOT dan SSH
Setelah konfigurasi selesai, kita dapat menguji apakah pengguna `chrootuser` benar-benar terbatas pada lingkungan CHROOT. Coba login ke server menggunakan SSH sebagai `chrootuser`.  

```bash
ssh chrootuser@127.0.0.1
```
---

### 1.4 Hardening Certificate Shell Login
1. Kita generate `SSH Key Pair`, ini merupakan key dengan enrypt RSA dan byte lenght 4096.
```bash
ssh-keygen -t rsa -b 4096 -C "freefire@sijactf.com"
```
2. Konfigurasi SSH Server
```
nano /etc/ssh/sshd_config
```
Ubah atau atur semua parameter yang diperlukan.
```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
Terakhir restart SSH Service.
```
systemctl restart ssh
```
3. Kita copy `SSH Key Pair` yang sudah kita generate sebelumnya ke server.
```
cat ~/.ssh/id_rsa.pub | ssh sshuser@your_server_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
4. Pengujian dengan login
```
ssh sshuser@your_server_ip
```

### 1.5 Hardening Directory Listing
1. Install Apache dan Dependecies yang dibutuhkan.
2. Isi `/var/www/html` dengan file non read-able by browser misalnya txt.
3. Nano file konfigurasi.
```
nano /etc/apache2/apache2.conf
```
Ubah bagian ini, agar directory listing mati.
```
<Directory /var/www/html/test>
    Options -Indexes
</Directory>
```
---
### 1.6 IDS (Snort)
1. Konfigurasi Snort
Pastikan Snort sudah terinstal dan dikonfigurasi dengan benar seperti yang dijelaskan sebelumnya. Berikut adalah ringkasan konfigurasi:

1. File Konfigurasi Snort (`/etc/snort/snort.conf`)  
   - Pastikan `HOME_NET` dan `EXTERNAL_NET` sudah sesuai:
     ```bash
     ipvar HOME_NET 192.168.1.0/24
     ipvar EXTERNAL_NET !$HOME_NET
     ```
   - Aktifkan rule komunitas atau tambahkan rule khusus di `/etc/snort/rules/local.rules`.

2. Rule untuk Deteksi Serangan
   Tambahkan rule berikut ke `/etc/snort/rules/local.rules`:
   - SQL Injection:
     ```bash
     alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"SQL Injection Detected"; content:"%27"; nocase; pcre:"/(\%27)|(\')|(\-\-)|(\%23)|(#)/i"; sid:1000001; rev:1;)
     ```
   - Blind SQL Injection:
     ```bash
     alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"Blind SQL Injection Detected"; content:"sleep("; nocase; sid:1000002; rev:1;)
     ```
   - Brute Force Attack:
     ```bash
     alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (msg:"Brute Force Attack Detected"; threshold:type threshold, track by_src, count 5, seconds 60; sid:1000003; rev:1;)
     ```
   - Cross-Site Scripting (XSS):
     ```bash
     alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"XSS Attack Detected"; content:"<script>"; nocase; sid:1000004; rev:1;)
     ```

3. Jalankan Snort  
   - Jalankan Snort dalam mode IDS:
     ```bash
     sudo snort -A console -q -u snort -g snort -c /etc/snort/snort.conf -i <interface>
     ```
     Ganti `<interface>` dengan nama interface jaringan Anda (misalnya, `eth0` atau `ens33`).

2. Pengujian Serangan

SQL Injection
1. Gunakan `curl` untuk Mengirim Payload SQL Injection
   - Kirim payload SQL Injection ke web server:
     ```bash
     curl -X GET "http://<IP_SERVER>/index.php?id=1%27%20OR%201=1--"
     ```
     Ganti `<IP_SERVER>` dengan alamat IP server Anda.

2. Periksa Log Snort
   - Snort akan mendeteksi payload SQL Injection dan mencatatnya di log:
     ```bash
     sudo tail -f /var/log/snort/alert
     ```
   - Anda akan melihat pesan seperti:
     ```
     [**] [1:1000001:1] SQL Injection Detected [**]
     ```

Blind SQL Injection
1. Gunakan `curl` untuk Mengirim Payload Blind SQL Injection
   - Kirim payload Blind SQL Injection:
     ```bash
     curl -X GET "http://<IP_SERVER>/index.php?id=1%20AND%20SLEEP(5)"
     ```

2. Periksa Log Snort  
   - Snort akan mendeteksi payload Blind SQL Injection:
     ```
     [**] [1:1000002:1] Blind SQL Injection Detected [**]
     ```


Brute Force Attack
1. Gunakan `hydra` untuk Simulasi Brute Force
   - Instal `hydra` jika belum ada:
     ```bash
     sudo apt install hydra
     ```
   - Jalankan serangan brute force ke SSH server:
     ```bash
     hydra -l root -P /usr/share/wordlists/rockyou.txt <IP_SERVER> ssh
     ```

2. **Periksa Log Snort**  
   - Snort akan mendeteksi aktivitas brute force dan mencatatnya:
     ```
     [**] [1:1000003:1] Brute Force Attack Detected [**]
     ```

---

Cross-Site Scripting (XSS)
1. Gunakan `curl` untuk Mengirim Payload XSS  
   - Kirim payload XSS:
     ```bash
     curl -X GET "http://<IP_SERVER>/index.php?name=<script>alert('XSS')</script>"
     ```

2. Periksa Log Snort
   - Snort akan mendeteksi payload XSS
     ```
     [**] [1:1000004:1] XSS Attack Detected [**]
     ```

3. Analisis Hasil Pengujian
- Semua alert yang dihasilkan oleh Snort akan dicatat di `/var/log/snort/alert`.
- Anda dapat memeriksa log tersebut untuk memastikan bahwa Snort berhasil mendeteksi serangan:
  ```bash
  sudo cat /var/log/snort/alert
  ```

### 1.7 Analisis dan Patching Security Header Web
 1. Install Apache Web Server
Jika Apache belum terinstal, jalankan perintah berikut:  

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y
```

Setelah instalasi selesai, pastikan Apache berjalan:  

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2
```

 2. Install Modul yang Dibutuhkan
Apache perlu modul tambahan untuk menangani header dan SSL. Aktifkan modul-modul berikut:  

```bash
sudo a2enmod headers
sudo a2enmod ssl
sudo systemctl restart apache2
```


 3. Buat atau Edit Virtual Host
Buka konfigurasi Virtual Host default:  

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Tambahkan security headers di dalam blok `<VirtualHost *:80>` atau buat konfigurasi khusus jika menggunakan HTTPS (`/etc/apache2/sites-available/default-ssl.conf`).  

Tambahkan bagian ini di dalam `<VirtualHost>`:

```apache
<IfModule mod_headers.c>
     #HSTS: Memaksa HTTPS
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

     #Mencegah Clickjacking
    Header always set X-Frame-Options "DENY"

     #Mencegah MIME Sniffing
    Header always set X-Content-Type-Options "nosniff"

     #Mengontrol Referer Header
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

     #Mengatur izin API browser (Permissions Policy)
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"

     #Mencegah XSS & injeksi (CSP)
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; frame-ancestors 'none';"
</IfModule>
```

Simpan (`CTRL + X`, lalu `Y`, dan tekan `Enter`).


 4. Restart Apache
Setelah mengedit konfigurasi, restart Apache:

```bash
sudo systemctl restart apache2
```

---

 5. Verifikasi Header Security
Cek apakah header sudah aktif dengan perintah:  

```bash
curl -I http://localhost
```

Atau jika pakai HTTPS:

```bash
curl -I https://localhost --insecure
```


 (Opsional) 6. Install dan Konfigurasi SSL (HTTPS)
Jika ingin menggunakan HTTPS, install Let's Encrypt untuk mendapatkan SSL gratis:  

```bash
sudo apt install certbot python3-certbot-apache -y
```

Kemudian jalankan perintah ini untuk mendapatkan sertifikat SSL secara otomatis:  

```bash
sudo certbot --apache
```

Ikuti petunjuk di layar untuk menyelesaikan instalasi.

Setelah selesai, uji apakah SSL sudah berjalan:  

```bash
sudo systemctl restart apache2
curl -I https://yourdomain.com
```

 7. Hardening Tambahan
Untuk meningkatkan keamanan lebih lanjut:
- Pastikan semua paket up to date:  
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- Batasi informasi server (server signature) dengan menambahkan di `/etc/apache2/conf-available/security.conf`:  
  ```apache
  ServerSignature Off
  ServerTokens Prod
  ```

Restart Apache:

```bash
sudo systemctl restart apache2
```
