# Soal 18 - Failover ke worker cadangan kirara

**Tugas:** Konfigurasikan upstream `api_cluster` pada **geto** agar **kirara** menjadi server backup, serta terapkan passive health check `max_fails=2` dan `fail_timeout=5s` pada worker API utama (**nanami** dan **todo**). Pastikan kirara menjalankan API yang kompatibel. Simulasikan gangguan dengan menghentikan Nginx atau layanan API pada nanami dan todo, lalu buktikan dari client bahwa request ke `/api` dialihkan ke kirara tanpa error **502 Bad Gateway**.

**Node:** geto (10.92.4.3), nanami (10.92.4.6), todo (10.92.4.7), kirara (10.92.4.9, backup), client itadori (Network 2)

| Direktif di geto | Fungsi |
|---|---|
| `max_fails=2 fail_timeout=5s` | setelah 2 kali gagal, worker dilewati selama 5 detik (passive health check) |
| `server 10.92.4.9 backup;` | kirara hanya menerima trafik saat semua worker utama tidak tersedia |
| `proxy_next_upstream error timeout http_502 http_503 http_504;` | request yang gagal di satu worker langsung dicoba ulang ke server berikutnya |
| `proxy_connect_timeout 2s;` | worker yang mati tidak membuat client menunggu lama |

## Langkah

### 1. geto - backup server dan health check
```bash
cat > /root/soal18_geto.sh <<'END'
#!/bin/bash
# Soal 18 - geto: kirara as backup + passive health check (max_fails=2, fail_timeout=5s) on the main workers
cat > /etc/nginx/sites-available/geto-lb <<'CFG'
map $remote_addr $verified_client_ip {
    10.92.1.4  $http_x_real_ip;
    default    $remote_addr;
}

upstream api_cluster {
    zone api_cluster 64k;
    server 10.92.4.6 weight=3 max_fails=2 fail_timeout=5s;   # nanami
    server 10.92.4.7 weight=1 max_fails=2 fail_timeout=5s;   # todo
    server 10.92.4.9 backup;                                 # kirara
}

server {
    listen 80 default_server;
    server_name _;
    location /api {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $verified_client_ip;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # a dead worker (connection refused) or a worker whose PHP is down (502) -> try the next server
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_connect_timeout 2s;
    }
}
CFG
nginx -t
service nginx restart
END
bash /root/soal18_geto.sh
```

![Script geto Soal 18](<Assets/No18_01a_geto-input.png>)
![Menjalankan script geto](<Assets/No18_01b_geto-run.png>)

Hasil: `nginx -t` berhasil dan nginx di-restart. `map` dan `zone` dari Soal 17 tetap dipakai.

### 2. kirara - API cadangan siap
```bash
service nginx status
service php8.4-fpm status
grep -n trusted_proxy /var/www/jujutsu/api/index.php
```

![kirara siap](<Assets/No18_02_kirara-api-ready.png>)

Hasil: nginx dan php-fpm berjalan, dan `index.php` kirara sama dengan nanami dan todo (versi Soal 17), sehingga API cadangan kompatibel dengan endpoint utama.

### 3. itadori - kondisi normal
```bash
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
```

![Sebelum gangguan](<Assets/No18_03_itadori-before-failure.png>)

Hasil: hanya nanami dan todo yang menjawab; kirara tidak menerima trafik selama worker utama sehat.

### 4. Uji A - Nginx dihentikan di nanami dan todo
Pada **nanami** dan **todo**:

```bash
service nginx stop
service nginx status
```

![Nginx nanami dihentikan](<Assets/No18_04a_nanami-nginx-stop.png>)
![Nginx todo dihentikan](<Assets/No18_04b_todo-nginx-stop.png>)

Pada **itadori**:

```bash
curl -i http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
```

![Failover ke kirara saat Nginx mati](<Assets/No18_04c_itadori-failover-nginx-down.png>)

Pada **geto**:

```bash
tail -n 4 /var/log/nginx/error.log
```

![Error log geto](<Assets/No18_04d_geto-error-log.png>)

Hasil: `nginx is not running ... failed!` di nanami dan todo hanya berarti Nginx sudah berhenti. Dari itadori, semua request dijawab `HTTP/1.1 200 OK` dengan `"worker":"kirara"`, tanpa 502. Error log geto mencatat `connect() failed (111: Connection refused)` ke `10.92.4.6:80` dan `10.92.4.7:80`, lalu request dialihkan ke kirara.

### 5. Uji B - Nginx menyala, PHP-FPM (layanan API) dihentikan
Pada **nanami** dan **todo**:

```bash
service nginx start
service php8.4-fpm stop
service nginx status
service php8.4-fpm status
```

![PHP-FPM dihentikan di todo dan nanami](<Assets/No18_05a_todo-nanami-php-down.png>)

Pada **itadori** (4 request yang sama seperti uji A):

![Failover ke kirara saat API mati](<Assets/No18_05c_itadori-failover-php-down.png>)

Bukti di worker. Pada **nanami** dan **todo**:

```bash
grep " 502 " /var/log/nginx/access.log
```

![nanami membalas 502 ke geto](<Assets/No18_05d_nanami-502.png>)
![todo membalas 502 ke geto](<Assets/No18_05e_todo-502.png>)

Pada **kirara**:

```bash
tail -n 8 /var/log/nginx/access.log
```

![kirara melayani request failover](<Assets/No18_05f_kirara-served.png>)

Hasil: Nginx di nanami dan todo berjalan tetapi PHP-FPM mati, sehingga keduanya membalas `502` ke geto (`10.92.4.3`, 16:00:04). Client tetap menerima `HTTP/1.1 200 OK` dari `kirara`, karena `proxy_next_upstream ... http_502` membuat geto mencoba ulang request ke server berikutnya. Access log kirara menunjukkan request dari geto dengan status `200` pada uji A (15:58:18) dan uji B (16:00:04).

### 6. Pemulihan nanami dan todo
Pada **nanami** dan **todo**:

```bash
service php8.4-fpm start
service php8.4-fpm status
service nginx status
```

![nanami dan todo pulih](<Assets/No18_06a_todo-nanami-restored.png>)

Pada **itadori**:

```bash
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
```

![Kembali ke worker utama](<Assets/No18_06c_itadori-back-to-main.png>)

Hasil: php-fpm dan nginx berjalan kembali di nanami dan todo, dan trafik kembali ke worker utama (3 nanami, 1 todo); kirara kembali menjadi cadangan.

## Catatan
- Uji A gagal di level koneksi TCP (`Connection refused` di error log geto), uji B gagal di level HTTP (worker membalas `502`). Kegagalan koneksi selalu dicoba ulang, sedangkan balasan `502` hanya dicoba ulang karena `http_502` tercantum di `proxy_next_upstream`.
- Setiap worker utama hanya mencatat satu `502` untuk empat request: setelah total dua kegagalan, `max_fails=2` membuat geto melewati nanami dan todo selama `fail_timeout=5s` dan langsung mengirim request ke kirara.
- `backup` membuat kirara tidak menerima trafik sama sekali selama worker utama sehat (langkah 3 dan 6).
