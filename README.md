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
1. Membuat direktori untuk chroot /var/chroot
mkdir -p digunakan untuk membuat direktori /var/chroot, yang akan menjadi lingkungan chroot.
```bash
sudo mkdir -p /var/chroot
```
2. Buat struktur direktori dasar dalam Chroot
- /bin → Menyimpan perintah dasar seperti bash, ls, nano, dll.
- /lib & /lib64 → Menyimpan library yang dibutuhkan oleh program dalam CHROOT.
- /usr → Menyimpan tambahan binary dan library lainnya.
- /home → Direktori untuk pengguna dalam CHROOT.
- /etc → Menyimpan file konfigurasi sistem.
- /dev → Menyimpan perangkat virtual seperti /dev/null.
- /proc & /sys → Menyediakan informasi sistem dalam CHROOT.
- /tmp → Direktori sementara untuk penyimpanan file sementara.
```bash
mkdir -p /var/chroot/{bin,lib,lib64,usr,home,etc,dev,proc,sys,tmp}
```
3. chmod direktori /tmp
Opsi 1777 memastikan bahwa semua pengguna bisa membaca, menulis, dan mengeksekusi file dalam /tmp, tapi selain owner menjadi read only.
```bash
chmod 1777 /var/chroot/tmp
```
4. Copy direktori bash
bash adalah shell utama yang akan digunakan untuk CHROOT nanti
```bash
cp --parents /bin/bash /var/chroot/
```
5. Menyalin command dasar yang diperlukan untuk pengujian
```bash
cp --parents /bin/{ls,cat,mkdir,rmdir,rm,cp,mv,echo,nano,sh,chmod,touch,pwd,grep} /var/chroot/
```
6. Menyalin library yang dibutuhkan di dalam CHROOT
```bash
cp --parents /lib/x86_64-linux-gnu/{libtinfo.so.6,libc.so.6,libselinux.so.1,libpcre2-8.so.0,libncursesw.so.6} /var/chroot/
cp --parents /lib64/ld-linux-x86-64.so.2 /var/chroot/
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
