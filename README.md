# hardenning-seleksi-lks25

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

### 1.3 Hardening CHROOT Shell
1. Membuat Direktori CHROOT 
Membuat direktori CHROOT adalah langkah pertama dalam mengisolasi lingkungan pengguna. Direktori ini akan menjadi root filesystem baru bagi pengguna yang dibatasi. Dengan menggunakan perintah `mkdir -p`, kita memastikan bahwa direktori akan dibuat tanpa error meskipun sudah ada. Direktori `/var/chroot` dipilih karena lokasi ini umum digunakan untuk keperluan isolasi seperti ini.  

```bash
sudo mkdir -p /var/chroot
```  

2. Menambahkan Pengguna yang Akan Dibatasi ke CHROOT
Pengguna yang akan dibatasi aksesnya perlu dibuat terlebih dahulu. Perintah `adduser` digunakan untuk membuat pengguna baru, dalam hal ini `chrootuser`. Setelah itu, direktori home untuk pengguna tersebut harus dibuat di dalam lingkungan CHROOT. Ini memastikan bahwa pengguna hanya memiliki akses ke direktori home mereka yang terisolasi. Perintah `chown` digunakan untuk mengubah kepemilikan direktori home tersebut agar sesuai dengan pengguna yang bersangkutan.  

```bash
sudo adduser chrootuser
sudo mkdir -p /var/chroot/home/chrootuser
sudo chown chrootuser:chrootuser /var/chroot/home/chrootuser
```  

3. Menyalin Binary yang Diperlukan
Agar pengguna dapat menjalankan perintah dasar di dalam lingkungan CHROOT, binary seperti `bash`, `ls`, `mkdir`, dan `nano` perlu disalin ke dalam direktori CHROOT. Binary ini adalah program inti yang akan digunakan oleh pengguna untuk berinteraksi dengan sistem.  

```bash
sudo cp --parents /bin/bash /var/chroot/
sudo cp --parents /bin/ls /var/chroot/
sudo cp --parents /bin/mkdir /var/chroot/
sudo cp --parents /bin/nano /var/chroot/
sudo cp --parents /bin/cat /var/chroot/
sudo cp --parents /bin/rm /var/chroot/
sudo cp --parents /usr/bin/id /var/chroot/
```  

---

4. Menyalin Library yang Diperlukan 
Setelah binary disalin, library yang diperlukan oleh binary tersebut juga harus disalin agar perintah dapat berfungsi dengan baik. Library seperti `libc`, `libtinfo`, dan lainnya adalah dependensi yang dibutuhkan oleh binary untuk berjalan.  

```bash
sudo cp --parents /lib/x86_64-linux-gnu/libtinfo.so.6 /var/chroot/
sudo cp --parents /lib/x86_64-linux-gnu/libdl.so.2 /var/chroot/
sudo cp --parents /lib/x86_64-linux-gnu/libc.so.6 /var/chroot/
sudo cp --parents /lib64/ld-linux-x86-64.so.2 /var/chroot/
sudo cp --parents /lib/x86_64-linux-gnu/libselinux.so.1 /var/chroot/
sudo cp --parents /lib/x86_64-linux-gnu/libpcre2-8.so.0 /var/chroot/
sudo cp --parents /lib/x86_64-linux-gnu/libncursesw.so.6 /var/chroot/
```

5. Menyalin Terminfo agar Nano Bisa Bekerja
Aplikasi seperti `nano` memerlukan file `terminfo` untuk dapat berinteraksi dengan terminal dengan benar. File-file ini berisi informasi tentang kemampuan terminal, seperti cara menangani warna atau kursor. Tanpa file ini, `nano` mungkin tidak akan berfungsi dengan baik.  

```bash
sudo mkdir -p /var/chroot/usr/share/terminfo
sudo cp -r /usr/share/terminfo/* /var/chroot/usr/share/terminfo/
```  


6. Membuat Struktur Direktori dalam CHROOT
Beberapa direktori seperti `dev`, `proc`, dan `sys` diperlukan agar sistem dapat berfungsi dengan baik di dalam CHROOT. Direktori ini akan di-mount ke dalam CHROOT untuk menyediakan akses ke perangkat, proses, dan sistem file yang diperlukan.  

```bash
sudo mkdir -p /var/chroot/{dev,proc,sys,home,tmp,usr,bin,etc,lib,lib64}
sudo mount -o bind /dev /var/chroot/dev
sudo mount -o bind /proc /var/chroot/proc
sudo mount -o bind /sys /var/chroot/sys
```  


7. Mengonfigurasi SSH agar Menggunakan CHROOT untuk Pengguna Tertentu
Untuk membatasi pengguna tertentu ke lingkungan CHROOT, kita perlu mengedit file konfigurasi SSH (`sshd_config`). Dengan menambahkan blok `Match User`, kita dapat menentukan bahwa pengguna tertentu (dalam hal ini `chrootuser`) hanya akan memiliki akses ke direktori CHROOT.  

```bash
sudo nano /etc/ssh/sshd_config
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
sudo systemctl restart ssh
```  

8. Pengujian CHROOT dan SSH
Setelah konfigurasi selesai, kita dapat menguji apakah pengguna `chrootuser` benar-benar terbatas pada lingkungan CHROOT. Coba login ke server menggunakan SSH sebagai `chrootuser`.  

```bash
ssh chrootuser@127.0.0.1
``` 
