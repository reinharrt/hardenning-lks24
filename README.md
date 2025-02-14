# hardenning-lks24

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

### 1.3 Hardening Closed Unusual Open Port
1. Buat direktori Chroot
```bash
sudo mkdir -p /var/chroot
```
2. Buat struktur direktori dasar dalam Chroot
```bash
sudo mkdir -p /var/chroot/{bin,lib,lib64,etc,dev,usr,proc,sys,home,tmp}
```
3. Copy binary penting ke dalam Chroot
```bash
sudo cp -v /bin/bash /var/chroot/bin/
sudo cp -v /bin/ls /var/chroot/bin/
sudo cp -v /bin/mkdir /var/chroot/bin/
sudo cp -v /bin/cp /var/chroot/bin/
sudo cp -v /bin/rm /var/chroot/bin/
sudo cp -v /bin/nano /var/chroot/bin/
sudo cp -v /bin/cat /var/chroot/bin/
```
4. Copy pustaka yang diperlukan
```bash
sudo mkdir -p /var/chroot/lib/x86_64-linux-gnu
sudo cp -v /lib/x86_64-linux-gnu/libtinfo.so.6 /var/chroot/lib/x86_64-linux-gnu/
sudo cp -v /lib/x86_64-linux-gnu/libdl.so.2 /var/chroot/lib/x86_64-linux-gnu/
sudo cp -v /lib/x86_64-linux-gnu/libc.so.6 /var/chroot/lib/x86_64-linux-gnu/
sudo cp -v /lib/x86_64-linux-gnu/libpcre2-8.so.0 /var/chroot/lib/x86_64-linux-gnu/
```
5. Copy dynamic linker (ld-linux)
```bash
sudo mkdir -p /var/chroot/lib64
sudo cp -v /lib64/ld-linux-x86-64.so.2 /var/chroot/lib64/
```
6. Konfigurasi device di dalam Chroot
```bash
sudo mknod -m 666 /var/chroot/dev/null c 1 3
sudo mknod -m 666 /var/chroot/dev/tty c 5 0
sudo mknod -m 666 /var/chroot/dev/zero c 1 5
sudo mknod -m 666 /var/chroot/dev/random c 1 8
```
7. Mount filesystem yang dibutuhkan
```bash
sudo mount -o bind /dev /var/chroot/dev
sudo mount -o bind /proc /var/chroot/proc
sudo mount -o bind /sys /var/chroot/sys
```
8. Masuk ke dalam Chroot
```bash
sudo chroot /var/chroot /bin/bash
```
