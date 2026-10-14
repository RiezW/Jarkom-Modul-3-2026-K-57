# Soal 16 - Reverse Proxy publik gojo dengan Path-Based Routing

**Tugas:** Konfigurasikan Nginx pada **gojo** untuk melayani domain `jujutsu.K57.com` dan `www.jujutsu.K57.com` dengan metode **Path-Based Routing**. Request ke root (`/`) diteruskan secara Round Robin ke worker frontend (**maki** dan **panda**), sedangkan request ke `/api` diteruskan ke **geto**. Uji dari client Divisi A atau B bahwa web utama dijawab maki/panda, dan `/api` dijawab nanami/todo.

**Node:** gojo (10.92.1.4), maki (10.92.4.4), panda (10.92.4.5), geto (10.92.4.3), client kusakabe (Network 3)

| Path | Diteruskan ke | Algoritma |
|---|---|---|
| `/` | upstream `web_frontend` (maki, panda) | Round Robin |
| `/api` | geto `10.92.4.3` (lalu `api_cluster` nanami/todo) | Weighted Round Robin 3:1 (Soal 15) |

## Langkah

### 1. gojo - instalasi Nginx
Klik kanan gojo -> Console.

```bash
cat > /root/soal16_gojo_install.sh <<'END'
# soal16 - gojo: install packages
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y nginx
END
bash /root/soal16_gojo_install.sh
```

![Input script instalasi gojo](<Assets/No16_01a_gojo-install-input.png>)
![Hasil instalasi Nginx di gojo](<Assets/No16_01b_gojo-install-output.png>)

Hasil: `nginx` 1.26.3 terpasang. Baris `policy-rc.d denied execution of start` normal di container docker.

### 2. gojo - Path-Based Routing
`zone web_frontend 64k;` membuat urutan Round Robin dibagi bersama oleh semua worker process nginx, sehingga pergantian maki dan panda langsung terlihat.

```bash
cat > /root/soal16_gojo.sh <<'END'
#!/bin/bash
cat > /etc/nginx/sites-available/gojo <<'CFG'
upstream web_frontend {
    zone web_frontend 64k;
    server 10.92.4.4;   # maki
    server 10.92.4.5;   # panda
}
server {
    listen 80 default_server;
    server_name jujutsu.K57.com www.jujutsu.K57.com;

    location / {
        proxy_pass http://web_frontend;
        proxy_set_header Host $host;
    }
    location /api {
        proxy_pass http://10.92.4.3;
        proxy_set_header Host $host;
    }
}
CFG
ln -sf /etc/nginx/sites-available/gojo /etc/nginx/sites-enabled/gojo
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
END
cat /root/soal16_gojo.sh
bash /root/soal16_gojo.sh
service nginx status
ls -l /etc/nginx/sites-enabled/
```

![Script dan hasil di gojo](<Assets/No16_02_gojo-script-run.png>)

Hasil: `nginx -t` berhasil, `nginx is running`, dan `sites-enabled/` hanya berisi `gojo`. `location /` meneruskan ke upstream `web_frontend` (maki, panda), `location /api` meneruskan ke geto `10.92.4.3`.

### 3. kusakabe (Network 3) - web utama
Klik kanan kusakabe -> Console.

```bash
cat /etc/resolv.conf
curl http://jujutsu.K57.com/
curl http://jujutsu.K57.com/
curl http://jujutsu.K57.com/
curl http://jujutsu.K57.com/
curl http://www.jujutsu.K57.com/
```

![Web utama dari kusakabe](<Assets/No16_03_kusakabe-web.png>)

Hasil: kusakabe memakai DNS `10.92.1.2` dan `10.92.1.3`. Request ke `/` dijawab bergantian `served by maki`, `served by panda`, `served by maki`, `served by panda` (Round Robin), dan `www.jujutsu.K57.com` juga dilayani (`served by maki`).

### 4. kusakabe - path /api
```bash
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
curl http://jujutsu.K57.com/api/
```

![API dari kusakabe](<Assets/No16_04_kusakabe-api.png>)

Hasil: request ke `/api/` diteruskan gojo ke geto dan dijawab oleh worker API `nanami` dan `todo` (3 nanami, 1 todo) dalam format JSON dengan `logged: true`. `client_ip` masih `10.92.4.3` (geto) karena alamat asli client belum diteruskan; hal ini diperbaiki pada Soal 17.

## Catatan
- nginx memilih `location` dengan prefix terpanjang yang cocok, sehingga `/api/...` masuk ke `location /api` dan path lain masuk ke `location /`.
- `www.jujutsu.K57.com` sampai ke gojo melalui CNAME dari Soal 12 dan diterima karena tercantum di `server_name`.
- Pengujian memakai kusakabe (Network 3) karena client Network 3 memakai DNS `10.92.1.2` dan `10.92.1.3`, sehingga domain dapat di-resolve.
