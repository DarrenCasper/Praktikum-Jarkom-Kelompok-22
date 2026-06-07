# Laporan Tugas Modul 4 — Firewall dan NAT

## 1. Overview

Pada tugas modul ini, jaringan dibuat dengan segmentasi **WAN**, **LAN**, dan **DMZ**. Topologi menggunakan beberapa perangkat utama, yaitu MikroTik, FortiGate, Cisco Router, TinyCore Linux, dan Ubuntu Server.

Tujuan dari tugas ini adalah:

* Membuat jaringan WAN, LAN, dan DMZ.
* Menghubungkan Client LAN ke server DMZ.
* Menghubungkan Client WAN ke server DMZ melalui VIP atau port forwarding.
* Membatasi akses langsung dari WAN ke LAN.
* Membatasi akses langsung dari WAN ke IP asli server DMZ.
* Memahami konsep firewall, NAT, routing statis, DMZ, dan port forwarding.

---

## 2. Topologi Jaringan dan Perangkat yang Digunakan

*Topologi Jaringan*

![gambar1](assets/topologi%20tumod%20modul%204%20jarkom.png)

| Perangkat            | Fungsi                                           |
| -------------------- | ------------------------------------------------ |
| Cloud / Jaringan Lab | Sumber koneksi luar atau internet dari PNETLab   |
| MikroTik ISP         | Router luar atau simulasi ISP                    |
| FortiGate            | Firewall utama yang memisahkan WAN, LAN, dan DMZ |
| Cisco Router         | Router internal untuk jaringan LAN               |
| Client LAN TinyCore  | Client internal di belakang Cisco Router         |
| Client WAN TinyCore  | Client sisi luar untuk menguji akses ke DMZ      |
| Ubuntu Server DMZ    | Web server yang diletakkan di zona DMZ           |

---

## 3. Segmentasi Jaringan

| Segment                 | Network                | Keterangan                     |
| ----------------------- | ---------------------- | ------------------------------ |
| Jaringan Lab / Internet | DHCP dari jaringan lab | Sumber koneksi luar            |
| ISP ke FortiGate        | `10.10.10.0/30`        | Link MikroTik ISP ke FortiGate |
| Client WAN              | `172.16.100.0/24`      | Jaringan client sisi luar      |
| FortiGate ke Cisco      | `10.20.20.0/30`        | Link FortiGate ke Cisco Router |
| LAN                     | `192.168.10.0/24`      | Jaringan internal client       |
| DMZ                     | `192.168.20.0/24`      | Jaringan server DMZ            |

Penjelasan singkat:

* **WAN** adalah sisi luar jaringan.
* **LAN** adalah jaringan internal yang digunakan oleh client lokal.
* **DMZ** adalah area khusus untuk server yang boleh diakses dari luar dengan aturan firewall tertentu.
* **FortiGate** menjadi perangkat utama yang mengatur akses antarsegmen.

---

## 4. Tabel IP Address

| Perangkat           | Interface       | IP Address         | Gateway        | Keterangan                        |
| ------------------- | --------------- | ------------------ | -------------- | --------------------------------- |
| MikroTik ISP        | `ether1`        | DHCP Client        | DHCP lab       | Terhubung ke Cloud / jaringan lab |
| MikroTik ISP        | `ether2`        | `10.10.10.1/30`    | -              | Link ke FortiGate `port1`         |
| MikroTik ISP        | `ether3`        | `172.16.100.1/24`  | -              | Gateway Client WAN                |
| FortiGate           | `port1`         | `10.10.10.2/30`    | `10.10.10.1`   | Interface WAN                     |
| FortiGate           | `port2`         | `10.20.20.1/30`    | -              | Interface INSIDE ke Cisco         |
| FortiGate           | `port3`         | `192.168.20.1/24`  | -              | Interface DMZ                     |
| Cisco Router        | `G0/0`          | `10.20.20.2/30`    | -              | Link ke FortiGate `port2`         |
| Cisco Router        | `G0/1`          | `192.168.10.1/24`  | -              | Gateway LAN                       |
| Client LAN TinyCore | `eth0`          | `192.168.10.10/24` | `192.168.10.1` | Client internal                   |
| Client WAN TinyCore | `eth0`          | `172.16.100.10/24` | `172.16.100.1` | Client luar                       |
| Ubuntu Server DMZ   | `eth0` / `ens3` | `192.168.20.10/24` | `192.168.20.1` | Web server DMZ                    |

---

## 5. Persiapan Awal di PNETLab

Sebelum konfigurasi dilakukan, siapkan topologi terlebih dahulu.

### Langkah Persiapan

1. Buat workspace baru di PNETLab.
2. Tambahkan semua node:

   * Cloud / Network
   * MikroTik ISP
   * FortiGate
   * Cisco Router
   * Client LAN TinyCore
   * Client WAN TinyCore
   * Ubuntu Server DMZ
3. Hubungkan interface sesuai tabel IP address.
4. Start semua perangkat.
5. Tunggu perangkat seperti FortiGate, Cisco Router, dan Ubuntu Server sampai selesai booting.
6. Jika terminal belum muncul, tekan `Enter`.

### Catatan

* FortiGate dan Ubuntu biasanya butuh waktu boot lebih lama.
* Pastikan kabel masuk ke interface yang benar.
* Jika konfigurasi terlihat benar tetapi ping gagal, cek kembali cabling dan interface.

---

# 6. Konfigurasi MikroTik ISP

## 6.1 Tujuan

MikroTik ISP digunakan sebagai router luar. MikroTik harus:

* Mendapat IP dari Cloud/lab menggunakan DHCP Client.
* Terhubung ke FortiGate melalui network `10.10.10.0/30`.
* Menjadi gateway Client WAN melalui network `172.16.100.0/24`.
* Melakukan NAT masquerade ke jaringan luar.
* Memiliki static route menuju LAN dan DMZ melalui FortiGate.

---

## 6.2 DHCP Client pada `ether1`

Interface `ether1` diarahkan ke Cloud/lab, sehingga IP didapat secara otomatis.

```bash
/ip dhcp-client add interface=ether1 add-default-route=yes use-peer-dns=yes disabled=no
```

Cek hasil DHCP Client:

```bash
/ip dhcp-client print
/ip address print
```

Jika berhasil, status DHCP Client akan menjadi `bound`.

Contoh IP yang mungkin muncul:

```text
10.4.71.14/20
```

IP dari DHCP bisa berbeda tergantung jaringan lab.

---

## 6.3 IP Address MikroTik

Tambahkan IP untuk link ke FortiGate:

```bash
/ip address add address=10.10.10.1/30 interface=ether2 comment="Link to FortiGate"
```

Tambahkan IP untuk gateway Client WAN:

```bash
/ip address add address=172.16.100.1/24 interface=ether3 comment="Gateway Client WAN"
```

Cek konfigurasi IP:

```bash
/ip address print
```

---

## 6.4 NAT Masquerade MikroTik

NAT masquerade digunakan agar jaringan di belakang MikroTik dapat keluar ke jaringan luar menggunakan IP interface `ether1`.

```bash
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade comment="NAT to Internet"
```

Cek konfigurasi NAT:

```bash
/ip firewall nat print
```

---

## 6.5 Static Route MikroTik

MikroTik perlu mengetahui bahwa jaringan LAN dan DMZ berada di belakang FortiGate. Oleh karena itu, tambahkan route menuju LAN dan DMZ melalui IP FortiGate `10.10.10.2`.

Route ke LAN:

```bash
/ip route add dst-address=192.168.10.0/24 gateway=10.10.10.2 comment="Route to LAN via FortiGate"
```

Route ke DMZ:

```bash
/ip route add dst-address=192.168.20.0/24 gateway=10.10.10.2 comment="Route to DMZ via FortiGate"
```

Cek routing table:

```bash
/ip route print
```

---

## 6.6 Pengujian MikroTik

Tes koneksi dari MikroTik ke FortiGate:

```bash
/ping 10.10.10.2
```

Tes koneksi MikroTik ke internet:

```bash
/ping 8.8.8.8
```

Jika ping ke `10.10.10.2` berhasil, berarti link MikroTik ke FortiGate sudah benar.

### Screenshot MikroTik

Ambil screenshot:

* `/ip address print`
* `/ip dhcp-client print`
* `/ip firewall nat print`
* `/ip route print`
* `/ping 10.10.10.2`

---

# 7. Konfigurasi FortiGate

## 7.1 Tujuan

FortiGate digunakan sebagai firewall utama. FortiGate harus:

* Terhubung ke MikroTik melalui `port1`.
* Terhubung ke Cisco Router melalui `port2`.
* Terhubung ke Ubuntu Server DMZ melalui `port3`.
* Memiliki default route ke MikroTik.
* Memiliki static route ke LAN melalui Cisco.
* Memiliki firewall policy LAN ke WAN dan LAN ke DMZ.
* Memiliki VIP agar Client WAN bisa mengakses server DMZ melalui IP WAN FortiGate.

---

## 7.2 Login FortiGate

Gunakan akun berikut jika sesuai dengan image FortiGate yang digunakan:

```text
Username : admin
Password : 22
```

---

## 7.3 Konfigurasi Interface FortiGate

Masuk ke CLI FortiGate, lalu jalankan:

```bash
config system interface
    edit port1
        set alias "WAN_to_MikroTik"
        set mode static
        set ip 10.10.10.2 255.255.255.252
        set allowaccess ping http https ssh
    next

    edit port2
        set alias "INSIDE_to_Cisco"
        set mode static
        set ip 10.20.20.1 255.255.255.252
        set allowaccess ping http https ssh
    next

    edit port3
        set alias "DMZ_to_Server"
        set mode static
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping http https ssh
    next
end
```

Verifikasi interface:

```bash
show system interface port1
show system interface port2
show system interface port3
```

---

## 7.4 Routing FortiGate

Tambahkan default route ke MikroTik ISP:

```bash
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.10.10.1
        set device port1
    next
end
```

Tambahkan static route menuju LAN melalui Cisco Router:

```bash
config router static
    edit 2
        set dst 192.168.10.0 255.255.255.0
        set gateway 10.20.20.2
        set device port2
    next
end
```

Cek routing table:

```bash
get router info routing-table all
```

---

## 7.5 Address Object FortiGate

Address object digunakan agar firewall policy lebih rapi.

```bash
config firewall address
    edit "LAN_192.168.10.0"
        set subnet 192.168.10.0 255.255.255.0
    next

    edit "DMZ_Server_192.168.20.10"
        set subnet 192.168.20.10 255.255.255.255
    next

    edit "Client_WAN_172.16.100.10"
        set subnet 172.16.100.10 255.255.255.255
    next
end
```

Cek address object:

```bash
show firewall address
```

---

## 7.6 Policy LAN ke WAN

Policy ini digunakan agar Client LAN bisa mengakses jaringan luar.

```bash
config firewall policy
    edit 1
        set name "LAN_to_WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "LAN_192.168.10.0"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

Penjelasan:

* `srcintf port2` berarti trafik berasal dari sisi Cisco/LAN.
* `dstintf port1` berarti trafik keluar ke sisi WAN.
* `set nat enable` digunakan agar client LAN dapat keluar ke jaringan luar.

---

## 7.7 Policy LAN ke DMZ

Policy ini digunakan agar Client LAN dapat mengakses server DMZ.

```bash
config firewall policy
    edit 2
        set name "LAN_to_DMZ"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "LAN_192.168.10.0"
        set dstaddr "DMZ_Server_192.168.20.10"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

Penjelasan:

* NAT dimatikan karena LAN dan DMZ masih berada dalam jaringan internal.
* Policy ini mengizinkan Client LAN melakukan ping dan akses web server DMZ.

---

## 7.8 VIP / Port Forwarding ke Server DMZ

VIP digunakan supaya Client WAN dapat mengakses web server DMZ melalui IP WAN FortiGate, yaitu `10.10.10.2`.

```bash
config firewall vip
    edit "VIP_HTTP_DMZ"
        set extintf "port1"
        set extip 10.10.10.2
        set mappedip "192.168.20.10"
        set portforward enable
        set protocol tcp
        set extport 80
        set mappedport 80
    next
end
```

Penjelasan:

* Jika Client WAN mengakses `http://10.10.10.2`, FortiGate akan meneruskan request ke `192.168.20.10`.
* Konfigurasi ini adalah contoh destination NAT atau port forwarding.

---

## 7.9 Policy WAN ke DMZ HTTP

VIP saja belum cukup. FortiGate tetap membutuhkan policy agar trafik dari WAN ke DMZ diizinkan.

```bash
config firewall policy
    edit 3
        set name "WAN_to_DMZ_HTTP"
        set srcintf "port1"
        set dstintf "port3"
        set srcaddr "Client_WAN_172.16.100.10"
        set dstaddr "VIP_HTTP_DMZ"
        set action accept
        set schedule "always"
        set service "HTTP"
        set nat disable
    next
end
```

Penjelasan:

* Sumber trafik adalah Client WAN.
* Tujuan trafik adalah VIP, bukan IP asli server DMZ.
* Service yang diizinkan hanya HTTP.

---

## 7.10 Policy Tambahan DMZ ke WAN

Policy ini digunakan jika Ubuntu Server DMZ perlu akses internet, misalnya untuk install Nginx menggunakan `apt`.

```bash
config firewall policy
    edit 4
        set name "DMZ_to_WAN"
        set srcintf "port3"
        set dstintf "port1"
        set srcaddr "DMZ_Server_192.168.20.10"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

Policy ini boleh dipakai selama proses instalasi package. Setelah praktikum selesai, policy dapat dinonaktifkan jika ingin DMZ lebih dibatasi.

---

## 7.11 Pengujian FortiGate

Tes ke MikroTik:

```bash
execute ping 10.10.10.1
```

Tes ke Cisco Router:

```bash
execute ping 10.20.20.2
```

Tes ke Ubuntu Server DMZ:

```bash
execute ping 192.168.20.10
```

Cek konfigurasi:

```bash
show system interface
get router info routing-table all
show firewall address
show firewall policy
show firewall vip
```

### Screenshot FortiGate

Ambil screenshot:

* Interface FortiGate
* Routing table FortiGate
* Firewall address object
* Firewall policy
* VIP / port forwarding
* Ping FortiGate ke MikroTik, Cisco, dan DMZ

---

# 8. Konfigurasi Cisco Router

## 8.1 Tujuan

Cisco Router digunakan sebagai router internal LAN. Cisco harus:

* Terhubung ke FortiGate melalui network `10.20.20.0/30`.
* Menjadi gateway Client LAN dengan IP `192.168.10.1`.
* Memiliki default route menuju FortiGate.

---

## 8.2 Konfigurasi Cisco Router

Masuk ke mode konfigurasi:

```bash
enable
configure terminal
```

Atur hostname:

```bash
hostname Cisco-LAN
```

Konfigurasi interface ke FortiGate:

```bash
interface GigabitEthernet0/0
 description LINK_TO_FORTIGATE
 ip address 10.20.20.2 255.255.255.252
 no shutdown
 exit
```

Konfigurasi interface ke LAN:

```bash
interface GigabitEthernet0/1
 description LAN_CLIENT
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit
```

Tambahkan default route ke FortiGate:

```bash
ip route 0.0.0.0 0.0.0.0 10.20.20.1
```

Simpan konfigurasi:

```bash
end
copy running-config startup-config
```

Atau:

```bash
wr
```

---

## 8.3 Verifikasi Cisco Router

Cek interface:

```bash
show ip interface brief
```

Cek routing table:

```bash
show ip route
```

Tes ping ke FortiGate:

```bash
ping 10.20.20.1
```

Tes ping ke server DMZ:

```bash
ping 192.168.20.10
```

### Screenshot Cisco

Ambil screenshot:

* `show ip interface brief`
* `show ip route`
* Ping ke `10.20.20.1`
* Ping ke `192.168.20.10`

---

# 9. Konfigurasi Client LAN TinyCore

## 9.1 Tujuan

Client LAN digunakan untuk menguji koneksi dari jaringan internal ke gateway, FortiGate, dan server DMZ.

Client LAN harus bisa:

* Ping ke gateway Cisco.
* Ping ke FortiGate.
* Ping ke server DMZ.
* Mengakses web server DMZ.

---

## 9.2 Konfigurasi IP Client LAN

Gunakan konfigurasi berikut:

| Parameter  | Nilai              |
| ---------- | ------------------ |
| IP Address | `192.168.10.10/24` |
| Netmask    | `255.255.255.0`    |
| Gateway    | `192.168.10.1`     |
| DNS        | `8.8.8.8`          |

Jika menggunakan terminal TinyCore:

```bash
sudo ifconfig eth0 192.168.10.10 netmask 255.255.255.0 up
sudo route add default gw 192.168.10.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Jika menggunakan GUI TinyCore:

1. Masuk ke **Control Panel**.
2. Pilih **Network**.
3. Isi IP Address: `192.168.10.10`
4. Isi Subnet Mask: `255.255.255.0`
5. Isi Gateway: `192.168.10.1`
6. Isi DNS: `8.8.8.8`
7. Klik **Apply**.

---

## 9.3 Pengujian Client LAN

Ping ke gateway Cisco:

```bash
ping 192.168.10.1
```

Ping ke FortiGate:

```bash
ping 10.20.20.1
```

Ping ke server DMZ:

```bash
ping 192.168.20.10
```

Akses web server DMZ:

```bash
wget -O - http://192.168.20.10
```

Atau buka browser:

```text
http://192.168.20.10
```

### Screenshot Client LAN

Ambil screenshot:

* Konfigurasi IP Client LAN
* Ping ke `192.168.10.1`
* Ping ke `10.20.20.1`
* Ping ke `192.168.20.10`
* Akses web DMZ dari Client LAN

---

# 10. Konfigurasi Client WAN TinyCore

## 10.1 Tujuan

Client WAN digunakan untuk menguji akses dari sisi luar. Client WAN harus bisa mengakses web server DMZ melalui IP WAN FortiGate, tetapi tidak boleh langsung mengakses LAN.

---

## 10.2 Konfigurasi IP Client WAN

Gunakan konfigurasi berikut:

| Parameter  | Nilai              |
| ---------- | ------------------ |
| IP Address | `172.16.100.10/24` |
| Netmask    | `255.255.255.0`    |
| Gateway    | `172.16.100.1`     |
| DNS        | `8.8.8.8`          |

Jika menggunakan terminal TinyCore:

```bash
sudo ifconfig eth0 172.16.100.10 netmask 255.255.255.0 up
sudo route add default gw 172.16.100.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Jika menggunakan GUI TinyCore:

1. Masuk ke **Control Panel**.
2. Pilih **Network**.
3. Isi IP Address: `172.16.100.10`
4. Isi Subnet Mask: `255.255.255.0`
5. Isi Gateway: `172.16.100.1`
6. Isi DNS: `8.8.8.8`
7. Klik **Apply**.

---

## 10.3 Pengujian Client WAN

Ping ke gateway MikroTik:

```bash
ping 172.16.100.1
```

Ping ke FortiGate WAN:

```bash
ping 10.10.10.2
```

Akses web server DMZ melalui VIP FortiGate:

```bash
wget -O - http://10.10.10.2
```

Atau buka browser:

```text
http://10.10.10.2
```

Tes ping ke Client LAN:

```bash
ping 192.168.10.10
```

Tes ping ke IP asli server DMZ:

```bash
ping 192.168.20.10
```

### Hasil yang Diharapkan

| Pengujian                 | Hasil                                        |
| ------------------------- | -------------------------------------------- |
| Ping ke `172.16.100.1`    | Berhasil                                     |
| Ping ke `10.10.10.2`      | Berhasil jika allowaccess ping aktif         |
| Akses `http://10.10.10.2` | Berhasil menampilkan halaman DMZ             |
| Ping ke `192.168.10.10`   | Gagal / timeout                              |
| Ping ke `192.168.20.10`   | Gagal / timeout jika akses hanya melalui VIP |

### Screenshot Client WAN

Ambil screenshot:

* Konfigurasi IP Client WAN
* Ping ke `172.16.100.1`
* Ping ke `10.10.10.2`
* Akses web melalui `http://10.10.10.2`
* Ping gagal ke `192.168.10.10`
* Ping gagal ke `192.168.20.10`

---

# 11. Konfigurasi Ubuntu Server DMZ

## 11.1 Tujuan

Ubuntu Server DMZ digunakan sebagai web server. Server ini harus:

* Menggunakan IP statis `192.168.20.10/24`.
* Menggunakan gateway `192.168.20.1`.
* Menjalankan Nginx.
* Menampilkan halaman web sesuai format tugas.
* Bisa diakses dari Client LAN.
* Bisa diakses dari Client WAN melalui VIP FortiGate.

---

## 11.2 Cek Nama Interface Ubuntu

Cek interface:

```bash
ip a
```

Nama interface biasanya:

```text
eth0
```

atau:

```text
ens3
```

Gunakan nama interface yang muncul pada perangkat masing-masing.

---

## 11.3 Konfigurasi IP Statis Ubuntu

Buka file netplan:

```bash
nano /etc/netplan/01-netcfg.yaml
```

Contoh jika interface bernama `eth0`:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.20.10/24
      gateway4: 192.168.20.1
      nameservers:
        addresses:
          - 8.8.8.8
```

Jika interface kamu bernama `ens3`, ganti `eth0` menjadi `ens3`.

Terapkan konfigurasi:

```bash
netplan apply
```

Cek IP dan route:

```bash
ip a
ip route
```

Tes ping ke gateway FortiGate DMZ:

```bash
ping 192.168.20.1
```

---

## 11.4 Install dan Jalankan Nginx

Update package:

```bash
apt update
```

Install Nginx:

```bash
apt install -y nginx
```

Aktifkan dan jalankan Nginx:

```bash
systemctl enable nginx
systemctl start nginx
```

Cek status Nginx:

```bash
systemctl status nginx
```

---

## 11.5 Ubah Halaman Web Default

Ubah isi halaman default Nginx:

```bash
echo "Tumod_4_DMZ_Firewall_No.kel-nama" > /var/www/html/index.html
```

Contoh:

```bash
echo "Tumod_4_DMZ_Firewall_03-Kelompok3" > /var/www/html/index.html
```

Tes dari Ubuntu:

```bash
curl http://localhost
curl http://192.168.20.10
```

Jika berhasil, output akan menampilkan teks yang sudah dimasukkan.

### Screenshot Ubuntu DMZ

Ambil screenshot:

* `ip a`
* `ip route`
* `systemctl status nginx`
* `curl http://localhost`
* `curl http://192.168.20.10`

---

# 12. Penyimpanan Konfigurasi

## 12.1 Cisco Router

```bash
copy running-config startup-config
```

Atau:

```bash
wr
```

---

## 12.2 MikroTik

Export konfigurasi:

```bash
/export
```

Jika ingin menyimpan ke file:

```bash
/export file=modul4-mikrotik
```

---

## 12.3 FortiGate

FortiGate biasanya otomatis menyimpan konfigurasi setelah perintah `end`.

Cek konfigurasi:

```bash
show
```

Backup konfigurasi jika diperlukan:

```bash
execute backup config flash backup.conf
```

---

## 12.4 Ubuntu Server

Pastikan Nginx aktif otomatis:

```bash
systemctl enable nginx
systemctl status nginx
```

---

# 13. Hasil Pengujian

## 13.1 Client LAN ke Gateway Cisco

Command:

```bash
ping 192.168.10.1
```

Hasil yang diharapkan:

```text
Reply / paket berhasil diterima
```

Analisis:

Client LAN berhasil terhubung ke gateway Cisco. Artinya konfigurasi IP Client LAN dan interface LAN Cisco sudah benar.

---

## 13.2 Client LAN ke FortiGate

Command:

```bash
ping 10.20.20.1
```

Hasil yang diharapkan:

```text
Reply / paket berhasil diterima
```

Analisis:

Client LAN berhasil mencapai FortiGate melalui Cisco Router. Artinya routing dari Cisco ke FortiGate sudah berjalan.

---

## 13.3 Client LAN ke Server DMZ

Command:

```bash
ping 192.168.20.10
```

Hasil yang diharapkan:

```text
Reply / paket berhasil diterima
```

Analisis:

Client LAN berhasil mencapai server DMZ karena policy `LAN_to_DMZ` mengizinkan trafik dari LAN menuju DMZ.

---

## 13.4 Client LAN Akses Web DMZ

Command:

```bash
wget -O - http://192.168.20.10
```

Atau melalui browser:

```text
http://192.168.20.10
```

Hasil yang diharapkan:

```text
Tumod_4_DMZ_Firewall_No.kel-nama
```

Analisis:

Web server DMZ berhasil diakses dari jaringan LAN. Artinya Nginx aktif dan policy LAN ke DMZ berjalan.

---

## 13.5 Client WAN ke Gateway MikroTik

Command:

```bash
ping 172.16.100.1
```

Hasil yang diharapkan:

```text
Reply / paket berhasil diterima
```

Analisis:

Client WAN berhasil terhubung ke gateway MikroTik ISP.

---

## 13.6 Client WAN ke FortiGate WAN

Command:

```bash
ping 10.10.10.2
```

Hasil yang diharapkan:

```text
Reply / paket berhasil diterima
```

Analisis:

Client WAN berhasil mencapai interface WAN FortiGate melalui MikroTik.

---

## 13.7 Client WAN Akses Web DMZ via VIP

Command:

```bash
wget -O - http://10.10.10.2
```

Atau melalui browser:

```text
http://10.10.10.2
```

Hasil yang diharapkan:

```text
Tumod_4_DMZ_Firewall_No.kel-nama
```

Analisis:

Client WAN berhasil mengakses web server DMZ melalui IP WAN FortiGate. Artinya VIP, port forwarding, dan policy `WAN_to_DMZ_HTTP` berhasil.

---

## 13.8 Client WAN Ping Client LAN

Command:

```bash
ping 192.168.10.10
```

Hasil yang diharapkan:

```text
Timeout / gagal
```

Analisis:

Client WAN tidak dapat langsung mengakses Client LAN karena tidak ada policy dari WAN ke LAN. Ini menunjukkan LAN terlindungi dari akses langsung sisi luar.

---

## 13.9 Client WAN Ping IP Asli DMZ

Command:

```bash
ping 192.168.20.10
```

Hasil yang diharapkan:

```text
Timeout / gagal
```

Analisis:

Client WAN tidak dapat langsung mengakses IP asli DMZ. Akses dari WAN hanya dibuka melalui VIP HTTP pada IP FortiGate `10.10.10.2`.

---

# 14. Dokumentasi Screenshot

Simpan screenshot pada folder:

```text
images/
```

Contoh penamaan file:

| No | Nama File                         | Isi Screenshot                |
| -: | --------------------------------- | ----------------------------- |
|  1 | `01-topologi.png`                 | Topologi lengkap              |
|  2 | `02-mikrotik-ip-address.png`      | `/ip address print`           |
|  3 | `03-mikrotik-dhcp-client.png`     | `/ip dhcp-client print`       |
|  4 | `04-mikrotik-nat.png`             | `/ip firewall nat print`      |
|  5 | `05-mikrotik-route.png`           | `/ip route print`             |
|  6 | `06-mikrotik-ping-fortigate.png`  | Ping MikroTik ke FortiGate    |
|  7 | `07-fortigate-interface.png`      | Interface FortiGate           |
|  8 | `08-fortigate-route.png`          | Routing table FortiGate       |
|  9 | `09-fortigate-policy.png`         | Firewall policy FortiGate     |
| 10 | `10-fortigate-vip.png`            | VIP FortiGate                 |
| 11 | `11-fortigate-ping-dmz.png`       | Ping FortiGate ke DMZ         |
| 12 | `12-cisco-interface.png`          | `show ip interface brief`     |
| 13 | `13-cisco-route.png`              | `show ip route`               |
| 14 | `14-cisco-ping-fortigate.png`     | Ping Cisco ke FortiGate       |
| 15 | `15-cisco-ping-dmz.png`           | Ping Cisco ke server DMZ      |
| 16 | `16-ubuntu-ip-address.png`        | `ip a`                        |
| 17 | `17-ubuntu-route.png`             | `ip route`                    |
| 18 | `18-ubuntu-nginx-status.png`      | `systemctl status nginx`      |
| 19 | `19-ubuntu-curl-localhost.png`    | `curl http://localhost`       |
| 20 | `20-lan-ip-config.png`            | IP Client LAN                 |
| 21 | `21-lan-ping-gateway.png`         | Ping LAN ke gateway           |
| 22 | `22-lan-ping-dmz.png`             | Ping LAN ke DMZ               |
| 23 | `23-lan-access-dmz-web.png`       | Akses web DMZ dari LAN        |
| 24 | `24-wan-ip-config.png`            | IP Client WAN                 |
| 25 | `25-wan-ping-gateway.png`         | Ping WAN ke gateway           |
| 26 | `26-wan-ping-fortigate.png`       | Ping WAN ke FortiGate         |
| 27 | `27-wan-access-dmz-vip.png`       | Akses web DMZ via VIP         |
| 28 | `28-wan-ping-lan-failed.png`      | Ping WAN ke LAN gagal         |
| 29 | `29-wan-ping-real-dmz-failed.png` | Ping WAN ke IP asli DMZ gagal |

Contoh pemanggilan gambar di README:

```md
![Topologi](./images/01-topologi.png)

![Konfigurasi FortiGate](./images/07-fortigate-interface.png)

![WAN Akses DMZ via VIP](./images/27-wan-access-dmz-vip.png)
```

---

# 15. Troubleshooting

## 15.1 Client LAN Tidak Bisa Ping Cisco

Kemungkinan penyebab:

* IP Client LAN salah.
* Gateway Client LAN bukan `192.168.10.1`.
* Interface Cisco ke LAN belum `no shutdown`.
* Kabel salah interface.

Cek di Cisco:

```bash
show ip interface brief
```

---

## 15.2 Cisco Tidak Bisa Ping FortiGate

Kemungkinan penyebab:

* IP Cisco `G0/0` salah.
* IP FortiGate `port2` salah.
* Interface Cisco belum `no shutdown`.
* Kabel Cisco ke FortiGate salah.

Cek Cisco:

```bash
show ip interface brief
show ip route
```

Cek FortiGate:

```bash
show system interface port2
```

---

## 15.3 Client LAN Tidak Bisa Akses DMZ

Kemungkinan penyebab:

* FortiGate belum punya route ke LAN.
* Policy `LAN_to_DMZ` belum dibuat.
* Ubuntu Server belum menggunakan gateway `192.168.20.1`.
* Nginx belum aktif.

Cek FortiGate:

```bash
get router info routing-table all
show firewall policy
```

Cek Ubuntu:

```bash
ip route
systemctl status nginx
```

---

## 15.4 Client WAN Tidak Bisa Akses Web DMZ

Kemungkinan penyebab:

* VIP FortiGate belum dibuat.
* Policy `WAN_to_DMZ_HTTP` belum dibuat.
* Nginx di Ubuntu belum aktif.
* Client WAN gateway salah.
* MikroTik belum bisa route ke FortiGate.

Cek FortiGate:

```bash
show firewall vip
show firewall policy
```

Cek dari Ubuntu:

```bash
curl http://localhost
```

Cek dari Client WAN:

```bash
ping 172.16.100.1
ping 10.10.10.2
wget -O - http://10.10.10.2
```

---

## 15.5 Ubuntu Tidak Bisa Install Nginx

Kemungkinan penyebab:

* Ubuntu DMZ belum punya gateway.
* FortiGate belum punya policy `DMZ_to_WAN`.
* MikroTik belum NAT masquerade ke internet.
* DNS belum diset.

Cek Ubuntu:

```bash
ip route
cat /etc/resolv.conf
ping 8.8.8.8
```

Jika ping IP berhasil tetapi domain gagal, kemungkinan masalah ada pada DNS.

---

# 16. Kesimpulan

Pada tugas modul ini, jaringan berhasil dibuat dengan segmentasi WAN, LAN, dan DMZ. MikroTik ISP digunakan sebagai router luar, FortiGate digunakan sebagai firewall utama, Cisco Router digunakan sebagai router internal LAN, dan Ubuntu Server digunakan sebagai web server DMZ.

Hasil pengujian menunjukkan bahwa Client LAN dapat mengakses gateway, FortiGate, dan server DMZ. Client WAN juga dapat mengakses web server DMZ melalui IP WAN FortiGate menggunakan VIP atau port forwarding.

Selain itu, akses langsung dari WAN ke LAN dan akses langsung dari WAN ke IP asli DMZ dapat dibatasi menggunakan firewall policy. Dengan konfigurasi ini, konsep firewall, NAT, routing statis, DMZ, dan port forwarding dapat dipahami melalui simulasi jaringan di PNETLab.
