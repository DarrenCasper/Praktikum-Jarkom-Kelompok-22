# Tugas Modul 5 - VLAN, VRRP, DHCP, GRE Tunnel, dan OSPF

## Deskripsi Singkat

Pada tugas modul ini dibuat jaringan enterprise sederhana yang terdiri dari dua site, yaitu **Jakarta** dan **Surabaya**. Kedua site dihubungkan melalui jaringan ISP menggunakan **FortiGate** sebagai firewall dan endpoint **GRE Tunnel**. Routing antar-site dilakukan menggunakan **OSPF over GRE**.

Pada sisi Jakarta digunakan VLAN 10, VLAN 20, dan VLAN 60. VLAN 60 digunakan untuk Ubuntu Server yang berfungsi sebagai **DHCP Server** dan **Web Server**. Pada sisi Surabaya digunakan VLAN 30 dan VLAN 40. VLAN 30 mendapatkan IP secara DHCP dari MikroTik Surabaya, sedangkan VLAN 40 menggunakan IP static.

## Topologi

Topologi jaringan terdiri dari perangkat Cisco Router, Cisco Switch, MikroTik, FortiGate, Ubuntu Server, dan beberapa client pada masing-masing VLAN.

![Topologi](../../assets/P2_Topologi.png)

## Konfigurasi Sisi Jakarta

Pada sisi Jakarta, switch dikonfigurasi dengan VLAN 10, VLAN 20, dan VLAN 60. VLAN 10 dan VLAN 20 digunakan untuk client, sedangkan VLAN 60 digunakan untuk Ubuntu Server.

![Switch VLAN Brief](../../assets/P2_switch_vlan_brief.png)

Trunk pada switch membawa VLAN 10, 20, dan 60 menuju Cisco Router dan MikroTik Jakarta.

![Switch Trunk](../../assets/P2_switch_interfaces_trunk.png)

Switch juga berhasil membaca MAC address Ubuntu pada VLAN 60.

![Switch MAC VLAN 60](../../assets/P2_switch_mac_vlan60.png)

Ubuntu Server berada pada VLAN 60 dan memiliki IP static `192.168.60.10/24`. Server ini digunakan sebagai DHCP Server untuk VLAN 10 dan VLAN 20, serta sebagai Web Server.

![Ubuntu IP](../../assets/P2_ubuntu_ip_show.png)

![Ubuntu Route](../../assets/P2_ubuntu_route_show.png)

![Ubuntu Ping](../../assets/P2_ubuntu_ping_all.png)

Service DHCP Server dan Nginx berhasil berjalan pada Ubuntu Server.

![Ubuntu Service](../../assets/P2_ubuntu_status_on.png)

Client VLAN 10 dan VLAN 20 berhasil mendapatkan IP dari DHCP Server Ubuntu.

![PC VLAN 10](../../assets/P2_PC1_VLAN10_ping_all.png)

![PC VLAN 20](../../assets/P2_PC2_VLAN20_ping_all.png)

Cisco Router Jakarta dikonfigurasi sebagai salah satu gateway VRRP untuk VLAN Jakarta.

![Cisco Interface Brief](../../assets/P2_cisco_interface_brief.png)

![Cisco VRRP](../../assets/P2_cisco_vrrp_brief.png)

![Cisco Route](../../assets/P2_cisco_show_ip_route.png)

![Cisco Ping](../../assets/P2_cisco_ping_all.png)

MikroTik Jakarta juga dikonfigurasi sebagai gateway VRRP dan DHCP Relay menuju Ubuntu Server.

![MikroTik IP](../../assets/P2_mikrotik_ip_show.png)

![MikroTik VLAN](../../assets/P2_mikrotik_vlan_show.png)

![MikroTik VRRP](../../assets/P2_mikrotik_vrrp_show.png)

![MikroTik DHCP](../../assets/P2_mikrotik_dhcp_all.png)

![MikroTik Ping](../../assets/P2_mikrotik_ping_all.png)

## Konfigurasi ISP

MikroTik ISP digunakan sebagai penghubung antara FortiGate Jakarta dan FortiGate Surabaya, serta sebagai jalur menuju internet.

![MikroTik ISP Address](../../assets/P2_mikrotik_isp_address_show.png)

![MikroTik ISP Route](../../assets/P2_mikrotik_isp_route_show.png)

![MikroTik ISP NAT](../../assets/P2_mikrotik_isp_firewall_nat_print.png)

![MikroTik ISP Ping](../../assets/P2_mikrotik_isp_ping_all.png)

## Konfigurasi Sisi Surabaya

Pada sisi Surabaya, digunakan VLAN 30 dan VLAN 40. VLAN 30 digunakan untuk client DHCP, sedangkan VLAN 40 digunakan untuk client static.

![Switch Surabaya VLAN Trunk](../../assets/P2_switch2_vlan_trunk.png)

MikroTik Surabaya dikonfigurasi sebagai gateway untuk VLAN 30 dan VLAN 40. MikroTik juga menjalankan DHCP Server untuk VLAN 30.

![MikroTik Surabaya IP](../../assets/P2_mikrotik2_show_ip_all.png)

![MikroTik Surabaya VLAN dan Route](../../assets/P2_mikrotik2_show_vlan_route_all.png)

![MikroTik Surabaya DHCP](../../assets/P2_mikrotik2_show_dhcp_all.png)

![MikroTik Surabaya Ping](../../assets/P2_mikrotik2_ping_all.png)

Client VLAN 30 berhasil mendapatkan IP DHCP, sedangkan client VLAN 40 menggunakan IP static dan dapat terhubung ke jaringan.

![PC VLAN 30](../../assets/P2_PC3_VLAN30_ping_all.png)

![PC VLAN 40](../../assets/P2_PC4_ping_all.png)

TinyCore pada VLAN 40 juga berhasil mengakses Web Server Jakarta.

![TinyCore Web](../../assets/P2_PC5_tinyCoreLinux_web.png)

## Konfigurasi FortiGate

FortiGate Jakarta dan FortiGate Surabaya digunakan sebagai firewall, NAT gateway, endpoint GRE Tunnel, dan router OSPF.

### FortiGate Jakarta

![FortiGate Jakarta Interface](../../assets/P2_fortigate_interface.png)

![FortiGate Jakarta Route](../../assets/P2_fortigate_route_all.png)

![FortiGate Jakarta Policy](../../assets/P2_fortigate_firewall_policy.png)

### FortiGate Surabaya

![FortiGate Surabaya Interface](../../assets/P2_fortigate2_interface_show_all.png)

![FortiGate Surabaya Policy](../../assets/P2_fortigate2_firewall_policy.png)

## GRE Tunnel dan OSPF

GRE Tunnel dibuat antara FortiGate Jakarta dan FortiGate Surabaya. Tunnel ini digunakan sebagai jalur routing antar-site. Setelah GRE aktif, OSPF dijalankan di atas tunnel tersebut agar network Jakarta dan Surabaya dapat saling bertukar route secara dinamis.

![GRE Tunnel Jakarta](../../assets/P2_fortigate_GreTunnel.png)

![GRE Tunnel Surabaya](../../assets/P2_fortigate2_GreTunnel.png)

Neighbor OSPF berhasil terbentuk antara FortiGate Jakarta dan FortiGate Surabaya.

![OSPF Neighbor Jakarta](../../assets/P2_fortigate_ospf_neighbor.png)

![OSPF Neighbor Surabaya](../../assets/P2_fortigate2_ospf_neighbor.png)

Routing table OSPF menunjukkan bahwa route dari site lawan berhasil diterima.

![OSPF Route Jakarta](../../assets/P2_fortigate_route_all_ospf.png)

![OSPF Route Surabaya](../../assets/P2_fortigate2_route_all_ospf.png)

Firewall policy untuk traffic OSPF/GRE juga sudah dikonfigurasi.

![FortiGate Policy OSPF](../../assets/P2_fortigate_firewall_policy_ospf.png)

## Pengujian Akhir

Pengujian dilakukan dengan ping antar-client, ping ke internet, serta akses Web Server Jakarta dari sisi Surabaya.

Client Jakarta berhasil terhubung ke jaringan Surabaya.

![PC Jakarta Ping Surabaya](../../assets/P2_PC1_ping_PC34.png)

Client Surabaya berhasil terhubung ke jaringan Jakarta.

![PC Surabaya Ping Jakarta](../../assets/P2_PC3_ping_PC12.png)

Client dapat melakukan ping ke internet.

![Ping All](../../assets/P2_mikrotik_ping_all.png)

Web Server Ubuntu Jakarta berhasil diakses dari TinyCore Surabaya.

![Web Server Jakarta](../../assets/P2_PC5_tinyCoreLinux_web.png)

## Kesimpulan

Konfigurasi jaringan enterprise HQ-Branch berhasil dilakukan. VLAN, trunk, VRRP, DHCP Server, NAT, GRE Tunnel, dan OSPF dapat berjalan dengan baik. Client pada sisi Jakarta dan Surabaya berhasil memperoleh koneksi sesuai kebutuhan, dapat mengakses internet, serta dapat saling berkomunikasi antar-site melalui GRE Tunnel dan OSPF.
