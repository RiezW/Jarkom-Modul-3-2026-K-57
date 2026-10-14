1. jalankan tcp dump di tengen untuk pantau traffic
   ```bash
   tcpdump -i eth0 -n -v "port 67 or port 68"
   ```
   ![alt text](<Assets/No8_view tcp dump tengen.png>)

2. minta lease baru di salah satu client (itadori)
   ```bash
   ifdown eth0 2>/dev/null && ifup eth0
   ```
   ![alt text](<Assets/No8_itadori minta ip baru.png>)

3. bukti tcp dump tengen
   ![alt text](<Assets/No8_tcp dump saat itadori minta baru.png>)
   rankuman
   1. Urutan Pesan Protocol (DHCP DORA):
        - DHCP-Message (53): Request $\rightarrow$ Paket permintaan lease IP dikirim oleh relay menuju server
        - DHCP-Message (53): ACK $\rightarrow$ Paket konfirmasi persetujuan dari server dikirimkan kembali.
    2. Identifikasi Field giaddr (Gateway IP Address):
        - Gateway-IP: 10.92.2.1
        - Analisis: Nilai giaddr terbukti 10.92.2.1, yang merupakan IP interface eth2 pada router jichi. Ini membuktikan bahwa paket request dari client di Network 2 telah diteruskan secara valid melalui DHCP Relay jichi
    3. Alamat Server & Client IP yang Dialokasikan:
        - Server-ID (54): 10.92.4.2 (IP dari server tengen).
        - Your-IP: 10.92.2.11 (IP yang dialokasikan untuk client itadori)
        - Client-Ethernet-Address: 02:42:b2:4f:49:00 (MAC Address resmi itadori).
    4. Parameter Opsi DHCP Tambahan yang Disertakan:
        - Lease-Time (51): 600 (Durasi masa sewa 600 detik / 10 menit).
        - Subnet-Mask (1): 255.255.255.0.
        - Default-Gateway (3): 10.92.2.1.
        - Domain-Name-Server (6): 10.92.1.2, 10.92.1.3.
        - BR (28): 10.92.2.255 (Broadcast address)