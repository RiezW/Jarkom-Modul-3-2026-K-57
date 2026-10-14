1. update tengen <br>
   Divisi A dan Divisi B sama-sama mendapatkan DNS Server 10.92.1.2, 10.92.1.3 dan domain-search secara konsisten
   ```bash
   nano /root/script.sh
   ```
   ```bash
   #!/bin/bash

    # Setup DNS resolver sementara untuk tengen
    echo "nameserver 192.168.122.1" > /etc/resolv.conf

    # Update & install isc-dhcp-server
    apt-get update
    apt-get install isc-dhcp-server -y

    # Atur interface listening
    echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
    echo 'INTERFACESv6=""' >> /etc/default/isc-dhcp-server

    # Clean lease lama
    service isc-dhcp-server stop
    echo "" > /var/lib/dhcp/dhcpd.leases

    # Konfigurasi Lengkap dhcpd.conf (Soal 1 - Soal 7)
    echo 'default-lease-time 600;' > /etc/dhcp/dhcpd.conf
    echo 'max-lease-time 3600;' >> /etc/dhcp/dhcpd.conf
    echo 'option domain-name-servers 10.92.1.2, 10.92.1.3;' >> /etc/dhcp/dhcpd.conf
    echo 'option domain-search "jujutsu.com";' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '# SOAL 4: Network 2 (Divisi A)' >> /etc/dhcp/dhcpd.conf
    echo 'class "Dev" {' >> /etc/dhcp/dhcpd.conf
    echo '    match hardware;' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'class "Support" {' >> /etc/dhcp/dhcpd.conf
    echo '    match hardware;' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "Dev" 1:02:42:b2:4f:49:00;' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "Dev" 1:f2:dd:ec:51:c9:cb;' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "Support" 1:02:42:54:1c:11:00;' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "Support" 1:02:42:0d:ec:d7:00;' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '# SOAL 5: Network 3 (Short & Long Lease)' >> /etc/dhcp/dhcpd.conf
    echo 'class "ShortLease" {' >> /etc/dhcp/dhcpd.conf
    echo '    match hardware;' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'class "LongLease" {' >> /etc/dhcp/dhcpd.conf
    echo '    match hardware;' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "ShortLease" 1:02:42:bb:de:44:00;' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "ShortLease" 1:02:42:b0:6e:76:00;' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'subclass "LongLease" 1:02:42:ce:d3:ac:00;' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '# Network 4 (Local Tengen)' >> /etc/dhcp/dhcpd.conf
    echo 'subnet 10.92.4.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '# Subnet Network 2 (Divisi A)' >> /etc/dhcp/dhcpd.conf
    echo 'subnet 10.92.2.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '    option routers 10.92.2.1;' >> /etc/dhcp/dhcpd.conf
    echo '    option broadcast-address 10.92.2.255;' >> /etc/dhcp/dhcpd.conf
    echo '    pool {' >> /etc/dhcp/dhcpd.conf
    echo '        allow members of "Dev";' >> /etc/dhcp/dhcpd.conf
    echo '        range 10.92.2.10 10.92.2.35;' >> /etc/dhcp/dhcpd.conf
    echo '    }' >> /etc/dhcp/dhcpd.conf
    echo '    pool {' >> /etc/dhcp/dhcpd.conf
    echo '        allow members of "Support";' >> /etc/dhcp/dhcpd.conf
    echo '        range 10.92.2.70 10.92.2.95;' >> /etc/dhcp/dhcpd.conf
    echo '    }' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '# Subnet Network 3 (Divisi B)' >> /etc/dhcp/dhcpd.conf
    echo 'subnet 10.92.3.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '    option routers 10.92.3.1;' >> /etc/dhcp/dhcpd.conf
    echo '    option broadcast-address 10.92.3.255;' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '    # SOAL 6: Static Lease untuk VIP kugisaki' >> /etc/dhcp/dhcpd.conf
    echo '    host kugisaki {' >> /etc/dhcp/dhcpd.conf
    echo '        hardware ethernet 02:42:f3:87:bc:00;' >> /etc/dhcp/dhcpd.conf
    echo '        fixed-address 10.92.3.99;' >> /etc/dhcp/dhcpd.conf
    echo '    }' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo '    pool {' >> /etc/dhcp/dhcpd.conf
    echo '        allow members of "ShortLease";' >> /etc/dhcp/dhcpd.conf
    echo '        default-lease-time 1800;' >> /etc/dhcp/dhcpd.conf
    echo '        max-lease-time 1800;' >> /etc/dhcp/dhcpd.conf
    echo '        range 10.92.3.10 10.92.3.89;' >> /etc/dhcp/dhcpd.conf
    echo '    }' >> /etc/dhcp/dhcpd.conf
    echo '    pool {' >> /etc/dhcp/dhcpd.conf
    echo '        allow members of "LongLease";' >> /etc/dhcp/dhcpd.conf
    echo '        default-lease-time 43200;' >> /etc/dhcp/dhcpd.conf
    echo '        max-lease-time 43200;' >> /etc/dhcp/dhcpd.conf
    echo '        range 10.92.3.100 10.92.3.199;' >> /etc/dhcp/dhcpd.conf
    echo '    }' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf

    # Validasi Syntax & Restart Service
    dhcpd -t
    service isc-dhcp-server restart
    ```
    jalankan
    ```bash
    bash /root/script.sh
    ```

2. lakukan renewal ip di semua client
   1. Client Divisi A (Network 2 - 10.92.2.0/24)
    itadori, fushiguro, zenin, okkotsu
    2. Client Divisi B (Network 3 - 10.92.3.0/24)
    ino, kusakabe, miwa, kugisaki
    ```bash
    ifdown eth0 2>/dev/null && ifup eth0
    cat /etc/resolv.conf
    ```

3. cek dengan ini di semua client tadi
    ```bash
    ip a
    ```
    ```bash
    ip route
    ```
    ```bash
    cat /etc/resolv.conf
    ```
    ![alt text](<Assets/No7_test fushiguro.png>)
    ![alt text](<Assets/No7_test ino.png>)
    ![alt text](<Assets/No7_test itadori.png>)
    ![alt text](<Assets/No7_test kugisaki.png>)
    ![alt text](<Assets/No7_test kusakabe.png>)
    ![alt text](<Assets/No7_test miwa.png>)
    ![alt text](<Assets/No7_test okkotsu.png>)
    ![alt text](<Assets/No7_test zenin.png>)

4. cek lease database di tengen
   ```bash
   cat /var/lib/dhcp/dhcpd.leases
   ```
    ![alt text](<Assets/No7_test tengen1.png>)
    ![alt text](<Assets/No7_test tengen2.png>)
    ![alt text](<Assets/No7_test tengen3.png>)