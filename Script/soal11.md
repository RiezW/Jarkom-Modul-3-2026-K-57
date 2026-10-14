# Soal 11 - DNS Master (utahime) dan DNS Slave (shoko) untuk jujutsu.K57.com

**Tugas:** Konfigurasi BIND9 pada **utahime** sebagai DNS Master dan **shoko** sebagai DNS Slave untuk domain `jujutsu.K57.com`. Atur SOA, NS untuk kedua server, dan `allow-transfer` dari Master ke Slave. Buktikan zona tersinkron otomatis ke shoko saat serial di utahime dinaikkan, lalu query `dig` ke kedua server dari salah satu client.

**Node:** utahime (10.92.1.2), shoko (10.92.1.3), client itadori

## Langkah

### 1. utahime dan shoko - instalasi BIND9
Klik kanan node -> Console. Pada **utahime**:

```bash
cat > /root/soal11_utahime_install.sh <<'END'
# soal11 - utahime: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y bind9 bind9-utils
END
bash /root/soal11_utahime_install.sh
```

![Instalasi bind9 di utahime (awal)](<Assets/No11_01a_utahime-install-top.png>)
![Instalasi bind9 di utahime (akhir)](<Assets/No11_01b_utahime-install-end.png>)

Pada **shoko**:

```bash
cat > /root/soal11_shoko_install.sh <<'END'
# soal11 - shoko: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y bind9 bind9-utils
END
bash /root/soal11_shoko_install.sh
```

![Instalasi bind9 di shoko (awal)](<Assets/No11_01c_shoko-install-top.png>)
![Instalasi bind9 di shoko (akhir)](<Assets/No11_01d_shoko-install-end.png>)

Hasil: `bind9`, `bind9-utils`, `dns-root-data` terpasang, `rndc.key` dibuat. Baris `policy-rc.d denied execution of start` normal di container docker (service dijalankan manual pada langkah berikutnya).

### 2. utahime - konfigurasi Master
```bash
cat > /root/soal11_utahime.sh <<'END'
#!/bin/bash
cat > /etc/bind/named.conf.options <<'CFG'
options {
    directory "/var/cache/bind";
    dnssec-validation no;
    allow-query { any; };
    recursion yes;
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
CFG
mkdir -p /etc/bind/jujutsu
cat > /etc/bind/jujutsu/jujutsu.K57.com <<'CFG'
$TTL 604800
@       IN  SOA ns1.jujutsu.K57.com. root.jujutsu.K57.com. (
                2026101401 ; Serial
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800 )   ; Negative Cache TTL
@       IN  NS  ns1.jujutsu.K57.com.
@       IN  NS  ns2.jujutsu.K57.com.
ns1     IN  A   10.92.1.2
ns2     IN  A   10.92.1.3
CFG
named-checkconf
named-checkzone jujutsu.K57.com /etc/bind/jujutsu/jujutsu.K57.com
service named restart
END
clear
cat /root/soal11_utahime.sh
```

![Script Master utahime](<Assets/No11_02_utahime-script.png>)

```bash
bash /root/soal11_utahime.sh
service named status
ls -l /etc/bind/jujutsu/
```

![Menjalankan script Master](<Assets/No11_03_utahime-run.png>)

Hasil: `zone jujutsu.K57.com/IN: loaded serial 2026101401`, `OK`, `bind is running`, file zona `jujutsu.K57.com` (391 byte, grup `bind`). Baris `rndc: connect failed ... connection refused` saat restart pertama tidak masalah karena named belum berjalan sebelumnya.

### 3. shoko - konfigurasi Slave
```bash
cat > /root/soal11_shoko.sh <<'END'
#!/bin/bash
cat > /etc/bind/named.conf.options <<'CFG'
options {
    directory "/var/cache/bind";
    dnssec-validation no;
    allow-query { any; };
    recursion yes;
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
CFG
named-checkconf
service named restart
END
clear
cat /root/soal11_shoko.sh
```

![Script Slave shoko](<Assets/No11_04_shoko-script.png>)

```bash
bash /root/soal11_shoko.sh
service named status
ls -l /var/lib/bind/
```

![Menjalankan script Slave](<Assets/No11_05_shoko-run.png>)

Hasil: `bind is running`, dan file `/var/lib/bind/jujutsu.K57.com` (303 byte, pemilik `bind`) muncul sendiri. File ini tidak dibuat manual, melainkan hasil zone transfer (AXFR) dari utahime.

### 4. itadori - query ke kedua server
Klik kanan itadori -> Console.

```bash
ip a show eth0
dig @10.92.1.2 jujutsu.K57.com SOA
dig @10.92.1.3 jujutsu.K57.com SOA
dig @10.92.1.2 jujutsu.K57.com NS
```

![dig SOA ke utahime dan shoko](<Assets/No11_06a_itadori-dig-soa.png>)
![dig SOA shoko dan NS](<Assets/No11_06b_itadori-dig-soa-ns.png>)

Hasil: itadori memakai IP `10.92.2.11` (Pool 1, dari DHCP). utahime (`SERVER: 10.92.1.2#53`) dan shoko (`SERVER: 10.92.1.3#53`) sama-sama menjawab `status: NOERROR` dengan flag `aa` (authoritative) dan serial `2026101401`. Query NS mengembalikan `ns1` dan `ns2` beserta A record `10.92.1.2` dan `10.92.1.3` di ADDITIONAL SECTION.

### 5. utahime - menaikkan serial dan menambah record
```bash
cat > /root/soal11_utahime_serial.sh <<'END'
#!/bin/bash
echo 'check   IN  TXT  "serial-2026101402"' >> /etc/bind/jujutsu/jujutsu.K57.com
sed -i 's/2026101401/2026101402/' /etc/bind/jujutsu/jujutsu.K57.com
named-checkzone jujutsu.K57.com /etc/bind/jujutsu/jujutsu.K57.com
rndc reload jujutsu.K57.com
END
cat /root/soal11_utahime_serial.sh
```

![Script kenaikan serial](<Assets/No11_07a_utahime-serial-script.png>)

```bash
bash /root/soal11_utahime_serial.sh
cat /etc/bind/jujutsu/jujutsu.K57.com
```

![Menjalankan script serial](<Assets/No11_07b_utahime-serial-run.png>)

Hasil: `loaded serial 2026101402`, `OK`, `zone reload queued`. File zona sekarang memakai serial `2026101402` dan memiliki record baru `check IN TXT "serial-2026101402"`. Script ini cukup dijalankan sekali, karena baris TXT ditambahkan dengan `>>`.

### 6. shoko dan itadori - bukti sinkronisasi otomatis
Pada **shoko** (tanpa perintah apa pun selain pengecekan):

```bash
date
ls -l /var/lib/bind/
```

![File zona di shoko diperbarui](<Assets/No11_08_shoko-zone-refreshed.png>)

Pada **itadori**:

```bash
dig @10.92.1.3 jujutsu.K57.com SOA +short
dig @10.92.1.3 check.jujutsu.K57.com TXT
```

![Slave menjawab serial baru](<Assets/No11_09_itadori-serial-check.png>)

Hasil: file zona di shoko berubah dari 303 byte (15:24) menjadi 366 byte (15:30) tanpa ada konfigurasi tambahan di shoko. shoko (`SERVER: 10.92.1.3#53`) menjawab serial `2026101402` dan record TXT `"serial-2026101402"` dengan flag `aa`. Zona berhasil tersinkron otomatis dari Master ke Slave setelah serial dinaikkan.

## Catatan
- `named.conf.options` sama di kedua node: `recursion yes`, `allow-query { any; }`, `dnssec-validation no`, IPv6 dimatikan.
- `allow-transfer { 10.92.1.3; }` hanya mengizinkan shoko mengambil zona dari utahime.
- `also-notify` dan `notify yes` membuat Master langsung memberi tahu Slave saat zona berubah, tanpa menunggu timer refresh.
- Serial memakai format `YYYYMMDDnn`. Slave hanya melakukan transfer jika serial di Master lebih besar dari salinannya.
