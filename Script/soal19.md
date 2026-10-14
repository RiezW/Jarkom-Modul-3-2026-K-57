# Soal 19 - Proxy Cache dan Rate Limiting di gojo

**Tugas:** Konfigurasikan Nginx Proxy Caching di **gojo** untuk halaman web frontend dengan masa berlaku cache 10 menit dan header **X-Cache-Status**. Terapkan rate limiting pada path `/api` dengan batas 10 request per detik per IP client. Buktikan request ke halaman web menghasilkan **MISS** pada akses awal dan **HIT** pada akses berikutnya, lalu uji rate limiting dengan request beruntun ke `/api` dan catat status yang dihasilkan ketika batas terlampaui.

**Node:** gojo (10.92.1.4), client itadori (Network 2) dan kusakabe (Network 3)

| Direktif di gojo | Fungsi |
|---|---|
| `proxy_cache_path ... keys_zone=jujutsu_cache:10m inactive=10m` | lokasi file cache dan indeks cache 10 MB di memori |
| `proxy_cache jujutsu_cache;` + `proxy_cache_valid 200 10m;` | jawaban `200` pada `/` disimpan selama 10 menit |
| `add_header X-Cache-Status $upstream_cache_status always;` | menampilkan `MISS` / `HIT` di setiap jawaban |
| `limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;` | batas 10 request per detik untuk setiap IP client |
| `limit_req zone=api_limit;` + `limit_req_status 429;` | hanya pada `/api`; request yang melebihi batas dibalas `429 Too Many Requests` |

## Langkah

### 1. gojo - cache dan rate limit
Konfigurasi ini melanjutkan versi Soal 17 (header IP asli dan `zone` tetap ada).

```bash
cat > /root/soal19_gojo.sh <<'END'
#!/bin/bash
# Soal 19 - gojo: cache the web frontend for 10 minutes, rate-limit /api to 10 requests/second per client IP
mkdir -p /var/cache/nginx/jujutsu
cat > /etc/nginx/sites-available/gojo <<'CFG'
proxy_cache_path /var/cache/nginx/jujutsu levels=1:2 keys_zone=jujutsu_cache:10m max_size=100m inactive=10m;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

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
        proxy_cache jujutsu_cache;
        proxy_cache_valid 200 10m;
        add_header X-Cache-Status $upstream_cache_status always;
    }
    location /api {
        limit_req zone=api_limit;
        limit_req_status 429;
        proxy_pass http://10.92.4.3;
    }
}
CFG
nginx -t
service nginx restart
END
bash /root/soal19_gojo.sh
```

![Script gojo Soal 19](<Assets/No19_01a_gojo-input.png>)
![Menjalankan script gojo](<Assets/No19_01b_gojo-run.png>)

Hasil: `nginx -t` berhasil dan nginx di-restart.

### 2. itadori - MISS lalu HIT
```bash
curl -I http://jujutsu.K57.com/
sleep 1
curl -I http://jujutsu.K57.com/
```

![MISS lalu HIT](<Assets/No19_02_itadori-cache-miss-hit.png>)

Hasil: akses pertama `X-Cache-Status: MISS` (16:08:58, diambil dari backend), akses berikutnya `X-Cache-Status: HIT` (16:08:59, diambil dari cache gojo).

### 3. itadori - isi halaman dari cache dan nama www
```bash
curl http://jujutsu.K57.com/
curl http://jujutsu.K57.com/
curl -I http://www.jujutsu.K57.com/
```

![Isi dari cache dan www](<Assets/No19_03_itadori-cache-body-www.png>)

Hasil: kedua isi halaman sama (`served by maki`) karena berasal dari salinan cache, bukan bergantian Round Robin seperti di Soal 16. `www.jujutsu.K57.com` langsung `HIT` karena kedua nama memakai entri cache yang sama.

### 4. gojo - file cache
```bash
ls -R /var/cache/nginx/jujutsu
```

![File cache di gojo](<Assets/No19_04_gojo-cache-files.png>)

Hasil: satu file cache tersimpan di `/var/cache/nginx/jujutsu/9/dd/`.

### 5. kusakabe - uji rate limiting
```bash
for i in $(seq 1 15); do curl -s -i http://jujutsu.K57.com/api/ | head -1; done
```

![15 request beruntun ke /api](<Assets/No19_05_kusakabe-ratelimit.png>)

Hasil: request pertama `HTTP/1.1 200 OK`, 13 request berikutnya `HTTP/1.1 429 Too Many Requests`, dan request terakhir kembali `200 OK` setelah kuota terisi lagi.

### 6. gojo - catatan rate limit di error log
```bash
tail -n 10 /var/log/nginx/error.log
```

![limiting requests di error log](<Assets/No19_06_gojo-error-log-limiting.png>)

Hasil: setiap request yang ditolak tercatat sebagai `limiting requests, excess: ... by zone "api_limit", client: 10.92.3.13` (kusakabe).

## Catatan
- Cache hanya dipasang pada `location /`; `/api` tidak di-cache dan hanya `/api` yang memakai `limit_req`.
- `limit_req_status 429;` dipakai sebagai pengganti default 503, sehingga penolakan oleh rate limit dapat dibedakan dari kegagalan upstream (502/503) pada Soal 20.
- Tanpa `burst`, batas 10 r/s berarti satu request per 100 ms; loop `curl` jauh lebih cepat sehingga hampir semua request ditolak.
- `curl -I` (HEAD) dibalas sebelum file cache selesai ditulis, sehingga dua `curl -I` dalam detik yang sama dapat sama-sama `MISS`. `sleep 1` memberi waktu cache tersimpan. Untuk mengulang dari kondisi kosong: `rm -rf /var/cache/nginx/jujutsu/*` lalu `service nginx restart`.
- Key cache bawaan adalah `$scheme$proxy_host$request_uri`, dengan `$proxy_host` = `web_frontend`, sehingga `jujutsu.K57.com` dan `www.jujutsu.K57.com` berbagi satu entri cache.
