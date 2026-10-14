# Jarkom-Modul-3-2026-K-57

Laporan resmi Praktikum Komunikasi Data dan Jaringan Komputer Modul 3: **Curtain Protocol: Shibuya Incident** (DHCP, DNS, Database, Reverse Proxy, dan Load Balancing).

| | |
|---|---|
| Kelompok | K-57 |
| Anggota | (isi nama dan NRP anggota) |
| Prefix IP | `10.92` |
| Domain | `jujutsu.K57.com` |
| Database | `db_shibuya_K57` |

## Topologi

![Topologi](<Script/Assets/No1_topologi.png>)

## Pembagian IP

| Jaringan | Node | Alamat |
|---|---|---|
| NAT | jichi `eth0` | DHCP dari NAT1 |
| Network 1 (DMZ) `10.92.1.0/24` | jichi `eth1` (gateway) | `10.92.1.1` |
| | utahime (DNS Master) | `10.92.1.2` |
| | shoko (DNS Slave) | `10.92.1.3` |
| | gojo (Reverse Proxy publik) | `10.92.1.4` |
| Network 2 (Divisi A) `10.92.2.0/24` | jichi `eth2` (gateway) | `10.92.2.1` |
| | itadori, fushiguro (Pool 1) | DHCP `10.92.2.10` - `10.92.2.35` |
| | zenin, okkotsu (Pool 2) | DHCP `10.92.2.70` - `10.92.2.95` |
| Network 3 (Divisi B) `10.92.3.0/24` | jichi `eth3` (gateway) | `10.92.3.1` |
| | ino, kusakabe (short lease 1800 s) | DHCP `10.92.3.10` - `10.92.3.89` |
| | miwa (long lease 43200 s) | DHCP `10.92.3.100` - `10.92.3.199` |
| | kugisaki (fixed address) | `10.92.3.99` |
| Network 4 (Data Center) `10.92.4.0/24` | jichi `eth4` (gateway) | `10.92.4.1` |
| | tengen (DHCP Server) | `10.92.4.2` |
| | geto (Load Balancer internal) | `10.92.4.3` |
| | maki, panda (worker frontend) | `10.92.4.4`, `10.92.4.5` |
| | nanami, todo (worker API utama) | `10.92.4.6`, `10.92.4.7` |
| | yaga (DB Master) | `10.92.4.8` |
| | kirara (worker API cadangan) | `10.92.4.9` |
| | inumaki (DB Slave) | `10.92.4.11` |

## Daftar Soal

| No | Topik | Dokumentasi |
|---|---|---|
| 1 | Topologi, IP statis, IP forwarding, dan NAT di jichi | [soal1.md](<Script/soal1.md>) |
| 2 | DHCP Server isc-dhcp-server di tengen | [soal2.md](<Script/soal2.md>) |
| 3 | DHCP Relay di jichi | [soal3.md](<Script/soal3.md>) |
| 4 | Dua pool DHCP di Network 2 dengan DHCP class | [soal4.md](<Script/soal4.md>) |
| 5 | Short lease dan long lease di Network 3, DNS dan domain-search | [soal5.md](<Script/soal5.md>) |
| 6 | Fixed address untuk kugisaki | [soal6.md](<Script/soal6.md>) |
| 7 | Verifikasi konfigurasi seluruh client dan lease | [soal7.md](<Script/soal7.md>) |
| 8 | Packet capture DORA dan giaddr | [soal8.md](<Script/soal8.md>) |
| 9 | Gangguan dan pemulihan DHCP Relay | [soal9.md](<Script/soal9.md>) |
| 10 | Autostart DHCP Server dan DHCP Relay | [soal10.md](<Script/soal10.md>) |
| 11 | DNS Master utahime dan DNS Slave shoko | [soal11.md](<Script/soal11.md>) |
| 12 | A record, CNAME, reverse DNS, dan forwarder | [soal12.md](<Script/soal12.md>) |
| 13 | Replikasi MariaDB Master yaga dan Slave inumaki | [soal13.md](<Script/soal13.md>) |
| 14 | Worker web dan API dengan pembatasan akses | [soal14.md](<Script/soal14.md>) |
| 15 | Load balancer geto dengan Weighted Round Robin 3:1 | [soal15.md](<Script/soal15.md>) |
| 16 | Reverse proxy gojo dengan path-based routing | [soal16.md](<Script/soal16.md>) |
| 17 | X-Real-IP dan X-Forwarded-For dengan proxy terpercaya | [soal17.md](<Script/soal17.md>) |
| 18 | Failover ke worker cadangan kirara | [soal18.md](<Script/soal18.md>) |
| 19 | Proxy cache dan rate limiting di gojo | [soal19.md](<Script/soal19.md>) |
| 20 | Uji beban Apache Benchmark dan analisis log gojo | [soal20.md](<Script/soal20.md>) |

## Script

Semua script instalasi dan konfigurasi disimpan di `/root` pada masing-masing node (contoh: `/root/script.sh`, `/root/soal12_utahime.sh`). Node Soal 11-20 juga memiliki `/root/init.sh` yang dijalankan otomatis oleh image debinet setiap node dinyalakan, sehingga layanan (bind9, MariaDB, nginx, php-fpm) aktif kembali setelah restart.
