# Soal 15 - Load Balancer internal geto dengan Weighted Round Robin 3:1

**Tugas:** Konfigurasikan Nginx pada **geto** sebagai Load Balancer internal dengan upstream `api_cluster` yang mengarahkan trafik ke **nanami** dan **todo**. Terapkan algoritma **Weighted Round Robin** dengan bobot **3:1** (nanami menerima porsi tiga kali lebih banyak). Buktikan distribusi beban dengan loop request berulang ke geto dari jaringan lokal Data Center dan periksa rasio responnya.

**Node:** geto (10.92.4.3), nanami (10.92.4.6, bobot 3), todo (10.92.4.7, bobot 1), yaga (10.92.4.8, client uji di Network 4)

## Langkah

### 1. geto - instalasi Nginx
Klik kanan geto -> Console.

```bash
cat > /root/soal15_geto_install.sh <<'END'
# soal15 - geto: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y nginx
END
bash /root/soal15_geto_install.sh
```

![Instalasi Nginx di geto](<Assets/No15_01_geto-install.png>)

Hasil: `nginx` 1.26.3 terpasang. Baris `policy-rc.d denied execution of start` normal di container docker.

### 2. geto - konfigurasi Load Balancer
```bash
cat > /root/soal15_geto.sh <<'END'
#!/bin/bash
cat > /etc/nginx/sites-available/geto-lb <<'CFG'
upstream api_cluster {
    server 10.92.4.6 weight=3;   # nanami
    server 10.92.4.7 weight=1;   # todo
}
server {
    listen 80 default_server;
    server_name _;
    location /api {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
    }
}
CFG
ln -sf /etc/nginx/sites-available/geto-lb /etc/nginx/sites-enabled/geto-lb
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
END
cat /root/soal15_geto.sh
```

![Script Load Balancer geto](<Assets/No15_02_geto-script.png>)

```bash
bash /root/soal15_geto.sh
service nginx status
ls -l /etc/nginx/sites-enabled/
```

![Menjalankan script geto](<Assets/No15_03_geto-run.png>)

Hasil: `nginx -t` berhasil, `nginx is running`, dan `sites-enabled/` hanya berisi `geto-lb`. Upstream `api_cluster` berisi nanami dengan `weight=3` dan todo dengan `weight=1`.

### 3. yaga (Network 4) - beberapa request tunggal
Klik kanan yaga -> Console.

```bash
curl http://10.92.4.3/api/
curl http://10.92.4.3/api/
curl http://10.92.4.3/api/
curl http://10.92.4.3/api/
```

![Request tunggal ke geto](<Assets/No15_04_yaga-curl-api.png>)

Hasil: geto meneruskan request ke worker API dan membalas JSON dengan `logged: true`. Pada sampel kecil ini keempatnya dijawab nanami, karena nginx di geto berjalan dengan beberapa worker process yang masing-masing memiliki urutan round robin sendiri. Rasio bobot terlihat jelas pada jumlah request yang lebih besar (langkah 4).

### 4. yaga - loop 20 request dan hitung rasio
Hasil 20 request disimpan ke satu file, sehingga request yang sama ditampilkan dan dihitung.

```bash
for i in $(seq 1 20); do curl -s http://10.92.4.3/api/; done > /root/soal15_hasil.txt
cat /root/soal15_hasil.txt
grep -c nanami /root/soal15_hasil.txt
grep -c todo /root/soal15_hasil.txt
```

![Loop 20 request dan jumlah jawaban](<Assets/No15_05_yaga-loop-20.png>)

Hasil: dari 20 request, **15** dijawab nanami dan **5** dijawab todo, sesuai rasio Weighted Round Robin **3:1**. todo muncul kira-kira setiap request ke-4.

## Catatan
- Pengujian dilakukan dari node Data Center (yaga) langsung ke geto, sesuai soal. Dari client Network 2 dan 3 jalurnya melalui gojo (Soal 16).
- `weight=3` dan `weight=1` adalah satu-satunya perubahan algoritma; tanpa bobot, round robin akan membagi 1:1.
- Setiap request juga tercatat sebagai baris baru di `request_logs` pada yaga (melalui worker API dari Soal 14).
