# Soal 14 - Worker Web (maki, panda) dan Worker API (nanami, todo, kirara) dengan pembatasan akses

**Tugas:** Pasang Nginx dan PHP-FPM pada **maki** dan **panda** untuk web frontend yang menampilkan hostname masing-masing. Pada **nanami**, **todo**, dan **kirara**, siapkan layanan API yang mencatat log request ke database **yaga**. Konfigurasikan virtual host untuk domain layanan. Batasi akses langsung dari klien: web worker hanya menerima trafik dari proxy web (gojo), worker API hanya menerima trafik dari **geto**. Uji bahwa akses langsung dari klien ditolak, sementara akses melalui reverse proxy berhasil.

**Node dan IP:**

| Node | IP | Peran | Hanya menerima dari |
|---|---|---|---|
| maki | 10.92.4.4 | web frontend | gojo `10.92.1.4` |
| panda | 10.92.4.5 | web frontend | gojo `10.92.1.4` |
| nanami | 10.92.4.6 | API utama | geto `10.92.4.3` |
| todo | 10.92.4.7 | API utama | geto `10.92.4.3` |
| kirara | 10.92.4.9 | API cadangan | geto `10.92.4.3` |

## Langkah

### 1. Kelima worker - instalasi paket
Klik kanan node -> Console. Pada **maki** dan **panda** (nama node disesuaikan):

```bash
cat > /root/soal14_maki_install.sh <<'END'
# soal14 - maki: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y nginx php8.4-fpm
END
bash /root/soal14_maki_install.sh
```

![Instalasi di maki dan panda (awal)](<Assets/No14_01a_maki-panda-install-top.png>)
![Instalasi di maki dan panda (akhir)](<Assets/No14_01b_maki-panda-install-end.png>)

Pada **nanami**, **todo**, dan **kirara** (nama node disesuaikan), ditambah `php8.4-mysql` untuk koneksi ke database:

```bash
cat > /root/soal14_nanami_install.sh <<'END'
# soal14 - nanami: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y nginx php8.4-fpm php8.4-mysql
END
bash /root/soal14_nanami_install.sh
```

![Instalasi di nanami dan todo (awal)](<Assets/No14_01c_nanami-todo-install-top.png>)
![Instalasi di nanami dan todo (akhir)](<Assets/No14_01d_nanami-todo-install-end.png>)
![Instalasi di kirara (awal)](<Assets/No14_01e_kirara-install-top.png>)
![Instalasi di kirara (akhir)](<Assets/No14_01f_kirara-install-end.png>)

Hasil: `nginx` dan `php8.4-fpm` (8.4.26) terpasang di kelima worker; worker API juga memiliki `php8.4-mysql` (modul `pdo_mysql` dan `mysqli`). Baris `policy-rc.d denied execution of start` normal di container docker, service dijalankan pada langkah berikutnya.

### 2. maki dan panda - web frontend
Script yang sama dipakai di **maki** dan **panda** (nama file disesuaikan). Halaman `index.php` menampilkan hostname worker yang menjawab. Hanya gojo (`10.92.1.4`) yang diizinkan, dan site `default` bawaan Debian dihapus sehingga `jujutsu` menjadi satu-satunya virtual host.

```bash
cat > /root/soal14_maki.sh <<'END'
#!/bin/bash
mkdir -p /var/www/jujutsu
cat > /var/www/jujutsu/index.php <<'CFG'
<?php
echo "Jujutsu High web frontend - served by " . gethostname() . "\n";
CFG
cat > /etc/nginx/sites-available/jujutsu <<'CFG'
server {
    listen 80 default_server;
    server_name jujutsu.K57.com www.jujutsu.K57.com;
    root /var/www/jujutsu;
    index index.php;

    # web workers only accept traffic coming through the public proxy gojo
    allow 10.92.1.4;
    deny all;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
CFG
ln -sf /etc/nginx/sites-available/jujutsu /etc/nginx/sites-enabled/jujutsu
rm -f /etc/nginx/sites-enabled/default
nginx -t
service php8.4-fpm restart
service nginx restart
END
cat /root/soal14_maki.sh
bash /root/soal14_maki.sh
service nginx status
service php8.4-fpm status
ls -l /etc/nginx/sites-enabled/
```

![Script dan hasil di maki dan panda](<Assets/No14_02_maki-panda-script-run.png>)

Hasil: di kedua node `nginx -t` menyatakan `syntax is ok` dan `test is successful`, `nginx is running` dan `php-fpm8.4 is running`. `sites-enabled/` hanya berisi `jujutsu -> /etc/nginx/sites-available/jujutsu`.

### 3. nanami, todo, dan kirara - layanan API
Script yang sama dipakai di ketiga worker API (nama file disesuaikan). `api/index.php` mencatat setiap request ke tabel `request_logs` di yaga (`10.92.4.8`) memakai user `shibuya` dari Soal 13, lalu membalas JSON berisi nama worker, `client_ip`, dan status `logged`. Hanya geto (`10.92.4.3`) yang diizinkan.

```bash
cat > /root/soal14_nanami.sh <<'END'
#!/bin/bash
mkdir -p /var/www/jujutsu/api
cat > /var/www/jujutsu/api/index.php <<'CFG'
<?php
header('Content-Type: application/json');
$worker = gethostname();
$client_ip = $_SERVER['REMOTE_ADDR'];
$forwarded = isset($_SERVER['HTTP_X_FORWARDED_FOR']) ? $_SERVER['HTTP_X_FORWARDED_FOR'] : '';
$logged = false;
try {
    $db = new PDO('mysql:host=10.92.4.8;dbname=db_shibuya_K57', 'shibuya', 'shibuya123');
    $st = $db->prepare('INSERT INTO request_logs (worker, client_ip, forwarded_for, method, uri) VALUES (?, ?, ?, ?, ?)');
    $st->execute([$worker, $client_ip, $forwarded, $_SERVER['REQUEST_METHOD'], $_SERVER['REQUEST_URI']]);
    $logged = true;
} catch (Exception $e) {
    $logged = false;
}
echo json_encode(['worker' => $worker, 'client_ip' => $client_ip, 'logged' => $logged]) . "\n";
CFG
cat > /etc/nginx/sites-available/jujutsu <<'CFG'
server {
    listen 80 default_server;
    server_name jujutsu.K57.com www.jujutsu.K57.com;
    root /var/www/jujutsu;
    index index.php;

    # API workers only accept traffic coming from the internal load balancer geto
    allow 10.92.4.3;
    deny all;

    location /api {
        try_files $uri $uri/ /api/index.php?$query_string;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
CFG
ln -sf /etc/nginx/sites-available/jujutsu /etc/nginx/sites-enabled/jujutsu
rm -f /etc/nginx/sites-enabled/default
nginx -t
service php8.4-fpm restart
service nginx restart
END
cat /root/soal14_nanami.sh
bash /root/soal14_nanami.sh
service nginx status
service php8.4-fpm status
ls -l /etc/nginx/sites-enabled/
```

**nanami:**

![Script API nanami (awal)](<Assets/No14_03a_nanami-script-top.png>)
![Script API nanami (akhir)](<Assets/No14_03b_nanami-script-end.png>)
![Menjalankan script di nanami](<Assets/No14_03c_nanami-run.png>)

**todo:**

![Script dan hasil di todo](<Assets/No14_04_todo-script-run.png>)

**kirara:**

![Script API kirara](<Assets/No14_05a_kirara-script.png>)
![Menjalankan script di kirara](<Assets/No14_05b_kirara-run.png>)

Hasil: di ketiga worker API `nginx -t` berhasil, `nginx is running` dan `php-fpm8.4 is running`, dan `sites-enabled/` hanya berisi `jujutsu`. kirara memakai API yang sama persis karena akan menjadi worker cadangan pada Soal 18.

### 4. Uji akses dari sumber yang diizinkan
Pada **gojo** (`10.92.1.4`, diizinkan oleh web worker):

```bash
curl -i http://10.92.4.4/
curl -i http://10.92.4.5/
```

![gojo mengakses maki dan panda](<Assets/No14_06_gojo-to-frontends.png>)

Hasil: keduanya `HTTP/1.1 200 OK`, dengan isi `Jujutsu High web frontend - served by maki` dan `... served by panda`.

Pada **geto** (`10.92.4.3`, diizinkan oleh worker API):

```bash
curl -i http://10.92.4.6/api/
curl -i http://10.92.4.7/api/
curl -i http://10.92.4.9/api/
```

![geto mengakses ketiga worker API](<Assets/No14_07_geto-to-api.png>)

Hasil: ketiganya `HTTP/1.1 200 OK` dengan `Content-Type: application/json`, berisi `{"worker":"nanami","client_ip":"10.92.4.3","logged":true}`, lalu `todo` dan `kirara` dengan format yang sama. `logged: true` berarti request berhasil dicatat ke database yaga.

Pada **yaga**, periksa log request:

```bash
mariadb -e "SELECT id, worker, client_ip, uri, created_at FROM db_shibuya_K57.request_logs;"
```

![Log request API di database yaga](<Assets/No14_08_yaga-api-logs.png>)

Hasil: selain baris 1-2 dari Soal 13, terdapat baris 3-5 dari `nanami`, `todo`, dan `kirara` dengan `client_ip` `10.92.4.3` (geto) dan `uri` `/api/` pada `2026-10-14 18:31:20`. Setiap request API tercatat di database.

### 5. Akses langsung dari client ditolak
Pada **itadori** (client Network 2):

```bash
curl -i http://10.92.4.4/
curl -i http://10.92.4.6/api/
```

![Akses langsung dari itadori ditolak](<Assets/No14_09_itadori-direct-denied.png>)

Hasil: web worker maki dan worker API nanami sama-sama membalas `HTTP/1.1 403 Forbidden` dari nginx, karena itadori bukan gojo atau geto.

### 6. Melalui reverse proxy berhasil, akses langsung ditolak (setelah Soal 15 dan 16)
Pada **kusakabe** (client Network 3, memakai DNS utahime/shoko):

```bash
curl -i http://jujutsu.K57.com/
curl -i http://10.92.4.4/
curl -i http://jujutsu.K57.com/api/
curl -i http://10.92.4.6/api/
```

![Melalui proxy 200, langsung 403](<Assets/No14_10_kusakabe-proxy-vs-direct.png>)

Hasil: melalui domain (gojo) web dijawab `HTTP/1.1 200 OK` (`served by panda`) dan `/api/` dijawab `200 OK` dengan JSON dari `nanami`. Akses langsung dari client yang sama ke maki (`10.92.4.4`) dan nanami (`10.92.4.6`) ditolak dengan `403 Forbidden`.

## Catatan
- Web worker: `allow 10.92.1.4;` (gojo) dan `deny all;`. Worker API: `allow 10.92.4.3;` (geto) dan `deny all;`.
- Pembatasan ini bekerja karena NAT di jichi hanya berlaku untuk `-o eth0` (Soal 1), sehingga worker melihat alamat sumber asli dari setiap koneksi.
- Request yang ditolak (403) berhenti di nginx dan tidak sampai ke PHP, sehingga tidak tercatat di `request_logs`.
