# Soal 12 - A record, CNAME, Reverse DNS (PTR) dan DNS Forwarder

**Tugas:** Pada **utahime**, buat A record `jujutsu.K57.com` yang mengarah ke gojo (10.92.1.4), CNAME `www` ke domain utama, Reverse DNS (zona PTR) untuk IP gojo, dan aktifkan DNS Forwarder ke `192.168.122.1`. Uji dari client dengan `dig` (CNAME dan PTR) dan buktikan client tetap bisa `ping google.com`.

**Node:** utahime (10.92.1.2), shoko (10.92.1.3), client ino (Network 3)

## Langkah

### 1. utahime - record, zona reverse dan forwarder
Script ini menulis ulang seluruh zona forward (termasuk record `check` dari Soal 11) dengan serial `2026101403` agar shoko melakukan transfer ulang, lalu menambah zona reverse `1.92.10.in-addr.arpa`.

```bash
cat > /root/soal12_utahime.sh <<'END'
#!/bin/bash
cat > /etc/bind/named.conf.options <<'CFG'
options {
    directory "/var/cache/bind";
    dnssec-validation no;
    allow-query { any; };
    recursion yes;
    forwarders { 192.168.122.1; };
    listen-on { any; };
    listen-on-v6 { none; };
};
CFG
cat > /etc/bind/named.conf.local <<'CFG'
zone "jujutsu.K57.com" {
    type master;
    file "/etc/bind/jujutsu/jujutsu.K57.com";
    allow-transfer { 10.92.1.3; };
    also-notify { 10.92.1.3; };
    notify yes;
};
zone "1.92.10.in-addr.arpa" {
    type master;
    file "/etc/bind/jujutsu/1.92.10.in-addr.arpa";
    allow-transfer { 10.92.1.3; };
    also-notify { 10.92.1.3; };
    notify yes;
};
CFG
cat > /etc/bind/jujutsu/jujutsu.K57.com <<'CFG'
$TTL 604800
@       IN  SOA ns1.jujutsu.K57.com. root.jujutsu.K57.com. (
                2026101403 ; Serial
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800 )   ; Negative Cache TTL
@       IN  NS  ns1.jujutsu.K57.com.
@       IN  NS  ns2.jujutsu.K57.com.
@       IN  A   10.92.1.4
ns1     IN  A   10.92.1.2
ns2     IN  A   10.92.1.3
www     IN  CNAME jujutsu.K57.com.
check   IN  TXT  "serial-2026101402"
CFG
cat > /etc/bind/jujutsu/1.92.10.in-addr.arpa <<'CFG'
$TTL 604800
@       IN  SOA ns1.jujutsu.K57.com. root.jujutsu.K57.com. (
                2026101401 ; Serial
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800 )   ; Negative Cache TTL
@       IN  NS  ns1.jujutsu.K57.com.
@       IN  NS  ns2.jujutsu.K57.com.
4       IN  PTR jujutsu.K57.com.
CFG
named-checkconf
named-checkzone jujutsu.K57.com /etc/bind/jujutsu/jujutsu.K57.com
named-checkzone 1.92.10.in-addr.arpa /etc/bind/jujutsu/1.92.10.in-addr.arpa
service named restart
END
cat /root/soal12_utahime.sh
```

![Script utahime Soal 12 (awal)](<Assets/No12_01a_utahime-script-top.png>)
![Script utahime Soal 12 (akhir)](<Assets/No12_01b_utahime-script-end.png>)

```bash
bash /root/soal12_utahime.sh
service named status
ls -l /etc/bind/jujutsu/
```

![Menjalankan script utahime](<Assets/No12_02_utahime-run.png>)

Hasil: zona forward `loaded serial 2026101403` dan zona reverse `loaded serial 2026101401`, keduanya `OK`. `bind is running`, dan terdapat dua file zona: `jujutsu.K57.com` (489 byte) dan `1.92.10.in-addr.arpa` (372 byte).

### 2. shoko - Slave untuk zona reverse dan forwarder
```bash
cat > /root/soal12_shoko.sh <<'END'
#!/bin/bash
cat > /etc/bind/named.conf.options <<'CFG'
options {
    directory "/var/cache/bind";
    dnssec-validation no;
    allow-query { any; };
    recursion yes;
    forwarders { 192.168.122.1; };
    listen-on { any; };
    listen-on-v6 { none; };
};
CFG
cat > /etc/bind/named.conf.local <<'CFG'
zone "jujutsu.K57.com" {
    type slave;
    masters { 10.92.1.2; };
    file "/var/lib/bind/jujutsu.K57.com";
};
zone "1.92.10.in-addr.arpa" {
    type slave;
    masters { 10.92.1.2; };
    file "/var/lib/bind/1.92.10.in-addr.arpa";
};
CFG
named-checkconf
service named restart
END
cat /root/soal12_shoko.sh
```

![Input script shoko Soal 12](<Assets/No12_03a_shoko-script-input.png>)
![Isi script shoko Soal 12](<Assets/No12_03b_shoko-script.png>)

```bash
bash /root/soal12_shoko.sh
service named status
ls -l /var/lib/bind/
```

![Menjalankan script shoko](<Assets/No12_04_shoko-run.png>)

Hasil: `bind is running`. Zona forward `jujutsu.K57.com` (469 byte) sudah diperbarui pukul 15:39, yaitu saat utahime menaikkan serial ke `2026101403` (NOTIFY ke shoko). Zona reverse `1.92.10.in-addr.arpa` (282 byte) muncul pukul 15:43 setelah shoko dikonfigurasi sebagai Slave untuk zona tersebut. Kedua file hasil zone transfer, tidak dibuat manual.

### 3. ino (Network 3) - resolver, CNAME dan PTR
Klik kanan ino -> Console.

```bash
cat /etc/resolv.conf
dig www.jujutsu.K57.com
dig -x 10.92.1.4
```

![resolv.conf, CNAME dan PTR](<Assets/No12_05_ino-resolv-www-ptr.png>)

Hasil: ino memakai nameserver `10.92.1.2` dan `10.92.1.3` (dari DHCP tengen). `www.jujutsu.K57.com` adalah CNAME ke `jujutsu.K57.com`, yang memiliki A record `10.92.1.4` (gojo). Query `dig -x 10.92.1.4` mengembalikan PTR `4.1.92.10.in-addr.arpa -> jujutsu.K57.com.`. Semua dijawab utahime (`SERVER: 10.92.1.2#53`) dengan flag `aa`.

### 4. ino - jawaban Slave dan nama domain internet
```bash
dig @10.92.1.3 www.jujutsu.K57.com
ping -c 3 google.com
```

![Jawaban shoko dan ping google.com](<Assets/No12_06_ino-slave-ping.png>)

Hasil: shoko (`SERVER: 10.92.1.3#53`) memberi jawaban CNAME dan A yang sama dengan flag `aa`. `google.com` berhasil di-resolve (74.125.200.101) melalui forwarder `192.168.122.1` di utahime, dan ping 3 dari 3 paket berhasil (0% packet loss).

## Catatan
- Zona reverse `1.92.10.in-addr.arpa` berisi satu PTR: `4` -> `jujutsu.K57.com.` (gojo adalah 10.92.1.4).
- `forwarders { 192.168.122.1; };` dipasang di utahime dan shoko, sehingga nama di luar zona lab (misalnya `google.com`) tetap bisa di-resolve oleh client yang memakai kedua DNS server tersebut.
- Titik di akhir nama (`jujutsu.K57.com.`) wajib: tanpa titik, BIND akan menambahkan nama zona di belakangnya.
