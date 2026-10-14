# Jarkom-Modul-3-2026-K-57

| | |
|---|---|
| **Nama** | : Riezco Eka Bayu Witantra & Sulthan Daffa Al Hasyimi |
| **NRP** | : 5027251057 & 5027251091 |
| **Kelompok** | : K-57 |
| **Mata Kuliah** | : JarKom |

---

## I. Ringkasan

Praktikum Modul 3 **Curtain Protocol: Shibuya Incident** membangun ulang jaringan Jujutsu High di GNS3 (project `K-57-MODUL-3`) dengan prefix IP `10.92`, domain `jujutsu.K57.com`, dan database `db_shibuya_K57`. Router **jichi** menghubungkan empat jaringan: Network 1 (DMZ), Network 2 (Divisi A), Network 3 (Divisi B), dan Network 4 (Data Center). Node server memakai image `ardhptr21/debinet:latest` dan seluruh client memakai `ardhptr21/alpinet:latest`.

| Bagian | Soal | Isi |
|---|---|---|
| Jaringan dan DHCP | 1 - 10 | IP statis, NAT, DHCP Server dan Relay, pool berbasis class, lease time, fixed address, analisis DORA, gangguan relay, autostart |
| DNS | 11 - 12 | DNS Master dan Slave, A record, CNAME, reverse DNS, forwarder |
| Database | 13 | Replikasi MariaDB Master dan Slave |
| Web, Load Balancing, dan Proxy | 14 - 20 | Worker web dan API, Weighted Round Robin, path-based routing, IP asli client, failover, cache, rate limiting, uji beban |

### Struktur Repository

| Lokasi | Isi |
|---|---|
| `README.md` | Laporan ini |
| `Script/soalN.md` | Langkah pengerjaan lengkap per soal (perintah, script, hasil, screenshot) |
| `Script/Assets/` | Screenshot dokumentasi (format nama `NoN_...png`) |

Semua script instalasi dan konfigurasi disimpan di `/root` pada node masing-masing.

---

## II. Topologi dan Pembagian IP

![Topologi](<Script/Assets/No1_topologi.png>)

| Interface jichi | Terhubung ke | Jaringan |
|---|---|---|
| `eth0` | NAT1 | internet (DHCP dari NAT) |
| `eth1` | Switch1: utahime, shoko, gojo | Network 1 `10.92.1.0/24` |
| `eth2` | Switch2 -> Switch3 (itadori, fushiguro), Switch4 (zenin, okkotsu) | Network 2 `10.92.2.0/24` |
| `eth3` | Switch5 -> Switch6 (ino, kusakabe), Switch7 (kugisaki, miwa) | Network 3 `10.92.3.0/24` |
| `eth4` | Switch8 -> Switch9 (geto, maki, panda, nanami, todo, kirara), Switch10 (tengen, yaga, inumaki) | Network 4 `10.92.4.0/24` |

| Jaringan | Node | Alamat |
|---|---|---|
| Network 1 (DMZ) | jichi `eth1` (gateway) | `10.92.1.1` |
| | utahime (DNS Master) | `10.92.1.2` |
| | shoko (DNS Slave) | `10.92.1.3` |
| | gojo (Reverse Proxy publik) | `10.92.1.4` |
| Network 2 (Divisi A) | jichi `eth2` (gateway) | `10.92.2.1` |
| | itadori, fushiguro (Pool 1) | DHCP `10.92.2.10` - `10.92.2.35` |
| | zenin, okkotsu (Pool 2) | DHCP `10.92.2.70` - `10.92.2.95` |
| Network 3 (Divisi B) | jichi `eth3` (gateway) | `10.92.3.1` |
| | ino, kusakabe (short lease 1800 detik) | DHCP `10.92.3.10` - `10.92.3.89` |
| | miwa (long lease 43200 detik) | DHCP `10.92.3.100` - `10.92.3.199` |
| | kugisaki (fixed address) | `10.92.3.99` |
| Network 4 (Data Center) | jichi `eth4` (gateway) | `10.92.4.1` |
| | tengen (DHCP Server) | `10.92.4.2` |
| | geto (Load Balancer internal) | `10.92.4.3` |
| | maki, panda (worker frontend) | `10.92.4.4`, `10.92.4.5` |
| | nanami, todo (worker API utama) | `10.92.4.6`, `10.92.4.7` |
| | yaga (DB Master) | `10.92.4.8` |
| | kirara (worker API cadangan) | `10.92.4.9` |
| | inumaki (DB Slave) | `10.92.4.11` |

---

## III. Pembahasan per Nomor Soal

### Nomor 1 - Konektivitas dasar, IP forwarding, dan NAT

jichi memperoleh WAN dari NAT lewat `eth0`, menjadi gateway `eth1` - `eth4`, serta mengaktifkan IP forwarding dan source NAT. Node infrastruktur memakai IP statis dengan `nameserver 192.168.122.1` untuk persiapan.

```bash
# jichi - /etc/network/interfaces (potongan)
auto eth0
iface eth0 inet dhcp
    up sysctl -w net.ipv4.ip_forward=1
    up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
auto eth1
iface eth1 inet static
    address 10.92.1.1
    netmask 255.255.255.0
# eth2 - eth4 sama dengan address 10.92.2.1 - 10.92.4.1
```

Hasil: host infrastruktur dapat menjangkau gateway, `8.8.8.8`, dan `google.com`.

![Uji ping](<Script/Assets/No1_test ping.png>)

Detail: [soal1.md](<Script/soal1.md>)

### Nomor 2 - DHCP Server di tengen

`isc-dhcp-server` dipasang di tengen dengan `INTERFACESv4="eth0"`. Network 4 dideklarasikan sebagai subnet lokal tanpa pool, lalu disiapkan scope untuk Network 2 dan Network 3.

```conf
subnet 10.92.4.0 netmask 255.255.255.0 {
}
subnet 10.92.2.0 netmask 255.255.255.0 {
    option routers 10.92.2.1;
    option subnet-mask 255.255.255.0;
}
```

Hasil: `dhcpd -t` valid dan service aktif tanpa error `No subnet declaration`.

Detail: [soal2.md](<Script/soal2.md>)

### Nomor 3 - DHCP Relay di jichi

`isc-dhcp-relay` di jichi meneruskan permintaan DHCP dari `eth2` (Network 2) dan `eth3` (Network 3) ke tengen.

```bash
SERVERS="10.92.4.2"
INTERFACES="eth2 eth3 eth4"
dhcrelay -i eth2 -i eth3 -i eth4 10.92.4.2
```

Hasil: client Network 2 (itadori, fushiguro) dan Network 3 (ino) memperoleh lease melalui relay.

Detail: [soal3.md](<Script/soal3.md>)

### Nomor 4 - Dua pool DHCP di Network 2 dengan DHCP class

MAC address setiap client dibaca dengan `ip link show eth0`, lalu dipakai pada `class` dan `subclass` berbasis `match hardware`. Rentang `10.92.2.36` - `10.92.2.69` tidak dialokasikan.

```conf
class "Dev"     { match hardware; }
class "Support" { match hardware; }
subclass "Dev" 1:02:42:b2:4f:49:00;       # itadori
subclass "Support" 1:02:42:54:1c:11:00;   # zenin
subnet 10.92.2.0 netmask 255.255.255.0 {
    pool { allow members of "Dev";     range 10.92.2.10 10.92.2.35; }
    pool { allow members of "Support"; range 10.92.2.70 10.92.2.95; }
}
```

Hasil: itadori dan fushiguro mendapat IP Pool 1, zenin dan okkotsu mendapat IP Pool 2.

Detail: [soal4.md](<Script/soal4.md>)

### Nomor 5 - Short lease dan long lease di Network 3

ino dan kusakabe mendapat short lease 1800 detik, miwa mendapat long lease 43200 detik. DNS `10.92.1.2` dan `10.92.1.3` diberikan ke client.

```conf
pool {
    allow members of "ShortLease";
    default-lease-time 1800;
    max-lease-time 1800;
    range 10.92.3.10 10.92.3.89;
}
pool {
    allow members of "LongLease";
    default-lease-time 43200;
    max-lease-time 43200;
    range 10.92.3.100 10.92.3.199;
}
```

Hasil: kusakabe memperoleh `10.92.3.13` dengan `lease time 1800`.

![kusakabe short lease](<Script/Assets/No5_test kusakabe.png>)

Detail: [soal5.md](<Script/soal5.md>)

### Nomor 6 - Fixed address kugisaki

```conf
host kugisaki {
    hardware ethernet 02:42:f3:87:bc:00;
    fixed-address 10.92.3.99;
}
```

Hasil: setelah renew, kugisaki selalu mendapat `10.92.3.99`, di luar seluruh pool dinamis.

Detail: [soal6.md](<Script/soal6.md>)

### Nomor 7 - Verifikasi konfigurasi seluruh client

DNS dan domain-search dipindahkan ke opsi global agar konsisten untuk Divisi A dan Divisi B. Seluruh client melakukan renew, lalu `ip a`, `ip route`, dan `/etc/resolv.conf` dicocokkan dengan `/var/lib/dhcp/dhcpd.leases` di tengen.

```conf
option domain-name-servers 10.92.1.2, 10.92.1.3;
option domain-search "jujutsu.com";
```

Hasil: setiap client berada di subnet `/24` yang benar dengan gateway dan broadcast sesuai jaringannya, tanpa alamat yang bertabrakan.

Detail: [soal7.md](<Script/soal7.md>)

### Nomor 8 - Packet capture DORA

`tcpdump -i eth0 -n -v "port 67 or port 68"` di tengen saat itadori meminta lease baru.

| Field | Nilai |
|---|---|
| Server-ID (54) | `10.92.4.2` (tengen) |
| Your-IP | `10.92.2.11` (itadori) |
| Lease-Time (51) | `600` detik |
| giaddr | `10.92.2.1` (interface `eth2` jichi) |

Hasil: nilai giaddr sesuai interface jichi yang menerima request dari Network 2.

![tcpdump di tengen](<Script/Assets/No8_tcp dump saat itadori minta baru.png>)

Detail: [soal8.md](<Script/soal8.md>)

### Nomor 9 - Gangguan dan pemulihan DHCP Relay

`pkill dhcrelay` di jichi membuat itadori gagal mendapat lease. Setelah relay dijalankan kembali, itadori memperoleh IP dari pool yang benar dengan gateway semula.

![itadori gagal mendapat lease](<Script/Assets/No9_itadori gagal ambil.png>)
![itadori pulih](<Script/Assets/No9_recovery itadori.png>)

Detail: [soal9.md](<Script/soal9.md>)

### Nomor 10 - Autostart DHCP Server dan DHCP Relay

Perintah autostart ditambahkan ke `/root/.bashrc`, yang dijalankan saat shell utama container dimulai ketika node menyala.

```bash
# tengen
service isc-dhcp-server status | grep -q "running" || bash /root/script.sh
# jichi
ps aux | grep -v grep | grep -q dhcrelay || dhcrelay -i eth2 -i eth3 -i eth4 10.92.4.2
```

Hasil: setelah restart, DHCP Server dan DHCP Relay aktif kembali, dan client Network 2 - 3 tetap mendapat alamat sesuai kebijakan pool, lease, dan fixed address.

Detail: [soal10.md](<Script/soal10.md>)

### Nomor 11 - DNS Master utahime dan DNS Slave shoko

BIND9 di utahime menjadi master zona `jujutsu.K57.com` (SOA, NS `ns1`/`ns2`, `allow-transfer` dan `also-notify` ke shoko), shoko menjadi slave.

```conf
zone "jujutsu.K57.com" {
    type master;
    file "/etc/bind/jujutsu/jujutsu.K57.com";
    allow-transfer { 10.92.1.3; };
    also-notify { 10.92.1.3; };
    notify yes;
};
```

Hasil: file zona tersalin otomatis ke shoko (303 byte). Setelah serial dinaikkan dari `2026101401` ke `2026101402`, file di shoko berubah menjadi 366 byte dan shoko menjawab serial baru serta record TXT baru dengan flag `aa`.

![Slave menjawab serial baru](<Script/Assets/No11_09_itadori-serial-check.png>)

Detail: [soal11.md](<Script/soal11.md>)

### Nomor 12 - A record, CNAME, reverse DNS, dan forwarder

```conf
@       IN  A     10.92.1.4
www     IN  CNAME jujutsu.K57.com.
4       IN  PTR   jujutsu.K57.com.      ; zona 1.92.10.in-addr.arpa
forwarders { 192.168.122.1; };
```

Hasil: dari ino, `www.jujutsu.K57.com` adalah CNAME ke `jujutsu.K57.com` (A `10.92.1.4`), `dig -x 10.92.1.4` mengembalikan `jujutsu.K57.com.`, dan `ping google.com` berhasil melalui forwarder (3/3 paket).

![CNAME dan PTR](<Script/Assets/No12_05_ino-resolv-www-ptr.png>)

Detail: [soal12.md](<Script/soal12.md>)

### Nomor 13 - Replikasi MariaDB

yaga menjadi Master (`server-id = 1`, `log-bin`), inumaki menjadi Slave (`server-id = 2`, `read_only = 1`) yang terhubung dengan `CHANGE MASTER TO` ke posisi `mysql-bin.000001` / `1160`. Database `db_shibuya_K57` dan tabel `request_logs` dibuat setelah Slave terhubung. User `shibuya@'10.92.4.%'` memberi akses remote untuk Network 4.

Hasil: `Slave_IO_Running: Yes` dan `Slave_SQL_Running: Yes`. Baris baru di yaga langsung muncul di inumaki dengan `created_at` yang sama, dan nanami membaca `COUNT(*)` berubah dari 1 menjadi 2 melalui jaringan.

![Replikasi realtime](<Script/Assets/No13_11_inumaki-realtime.png>)

Detail: [soal13.md](<Script/soal13.md>)

### Nomor 14 - Worker web dan API dengan pembatasan akses

maki dan panda menjalankan Nginx dan PHP-FPM dengan halaman yang menampilkan hostname. nanami, todo, dan kirara menjalankan API yang mencatat setiap request ke `request_logs` di yaga. Virtual host melayani `jujutsu.K57.com` dan `www.jujutsu.K57.com`.

```nginx
# maki, panda
allow 10.92.1.4;   # hanya gojo
deny all;
# nanami, todo, kirara
allow 10.92.4.3;   # hanya geto
deny all;
```

Hasil: akses dari gojo dan geto dijawab `200 OK` (API `logged: true`), akses langsung dari itadori dan kusakabe ke worker ditolak `403 Forbidden`, dan akses melalui `jujutsu.K57.com` berhasil.

![Melalui proxy 200, langsung 403](<Script/Assets/No14_10_kusakabe-proxy-vs-direct.png>)

Detail: [soal14.md](<Script/soal14.md>)

### Nomor 15 - Load balancer geto dengan Weighted Round Robin

```nginx
upstream api_cluster {
    server 10.92.4.6 weight=3;   # nanami
    server 10.92.4.7 weight=1;   # todo
}
```

Hasil: dari 20 request yang dikirim yaga ke geto, 15 dijawab nanami dan 5 dijawab todo (rasio 3:1).

![Loop 20 request](<Script/Assets/No15_05_yaga-loop-20.png>)

Detail: [soal15.md](<Script/soal15.md>)

### Nomor 16 - Reverse proxy gojo dengan path-based routing

```nginx
upstream web_frontend {
    zone web_frontend 64k;
    server 10.92.4.4;   # maki
    server 10.92.4.5;   # panda
}
location /    { proxy_pass http://web_frontend; }
location /api { proxy_pass http://10.92.4.3; }
```

Hasil: dari kusakabe, `/` dijawab bergantian maki dan panda, `www.jujutsu.K57.com` juga dilayani, dan `/api` dijawab nanami dan todo.

Detail: [soal16.md](<Script/soal16.md>)

### Nomor 17 - IP asli client dengan proxy terpercaya

gojo mengisi `X-Real-IP` dan `X-Forwarded-For`. geto hanya mempercayai `X-Real-IP` dari gojo melalui `map`, dan skrip API hanya mempercayainya jika pengirim adalah geto.

```nginx
# geto
map $remote_addr $verified_client_ip {
    10.92.1.4  $http_x_real_ip;
    default    $remote_addr;
}
```

Hasil: `request_logs` mencatat `10.92.2.11` (itadori, Network 2) dan `10.92.3.13` (kusakabe, Network 3), bukan IP gojo atau geto. Header palsu `6.6.6.6` dari client hanya muncul di `forwarded_for`, dan spoof langsung dari yaga ke geto tercatat sebagai `10.92.4.8`.

![request_logs](<Script/Assets/No17_08_yaga-spoof-and-logs.png>)

Detail: [soal17.md](<Script/soal17.md>)

### Nomor 18 - Failover ke kirara

```nginx
upstream api_cluster {
    zone api_cluster 64k;
    server 10.92.4.6 weight=3 max_fails=2 fail_timeout=5s;   # nanami
    server 10.92.4.7 weight=1 max_fails=2 fail_timeout=5s;   # todo
    server 10.92.4.9 backup;                                 # kirara
}
proxy_next_upstream error timeout http_502 http_503 http_504;
```

Hasil: saat Nginx dihentikan (uji A) maupun saat PHP-FPM dihentikan (uji B) di nanami dan todo, itadori tetap menerima `HTTP/1.1 200 OK` dari `kirara` tanpa `502 Bad Gateway`. Setelah pemulihan, trafik kembali ke nanami dan todo.

![Failover saat API mati](<Script/Assets/No18_05c_itadori-failover-php-down.png>)

Detail: [soal18.md](<Script/soal18.md>)

### Nomor 19 - Proxy cache dan rate limiting

```nginx
proxy_cache_path /var/cache/nginx/jujutsu levels=1:2 keys_zone=jujutsu_cache:10m max_size=100m inactive=10m;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
location / {
    proxy_cache jujutsu_cache;
    proxy_cache_valid 200 10m;
    add_header X-Cache-Status $upstream_cache_status always;
}
location /api {
    limit_req zone=api_limit;
    limit_req_status 429;
}
```

Hasil: akses pertama `X-Cache-Status: MISS`, akses berikutnya `HIT`. Dari 15 request beruntun ke `/api`, 13 dibalas `429 Too Many Requests` dan tercatat di error log gojo sebagai `limiting requests ... zone "api_limit"`.

![MISS lalu HIT](<Script/Assets/No19_02_itadori-cache-miss-hit.png>)

Detail: [soal19.md](<Script/soal19.md>)

### Nomor 20 - Uji beban Apache Benchmark

`ab -n 600 -c 30 -l` dari kusakabe, dengan access log khusus di gojo (`status`, `upstream`, `upstream_status`, `time`).

| | `/` | `/api/` rate limit OFF | `/api/` rate limit ON |
|---|---|---|---|
| Requests per second | 4182,09 | 700,14 | 4697,85 |
| Latensi median / terlama | 6 / 13 ms | 47 / 69 ms | 5 / 21 ms |
| Failed requests / Non-2xx | 0 / - | 0 / - | 0 / 598 |
| Log gojo 200 / 429 / 502 / 504 / 499 | 600 / 0 / 0 / 0 / 0 | 600 / 0 / 0 / 0 / 0 | 2 / 598 / 0 / 0 / 0 |

Hasil: halaman web dilayani dari cache, baseline API mencapai 700 request per detik dengan pembagian worker tepat 450:150, dan rate limit menolak 598 request di gojo dengan `429` tanpa diteruskan ke backend. Tidak ada kegagalan aplikasi (502/504) maupun koneksi (499).

Detail: [soal20.md](<Script/soal20.md>)

---

## IV. Rekap Script

| Node | Script di `/root` |
|---|---|
| jichi | `script.sh` (NAT, relay), `.bashrc` (autostart relay) |
| tengen | `script.sh` (isc-dhcp-server dan `dhcpd.conf`), `.bashrc` (autostart DHCP) |
| utahime | `soal11_utahime_install.sh`, `soal11_utahime.sh`, `soal11_utahime_serial.sh`, `soal12_utahime.sh`, `init.sh` |
| shoko | `soal11_shoko_install.sh`, `soal11_shoko.sh`, `soal12_shoko.sh`, `init.sh` |
| yaga | `soal13_yaga_install.sh`, `soal13_yaga.sh`, `soal13_yaga_db.sh`, `init.sh` |
| inumaki | `soal13_inumaki_install.sh`, `soal13_inumaki.sh`, `init.sh` |
| maki, panda | `soal14_<node>_install.sh`, `soal14_<node>.sh`, `init.sh` |
| nanami, todo, kirara | `soal14_<node>_install.sh`, `soal14_<node>.sh`, `soal17_<node>.sh`, `init.sh` (nanami juga `soal13_nanami_install.sh`) |
| geto | `soal15_geto_install.sh`, `soal15_geto.sh`, `soal17_geto.sh`, `soal18_geto.sh`, `init.sh` |
| gojo | `soal16_gojo_install.sh`, `soal16_gojo.sh`, `soal17_gojo.sh`, `soal19_gojo.sh`, `soal20_gojo_log.sh`, `soal20_ratelimit_off.sh`, `soal20_ratelimit_on.sh`, `init.sh` |

Seluruh node debinet juga memiliki `script.sh` yang menyetel `nameserver 192.168.122.1` dan menjalankan `apt-get update` saat interface aktif. `init.sh` dijalankan otomatis oleh image debinet setiap node dinyalakan untuk menyalakan kembali layanan (bind9, MariaDB, nginx, php-fpm); jika paket hilang karena container dibuat ulang, `init.sh` menjalankan kembali script instalasi dan konfigurasi soal terkait.

---

## V. Kesimpulan

Seluruh 20 soal berjalan sesuai rancangan. jichi menghubungkan empat jaringan dan meneruskan DHCP ke tengen, yang membagikan alamat berdasarkan class MAC, lease time, dan fixed address. utahime dan shoko menyediakan DNS internal dengan zone transfer otomatis dan forwarder ke internet. yaga dan inumaki menyimpan log request secara redundan. gojo dan geto membentuk jalur proxy dua lapis dengan Round Robin, Weighted Round Robin 3:1, IP asli client yang tervalidasi, failover ke kirara, cache 10 menit, dan rate limit 10 request per detik per IP. Uji beban menunjukkan kapasitas sekitar 4200 request per detik untuk halaman yang di-cache, sekitar 700 request per detik untuk jalur API penuh, dan penolakan kelebihan request di gojo tanpa membebani backend.
