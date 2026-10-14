# Soal 17 - Meneruskan IP asli client (X-Real-IP dan X-Forwarded-For)

**Tugas:** Konfigurasikan **gojo** dan **geto** agar meneruskan alamat client melalui header **X-Real-IP** dan **X-Forwarded-For**. Pastikan **geto** hanya mempercayai informasi alamat client dari proxy terpercaya, dan skrip API menyimpan alamat client yang telah diverifikasi ke kolom **client_ip** pada tabel **request_logs** di **yaga**. Uji dengan request dari client Network 2 dan Network 3 ke `/api`, lalu periksa database bahwa alamat yang tercatat adalah IP client asal, bukan IP gojo atau geto.

**Node:** gojo (10.92.1.4), geto (10.92.4.3), nanami/todo/kirara (API), yaga (10.92.4.8), inumaki (10.92.4.11), client itadori (Network 2) dan kusakabe (Network 3)

| Node | Peran | Aturan kepercayaan |
|---|---|---|
| gojo | proxy pertama, melihat client langsung | mengisi `X-Real-IP $remote_addr` dan menambah `X-Forwarded-For` |
| geto | proxy internal | memakai `X-Real-IP` hanya jika pengirimnya gojo (`10.92.1.4`), selain itu memakai alamat pengirim |
| worker API | menulis ke database | memakai `X-Real-IP` hanya jika pengirimnya geto (`10.92.4.3`) dan nilainya IP valid |

## Langkah

### 1. nanami, todo, dan kirara - skrip API dengan proxy terpercaya
Script yang sama dipakai di ketiga worker API (nama file disesuaikan). Hanya `api/index.php` yang diganti, sehingga tidak perlu restart service.

```bash
cat > /root/soal17_nanami.sh <<'END'
#!/bin/bash
# Soal 17 - API worker: store the VERIFIED client IP in request_logs.client_ip
# Only geto (10.92.4.3) is a trusted proxy; from anyone else X-Real-IP is ignored.
cat > /var/www/jujutsu/api/index.php <<'CFG'
<?php
header('Content-Type: application/json');
$worker = gethostname();
$trusted_proxy = '10.92.4.3';
$peer = $_SERVER['REMOTE_ADDR'];
$real_ip = isset($_SERVER['HTTP_X_REAL_IP']) ? $_SERVER['HTTP_X_REAL_IP'] : '';
if ($peer === $trusted_proxy && filter_var($real_ip, FILTER_VALIDATE_IP)) {
    $client_ip = $real_ip;
} else {
    $client_ip = $peer;
}
$forwarded = isset($_SERVER['HTTP_X_FORWARDED_FOR']) ? $_SERVER['HTTP_X_FORWARDED_FOR'] : '';
$logged = false;
try {
    $db = new PDO('mysql:host=10.92.4.8;dbname=db_shibuya_K57', 'shibuya', 'shibuya123');
    $sql = 'INSERT INTO request_logs (worker, client_ip, forwarded_for, method, uri) VALUES (?, ?, ?, ?, ?)';
    $st = $db->prepare($sql);
    $st->execute([$worker, $client_ip, $forwarded, $_SERVER['REQUEST_METHOD'], $_SERVER['REQUEST_URI']]);
    $logged = true;
} catch (Exception $e) {
    $logged = false;
}
echo json_encode(['worker' => $worker, 'client_ip' => $client_ip, 'forwarded_for' => $forwarded, 'logged' => $logged]) . "\n";
CFG
END
cat /root/soal17_nanami.sh
```

**nanami:**

![Input script API nanami](<Assets/No17_01a_nanami-api-input.png>)
![Isi script API nanami](<Assets/No17_01b_nanami-api-script.png>)

**todo:**

![Input script API todo](<Assets/No17_01c_todo-api-input.png>)
![Isi script API todo](<Assets/No17_01d_todo-api-script.png>)

**kirara:**

![Input script API kirara](<Assets/No17_01e_kirara-api-input.png>)
![Isi script API kirara](<Assets/No17_01f_kirara-api-script.png>)

Jalankan script, pastikan `index.php` baru sudah aktif, lalu restart PHP-FPM agar versi lama tidak dipakai dari cache (opcache):

```bash
bash /root/soal17_nanami.sh
grep -n trusted_proxy /var/www/jujutsu/api/index.php
service php8.4-fpm restart
```

![Menjalankan script di nanami](<Assets/No17_01g_nanami-api-run.png>)
![Menjalankan script di todo](<Assets/No17_01h_todo-api-run.png>)
![Menjalankan script di kirara](<Assets/No17_01i_kirara-api-run.png>)

Hasil: ketiga worker memakai skrip yang sama. `client_ip` diambil dari `X-Real-IP` hanya jika `REMOTE_ADDR` adalah geto `10.92.4.3` dan nilainya alamat IP yang valid; selain itu yang dicatat adalah alamat pengirim. Isi `X-Forwarded-For` disimpan terpisah di kolom `forwarded_for`.

### 2. geto - hanya mempercayai gojo
```bash
cat > /root/soal17_geto.sh <<'END'
#!/bin/bash
# Soal 17 - geto: pass the client address on, trusting X-Real-IP ONLY when it comes from gojo
cat > /etc/nginx/sites-available/geto-lb <<'CFG'
# gojo (10.92.1.4) is the only trusted proxy; from any other peer use the peer address itself
map $remote_addr $verified_client_ip {
    10.92.1.4  $http_x_real_ip;
    default    $remote_addr;
}

upstream api_cluster {
    zone api_cluster 64k;
    server 10.92.4.6 weight=3;   # nanami
    server 10.92.4.7 weight=1;   # todo
}

server {
    listen 80 default_server;
    server_name _;
    location /api {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $verified_client_ip;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
CFG
nginx -t
service nginx restart
END
cat /root/soal17_geto.sh
bash /root/soal17_geto.sh
```

![Input script geto](<Assets/No17_02a_geto-input.png>)
![Isi script geto](<Assets/No17_02b_geto-script.png>)
![Menjalankan script geto](<Assets/No17_02c_geto-run.png>)

Hasil: `nginx -t` berhasil dan nginx di-restart. `map` menghasilkan `$verified_client_ip` yang bernilai `X-Real-IP` dari gojo, atau alamat pengirim untuk sumber lain. Upstream `api_cluster` tetap berbobot 3:1 dan kini memakai `zone` bersama.

### 3. gojo - meneruskan header client
```bash
cat > /root/soal17_gojo.sh <<'END'
#!/bin/bash
# Soal 17 - gojo: forward the real client address with X-Real-IP and X-Forwarded-For
cat > /etc/nginx/sites-available/gojo <<'CFG'
upstream web_frontend {
    zone web_frontend 64k;
    server 10.92.4.4;   # maki
    server 10.92.4.5;   # panda
}
server {
    listen 80 default_server;
    server_name jujutsu.K57.com www.jujutsu.K57.com;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    location / {
        proxy_pass http://web_frontend;
    }
    location /api {
        proxy_pass http://10.92.4.3;
    }
}
CFG
nginx -t
service nginx restart
END
cat /root/soal17_gojo.sh
bash /root/soal17_gojo.sh
```

![Input script gojo](<Assets/No17_03a_gojo-input.png>)
![Isi script gojo](<Assets/No17_03b_gojo-script.png>)
![Menjalankan script gojo](<Assets/No17_03c_gojo-run.png>)

Hasil: `nginx -t` berhasil dan nginx di-restart. Ketiga `proxy_set_header` berada di level `server`, sehingga berlaku untuk `location /` dan `location /api`.

### 4. Troubleshoot: MariaDB `certificate is not yet valid`
Saat pengerjaan, jam server ITS dimundurkan sekitar 3,5 jam. MariaDB 11.8 membuat sertifikat TLS sendiri setiap kali service dijalankan dan client memverifikasinya secara default, sehingga sertifikat yang dibuat sebelum jam dimundurkan dianggap belum berlaku (`ERROR 2026 (HY000): TLS/SSL error: certificate is not yet valid`). `service mariadb stop` juga gagal karena memakai client `mariadb-admin`. Solusinya menghentikan proses secara langsung (SIGTERM, shutdown normal) lalu menjalankan ulang MariaDB agar sertifikat baru dibuat dengan waktu sekarang. Dilakukan di **yaga** lalu **inumaki**:

```bash
pgrep -a mariadb
pkill mariadbd-safe
pkill mysqld_safe
pkill mariadbd
sleep 5
pgrep -a mariadb
service mariadb start
mariadb -e "SELECT NOW();"
```

Pada inumaki, periksa juga replikasi:

```bash
mariadb -e "SHOW SLAVE STATUS\G" | head -14
```

![MariaDB yaga berjalan kembali](<Assets/No17_04a_yaga-mariadb-fix.png>)
![MariaDB inumaki dan status replikasi](<Assets/No17_04b_inumaki-mariadb-fix.png>)

Hasil: `SELECT NOW()` berhasil di kedua node tanpa error TLS. Replikasi tersambung kembali secara otomatis: `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes`, dan Slave kini membaca `mysql-bin.000002` (binary log baru yang dibuat yaga saat restart).

### 5. yaga - mengosongkan tabel untuk pengujian bersih
```bash
mariadb -e "TRUNCATE TABLE db_shibuya_K57.request_logs;"
mariadb -e "SELECT COUNT(*) FROM db_shibuya_K57.request_logs;"
```

![Tabel dikosongkan](<Assets/No17_05_yaga-truncate.png>)

Hasil: `COUNT(*) = 0`. Perintah `TRUNCATE` juga tereplikasi ke inumaki.

### 6. itadori (Network 2) - request biasa dan request dengan header palsu
```bash
ip -br a show eth0
cat /etc/resolv.conf
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl -H "X-Real-IP: 6.6.6.6" -H "X-Forwarded-For: 6.6.6.6" http://jujutsu.K57.com/api/
```

![Request dari Network 2](<Assets/No17_06_itadori-network2-api.png>)

Hasil: itadori (`10.92.2.11`, DNS `10.92.1.2`/`10.92.1.3`) tercatat sebagai `client_ip` `10.92.2.11` pada ketiga request, bukan IP gojo atau geto. Pada request dengan header palsu, `client_ip` tetap `10.92.2.11`; nilai `6.6.6.6` hanya muncul sebagai entri pertama `forwarded_for` (`6.6.6.6, 10.92.2.11, 10.92.1.4`).

### 7. kusakabe (Network 3)
```bash
ip -br a show eth0
curl http://jujutsu.K57.com/api/
curl http://www.jujutsu.K57.com/api/
```

![Request dari Network 3](<Assets/No17_07_kusakabe-network3-api.png>)

Hasil: kusakabe (`10.92.3.13`) tercatat sebagai `client_ip` `10.92.3.13` melalui `jujutsu.K57.com` maupun `www.jujutsu.K57.com`, dengan `forwarded_for` `10.92.3.13, 10.92.1.4`.

### 8. yaga - pengirim tidak terpercaya langsung ke geto, lalu cek database
```bash
curl -H "X-Real-IP: 6.6.6.6" http://10.92.4.3/api/
mariadb -e "SELECT id, worker, client_ip, forwarded_for, uri FROM db_shibuya_K57.request_logs;"
```

![Spoof langsung ke geto dan isi request_logs di yaga](<Assets/No17_08_yaga-spoof-and-logs.png>)

Hasil: request palsu dari yaga ke geto tercatat sebagai `client_ip` `10.92.4.8` (alamat yaga sendiri), karena geto hanya mempercayai `X-Real-IP` dari gojo. Tabel `request_logs` berisi 6 baris: 3 dari `10.92.2.11` (Network 2, termasuk request palsu), 2 dari `10.92.3.13` (Network 3), dan 1 dari `10.92.4.8`. Tidak ada baris yang mencatat IP gojo (`10.92.1.4`) atau geto (`10.92.4.3`) sebagai `client_ip`.

### 9. inumaki - data yang sama di Slave
```bash
mariadb -e "SELECT id, worker, client_ip, forwarded_for FROM db_shibuya_K57.request_logs;"
```

![request_logs di inumaki](<Assets/No17_09_inumaki-request-logs.png>)

Hasil: keenam baris dengan `client_ip` yang sama juga ada di inumaki melalui replikasi.

## Catatan
- `X-Real-IP` berisi satu alamat yang ditimpa di setiap proxy terpercaya, sehingga nilainya dipakai untuk `client_ip`. `X-Forwarded-For` adalah rantai alamat yang ditambah oleh setiap proxy; entri paling kiri dapat diisi bebas oleh client, sehingga hanya disimpan untuk audit di `forwarded_for`.
- Kepercayaan diperiksa di dua lapis: geto hanya mempercayai gojo (`map`), dan PHP hanya mempercayai geto (`$trusted_proxy`).
- Setelah `api/index.php` diganti, `service php8.4-fpm restart` memastikan PHP tidak lagi memakai versi lama dari cache (opcache).
