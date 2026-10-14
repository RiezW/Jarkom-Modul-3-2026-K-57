# Soal 20 - Uji beban dengan Apache Benchmark dan analisis log gojo

**Tugas:** Dari terminal **kusakabe**, jalankan Apache Benchmark dengan parameter `-n 600 -c 30` menuju `http://jujutsu.K57.com/`. Catat Requests per Second (RPS), latensi, dan jumlah request yang gagal. Lakukan pengujian tambahan pada endpoint `/api` untuk mengamati efek rate limiting. Bandingkan hasil baseline dengan hasil saat rate limiting aktif, dan gunakan access log **gojo** untuk membedakan kegagalan aplikasi, koneksi, dan request yang ditolak oleh kebijakan rate limit.

**Node:** kusakabe (10.92.3.13, client uji), gojo (10.92.1.4), yaga (10.92.4.8)

| Run | Target | Pengaturan gojo | Yang diukur |
|---|---|---|---|
| 1 | `/` | cache aktif (Soal 19) | halaman web dari cache gojo |
| 2 | `/api/` | rate limit **OFF** (baseline) | jalur aplikasi penuh: gojo -> geto -> nanami/todo -> PHP -> INSERT di yaga |
| 3 | `/api/` | rate limit **ON** | beban yang sama dengan kebijakan Soal 19 |

`ab` dijalankan dengan `-l` (seperti pada materi Modul 3) agar jawaban dengan panjang berbeda (`"nanami"` vs `"todo"`) tidak dihitung sebagai request gagal.

## Langkah

### 1. gojo - access log khusus benchmark dan script toggle rate limit
Konfigurasi Soal 19 ditambah `log_format bench` yang mencatat `status`, `upstream`, `upstream_status`, dan waktu setiap request ke `/var/log/nginx/jujutsu_bench.log`.

```bash
cat > /root/soal20_gojo_log.sh <<'END'
#!/bin/bash
# Soal 20 - gojo: same config as Soal 19 plus a dedicated benchmark access log
mkdir -p /var/cache/nginx/jujutsu
cat > /etc/nginx/sites-available/gojo <<'CFG'
proxy_cache_path /var/cache/nginx/jujutsu levels=1:2 keys_zone=jujutsu_cache:10m max_size=100m inactive=10m;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
log_format bench 'client=$remote_addr status=$status upstream=$upstream_addr upstream_status=$upstream_status time=$request_time "$request"';

upstream web_frontend {
    zone web_frontend 64k;
    server 10.92.4.4;   # maki
    server 10.92.4.5;   # panda
}
server {
    listen 80 default_server;
    server_name jujutsu.K57.com www.jujutsu.K57.com;
    access_log /var/log/nginx/jujutsu_bench.log bench;

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
cat > /root/soal20_ratelimit_off.sh <<'END'
#!/bin/bash
sed -i 's/        limit_req zone=api_limit;/        # limit_req zone=api_limit;/' /etc/nginx/sites-available/gojo
nginx -t
service nginx reload
END
cat > /root/soal20_ratelimit_on.sh <<'END'
#!/bin/bash
sed -i 's/        # limit_req zone=api_limit;/        limit_req zone=api_limit;/' /etc/nginx/sites-available/gojo
nginx -t
service nginx reload
END
```

![Input script log benchmark](<Assets/No20_01a_gojo-log-input.png>)
![Isi script log benchmark](<Assets/No20_01b_gojo-log-script.png>)
![Input script rate limit off](<Assets/No20_01d_ratelimit-off-input.png>)
![Input script rate limit on](<Assets/No20_01e_ratelimit-on-input.png>)
![Isi script rate limit off](<Assets/No20_01f_ratelimit-off-script.png>)
![Isi script rate limit on](<Assets/No20_01g_ratelimit-on-script.png>)

Catatan: pada saat paste, baris `X-Forwarded-For` sempat terpotong (No20_01a); baris diperbaiki dengan `sed` sehingga isi script menjadi seperti No20_01b.

```bash
bash /root/soal20_gojo_log.sh
grep limit_req /etc/nginx/sites-available/gojo
ls -l /var/log/nginx/
```

![Menjalankan script log benchmark](<Assets/No20_01c_gojo-run.png>)

Hasil: `nginx -t` berhasil, rate limit aktif, dan file `jujutsu_bench.log` tersedia.

### 2. kusakabe - instalasi Apache Benchmark
```bash
apk add apache2-utils
ab -V
```

![Instalasi ab](<Assets/No20_02_kusakabe-install-ab.png>)

Hasil: `ApacheBench, Version 2.3` terpasang dari paket `apache2-utils 2.4.69`.

### 3. Run 1 - halaman web `/`
Pada **gojo**, kosongkan log:

```bash
truncate -s 0 /var/log/nginx/jujutsu_bench.log
wc -l /var/log/nginx/jujutsu_bench.log
```

![Log dikosongkan](<Assets/No20_03a_run1-gojo-truncate.png>)

Pada **kusakabe**:

```bash
ab -n 600 -c 30 -l http://jujutsu.K57.com/
```

![ab run 1](<Assets/No20_03c_run1-ab-summary.png>)
![Persentil run 1](<Assets/No20_03d_run1-ab-percentiles.png>)

Pada **gojo**, hitung status:

```bash
wc -l /var/log/nginx/jujutsu_bench.log
grep -c "status=200 " /var/log/nginx/jujutsu_bench.log
grep -c "status=429 " /var/log/nginx/jujutsu_bench.log
grep -c "status=502 " /var/log/nginx/jujutsu_bench.log
grep -c "status=504 " /var/log/nginx/jujutsu_bench.log
grep -c "status=499 " /var/log/nginx/jujutsu_bench.log
tail -n 3 /var/log/nginx/jujutsu_bench.log
```

![Hitungan status run 1](<Assets/No20_03e_run1-gojo-counts.png>)

Hasil: 600 request, semua `status=200` dengan `upstream=-` dan `time=0.000`, artinya seluruh request dilayani dari cache gojo.

### 4. Run 2 - `/api/` dengan rate limit OFF (baseline)
Pada **gojo** dan **yaga**:

```bash
bash /root/soal20_ratelimit_off.sh
grep limit_req /etc/nginx/sites-available/gojo
truncate -s 0 /var/log/nginx/jujutsu_bench.log
```

```bash
mariadb -e "TRUNCATE TABLE db_shibuya_K57.request_logs;"
```

![Rate limit dimatikan](<Assets/No20_04a_gojo-ratelimit-off.png>)
![request_logs dikosongkan](<Assets/No20_04a2_yaga-truncate.png>)

Pada **kusakabe**:

```bash
ab -n 600 -c 30 -l http://jujutsu.K57.com/api/
```

![ab run 2](<Assets/No20_04b_run2-api-nolimit-ab.png>)
![Persentil run 2](<Assets/No20_04b2_run2-ab-percentiles.png>)

Pada **gojo** (perintah hitung yang sama dengan run 1) dan **yaga**:

```bash
mariadb -e "SELECT worker, COUNT(*) FROM db_shibuya_K57.request_logs GROUP BY worker;"
```

![Hitungan status run 2](<Assets/No20_04c_run2-gojo-counts.png>)
![Pembagian worker run 2](<Assets/No20_04d_run2-yaga-worker-split.png>)

Hasil: 600 request `status=200`, semuanya diteruskan ke geto (`upstream=10.92.4.3:80 upstream_status=200`). Di database tercatat nanami 450 dan todo 150, tepat 3:1 sesuai Weighted Round Robin Soal 15.

### 5. Run 3 - `/api/` dengan rate limit ON
Pada **gojo**:

```bash
bash /root/soal20_ratelimit_on.sh
grep limit_req /etc/nginx/sites-available/gojo
truncate -s 0 /var/log/nginx/jujutsu_bench.log
```

![Rate limit diaktifkan](<Assets/No20_05a_gojo-ratelimit-on.png>)

Pada **kusakabe**:

```bash
ab -n 600 -c 30 -l http://jujutsu.K57.com/api/
```

![ab run 3](<Assets/No20_05b_run3-api-limit-ab.png>)
![Persentil run 3](<Assets/No20_05b2_run3-ab-percentiles.png>)

Pada **gojo** (perintah hitung yang sama):

![Hitungan status run 3](<Assets/No20_05c_run3-gojo-counts.png>)

Hasil: 2 request `status=200` dan 598 request `status=429` dengan `upstream=-` dan `time=0.000`, artinya ditolak langsung oleh gojo tanpa diteruskan ke geto. Rate limit tetap ON setelah pengujian, sesuai Soal 19.

## Hasil

| Metrik | Run 1: `/` | Run 2: `/api/` limit OFF (baseline) | Run 3: `/api/` limit ON |
|---|---|---|---|
| Time taken for tests | 0,143 s | 0,857 s | 0,128 s |
| Complete requests | 600 | 600 | 600 |
| Failed requests (ab) | 0 | 0 | 0 |
| Non-2xx responses (ab) | - | - | 598 |
| **Requests per second** | **4182,09** | **700,14** | **4697,85** |
| Time per request (mean) | 7,173 ms | 42,849 ms | 6,386 ms |
| Latensi median / 99% / terlama | 6 / 12 / 13 ms | 47 / 62 / 69 ms | 5 / 20 / 21 ms |
| Log gojo 200 / 429 / 502 / 504 / 499 | 600 / 0 / 0 / 0 / 0 | 600 / 0 / 0 / 0 / 0 | 2 / 598 / 0 / 0 / 0 |
| Sampai ke backend? | tidak (`upstream=-`, cache) | ya (`upstream=10.92.4.3:80`) | hanya 2 request |

## Analisis

- **Run 1 (web):** semua request dilayani dari cache Soal 19, sehingga maki dan panda tidak menerima request sama sekali. RPS tinggi (4182) dan latensi terlama hanya 13 ms.
- **Run 2 (baseline API):** inilah kapasitas sebenarnya jalur aplikasi. Setiap request melewati tiga proxy, PHP, dan satu INSERT ke database, sehingga RPS turun menjadi 700 dan latensi median naik menjadi 47 ms. Tidak ada request yang gagal, dan pembagian beban tetap tepat 3:1 (450:150) di bawah 30 koneksi bersamaan.
- **Run 3 (API dengan rate limit):** RPS yang tinggi (4698) bukan karena aplikasi lebih cepat, melainkan karena 598 dari 600 request ditolak di gojo dengan `429` dalam waktu hampir 0 ms. Hanya 2 request yang sampai ke backend (terlihat sebagai ekor latensi 19-21 ms). Kebijakan 10 request per detik per IP melindungi geto, worker, dan database dari lonjakan request dari satu client.

### Membedakan jenis kegagalan dengan log gojo

| Jenis kegagalan | Tanda di log `bench` gojo | Tanda di ab | Jumlah pada pengujian ini |
|---|---|---|---|
| Ditolak kebijakan rate limit | `status=429 upstream=-` (dijawab gojo sendiri) | `Non-2xx responses` | 598 (run 3) |
| Kegagalan aplikasi / upstream | `status=502` atau `504` dengan `upstream=` dan `upstream_status=` terisi | `Non-2xx responses` | 0 |
| Kegagalan koneksi | `status=499` (client menutup koneksi) atau tidak ada baris sama sekali | `Failed requests` (Connect / Receive / Exceptions) | 0 |

`Failed requests: 0` pada ab sementara `Non-2xx responses: 598` menunjukkan bahwa 429 adalah jawaban HTTP yang lengkap, bukan kegagalan koneksi. Log gojo kemudian memastikan bahwa jawaban non-2xx tersebut berasal dari kebijakan rate limit (`429`, tanpa upstream), bukan dari kegagalan aplikasi (`502`/`504`).

## Kesimpulan
- Melalui jalur proxy, sistem melayani 600/600 request web dan 600/600 request API tanpa kegagalan aplikasi maupun koneksi.
- Rate limit membatasi setiap IP client pada 10 request API per detik. Kelebihan request ditolak di gojo dengan biaya hampir nol, sehingga lapisan di belakangnya tidak terbebani.
- Jika client sah perlu mengirim lonjakan singkat, `limit_req zone=api_limit burst=20 nodelay;` dapat dipakai untuk mengizinkan burst kecil dengan rata-rata tetap 10 r/s. Hal ini tidak dikonfigurasi karena soal meminta batas 10 request per detik.
