# Soal 13 - Replikasi MariaDB Master (yaga) dan Slave (inumaki)

**Tugas:** Konfigurasi MariaDB pada **yaga** sebagai DB-Master dan **inumaki** sebagai DB-Slave. Buat database `db_shibuya_K57` beserta tabel `request_logs` di Master, aktifkan binary logging (`log-bin`, `server-id`), lalu hubungkan inumaki sebagai Slave dengan `CHANGE MASTER TO`. Berikan hak akses remote bagi seluruh node di Network 4, lalu buktikan replikasi dengan memasukkan data baru di Master dan memeriksa hasilnya di Slave.

**Node:** yaga (10.92.4.8), inumaki (10.92.4.11), nanami (10.92.4.6, uji akses remote)

## Langkah

### 1. yaga dan inumaki - instalasi MariaDB
Klik kanan node -> Console. Pada **yaga**:

```bash
ip a show eth0
cat > /root/soal13_yaga_install.sh <<'END'
# soal13 - yaga: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y mariadb-server
END
bash /root/soal13_yaga_install.sh
```

![Instalasi MariaDB di yaga (awal)](<Assets/No13_01a_yaga-install-top.png>)
![Instalasi MariaDB di yaga (akhir)](<Assets/No13_01b_yaga-install-end.png>)

Pada **inumaki**:

```bash
ip a show eth0
cat > /root/soal13_inumaki_install.sh <<'END'
# soal13 - inumaki: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y mariadb-server
END
bash /root/soal13_inumaki_install.sh
```

![Instalasi MariaDB di inumaki (awal)](<Assets/No13_01c_inumaki-install-top.png>)
![Instalasi MariaDB di inumaki (akhir)](<Assets/No13_01d_inumaki-install-end.png>)

Hasil: yaga memakai IP `10.92.4.8` dan inumaki `10.92.4.11`. `mariadb-server` 11.8.6 terpasang di kedua node. Baris `policy-rc.d denied execution of start` normal di container docker, service dijalankan manual pada langkah berikutnya.

### 2. yaga - konfigurasi Master, user, dan posisi binary log
Database belum dibuat pada langkah ini. Database dan tabel dibuat setelah Slave terhubung (langkah 4), agar perintah `CREATE` ikut tercatat di binary log yang dibaca inumaki.

```bash
cat > /root/soal13_yaga.sh <<'END'
#!/bin/bash
cat > /etc/mysql/mariadb.conf.d/60-replication.cnf <<'CFG'
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log-bin = mysql-bin
CFG
service mariadb restart
mariadb <<'SQL'
CREATE USER IF NOT EXISTS 'repl'@'10.92.4.%' IDENTIFIED BY 'repl123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.92.4.%';
CREATE USER IF NOT EXISTS 'shibuya'@'10.92.4.%' IDENTIFIED BY 'shibuya123';
GRANT ALL PRIVILEGES ON db_shibuya_K57.* TO 'shibuya'@'10.92.4.%';
FLUSH PRIVILEGES;
SQL
END
cat /root/soal13_yaga.sh
```

![Script Master yaga](<Assets/No13_02_yaga-script.png>)

```bash
bash /root/soal13_yaga.sh
service mariadb status
mariadb -e "SHOW MASTER STATUS;"
```

![Menjalankan script dan SHOW MASTER STATUS](<Assets/No13_03_yaga-run-master-status.png>)

Hasil: MariaDB berjalan dengan versi `11.8.6-MariaDB-0+deb13u1 ... -log` (akhiran `-log` menandakan binary log aktif). `SHOW MASTER STATUS` menampilkan File `mysql-bin.000001` dan Position `1160`. Kedua nilai ini dipakai pada `CHANGE MASTER TO` di inumaki.

### 3. inumaki - konfigurasi Slave
`MASTER_LOG_FILE` dan `MASTER_LOG_POS` diisi sesuai hasil `SHOW MASTER STATUS` di yaga.

```bash
cat > /root/soal13_inumaki.sh <<'END'
#!/bin/bash
cat > /etc/mysql/mariadb.conf.d/60-replication.cnf <<'CFG'
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log = relay-bin
read_only = 1
CFG
service mariadb restart
mariadb <<'SQL'
CHANGE MASTER TO MASTER_HOST='10.92.4.8', MASTER_USER='repl', MASTER_PASSWORD='repl123', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1160, MASTER_USE_GTID=no;
START SLAVE;
SQL
END
cat /root/soal13_inumaki.sh
```

![Script Slave inumaki](<Assets/No13_04_inumaki-script.png>)

```bash
bash /root/soal13_inumaki.sh
mariadb -e "SHOW SLAVE STATUS\G"
```

![SHOW SLAVE STATUS (awal)](<Assets/No13_05a_inumaki-slave-status-top.png>)
![SHOW SLAVE STATUS (akhir)](<Assets/No13_05b_inumaki-slave-status-end.png>)

Hasil: `Slave_IO_State: Waiting for master to send event`, `Master_Host: 10.92.4.8`, `Master_User: repl`, `Master_Log_File: mysql-bin.000001`, `Read_Master_Log_Pos` dan `Exec_Master_Log_Pos` `1160`. `Slave_IO_Running: Yes` dan `Slave_SQL_Running: Yes`, `Last_IO_Error` dan `Last_SQL_Error` kosong, `Seconds_Behind_Master: 0`, `Master_Server_Id: 1`, `Using_Gtid: No`, dan `Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates`.

### 4. yaga - membuat database, tabel, dan baris pertama
Dijalankan setelah Slave terhubung, sehingga semua perintah di bawah ikut direplikasi ke inumaki.

```bash
cat > /root/soal13_yaga_db.sh <<'END'
#!/bin/bash
mariadb <<'SQL'
CREATE DATABASE IF NOT EXISTS db_shibuya_K57;
USE db_shibuya_K57;
CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    worker VARCHAR(50),
    client_ip VARCHAR(45),
    forwarded_for VARCHAR(255),
    method VARCHAR(10),
    uri VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO request_logs (worker, client_ip, forwarded_for, method, uri) VALUES ('yaga-test', '10.92.4.8', '', 'TEST', '/replication-check');
SQL
END
cat /root/soal13_yaga_db.sh
bash /root/soal13_yaga_db.sh
mariadb -e "SHOW DATABASES;"
mariadb -e "DESCRIBE db_shibuya_K57.request_logs;"
mariadb -e "SELECT * FROM db_shibuya_K57.request_logs;"
```

![Script database di yaga](<Assets/No13_06a_yaga-db-script.png>)
![Database, struktur tabel, dan baris pertama](<Assets/No13_06b_yaga-db-result.png>)

Hasil: `db_shibuya_K57` muncul di daftar database. `DESCRIBE` menampilkan 7 kolom tabel `request_logs` (`id` auto_increment primary key, `worker`, `client_ip`, `forwarded_for`, `method`, `uri`, `created_at` default `current_timestamp()`). Baris 1 `yaga-test` tersimpan dengan `created_at` `2026-10-14 15:58:43`.

### 5. inumaki - data sudah tereplikasi
Tidak ada perintah `CREATE` atau `INSERT` yang dijalankan di inumaki.

```bash
mariadb -e "SHOW DATABASES;"
mariadb -e "SELECT * FROM db_shibuya_K57.request_logs;"
```

![Database dan baris hasil replikasi](<Assets/No13_07_inumaki-replicated.png>)

Hasil: `db_shibuya_K57` dan baris 1 `yaga-test` ada di inumaki dengan `created_at` yang sama persis (`2026-10-14 15:58:43`), karena data disalin dari binary log yaga.

### 6. nanami (Network 4) - akses remote ke database
nanami belum memiliki client MariaDB, jadi client dipasang terlebih dahulu.

```bash
ip a show eth0
cat > /root/soal13_nanami_install.sh <<'END'
#!/bin/bash
apt-get update
apt-get install -y mariadb-client
END
cat /root/soal13_nanami_install.sh
bash /root/soal13_nanami_install.sh
mariadb --version
```

![Instalasi client MariaDB di nanami (awal)](<Assets/No13_08a_nanami-install-top.png>)
![Instalasi client MariaDB di nanami (akhir)](<Assets/No13_08b_nanami-install-end.png>)

```bash
mariadb -h 10.92.4.8 -u shibuya -pshibuya123 -e "SELECT COUNT(*) FROM db_shibuya_K57.request_logs;"
```

![Query remote dari nanami ke yaga](<Assets/No13_09_nanami-remote-db.png>)

Hasil: nanami (`10.92.4.6`) memasang `mariadb` client 11.8.6, lalu berhasil login ke yaga (`10.92.4.8`) melalui jaringan sebagai user `shibuya` dan membaca `COUNT(*) = 1`. Ini membuktikan `bind-address = 0.0.0.0` dan grant `shibuya@'10.92.4.%'` untuk seluruh Network 4.

### 7. yaga, inumaki, nanami - bukti replikasi realtime
Pada **yaga**, tambahkan baris baru:

```bash
mariadb db_shibuya_K57 -e "INSERT INTO request_logs (worker, uri) VALUES ('yaga-realtime', '/realtime');"
mariadb -e "SELECT id, worker, uri, created_at FROM db_shibuya_K57.request_logs;"
```

![Insert baris baru di yaga](<Assets/No13_10_yaga-insert.png>)

Langsung periksa di **inumaki**:

```bash
mariadb -e "SELECT id, worker, uri, created_at FROM db_shibuya_K57.request_logs;"
mariadb -e "SHOW SLAVE STATUS\G" | head -3
```

![Baris baru langsung ada di inumaki](<Assets/No13_11_inumaki-realtime.png>)

Lalu dari **nanami** melalui jaringan:

```bash
mariadb -h 10.92.4.8 -u shibuya -pshibuya123 -e "SELECT COUNT(*) FROM db_shibuya_K57.request_logs;"
```

![COUNT setelah insert](<Assets/No13_12_nanami-count-after.png>)

Hasil: baris 2 `yaga-realtime` (`/realtime`, `2026-10-14 16:07:00`) dibuat di yaga dan langsung muncul di inumaki dengan `created_at` yang sama, sementara `Slave_IO_State` tetap `Waiting for master to send event` dari `Master_Host: 10.92.4.8`. Dari nanami, `COUNT(*)` berubah dari 1 menjadi 2. Replikasi berjalan realtime dan data dapat diakses dari Network 4.

## Catatan
- yaga memakai `server-id = 1` dan `log-bin = mysql-bin`; inumaki memakai `server-id = 2`, `relay-log` dan `read_only = 1` agar salinan tidak diubah langsung.
- User `repl` (`REPLICATION SLAVE`) hanya untuk replikasi. User `shibuya` memiliki semua hak pada `db_shibuya_K57` dari host mana pun di `10.92.4.%` (dipakai worker API pada Soal 14).
- `MASTER_USE_GTID=no` membuat Slave mengikuti posisi file binary log (`mysql-bin.000001`, `1160`) dari `SHOW MASTER STATUS`.
- Database dibuat setelah Slave terhubung agar perintah `CREATE DATABASE` dan `CREATE TABLE` ikut tereplikasi.
