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
