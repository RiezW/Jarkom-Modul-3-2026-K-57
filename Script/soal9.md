1. kill dhcp relay di jichi dulu
   ```bash
   pkill dhcrelay
   ```
   dan cek apakah sudah mati
   ```bash
   ps aux | grep dhcrelay
   ```
   ![alt text](<Assets/No9_kill dhcp jichi.png>)

2. uji gejala mati di client itadori dengan minta ip baru lagi
   ```bash
   ifdown eth0 2>/dev/null && ifup eth0
   ```
   ![alt text](<Assets/No9_itadori gagal ambil.png>)

3. nyalakan ulang jichi
   ```bash
   dhcrelay -i eth2 -i eth3 -i eth4 10.92.4.2
   ```
   cek apakah nyala
   ```bash
   ps aux | grep dhcrelay
   ```
   ![alt text](<Assets/No9_jichi jalan lagi.png>)

4. recovery client itadori
   ```bash
   ifdown eth0 2>/dev/null && ifup eth0
   ```
   ```bash
    ip a
    ```
    ```bash
    ip route
    ```
    ![alt text](<Assets/No9_recovery itadori.png>)
