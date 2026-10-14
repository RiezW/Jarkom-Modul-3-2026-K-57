1. check mac addres dari ino, kusakabe, miwa
   ```bash
   ip link show eth0
   ```
    ino (ShortLease): 02:42:bb:de:44:00 <br>
    kusakabe (ShortLease): 02:42:b0:6e:76:00 <br>
    miwa (LongLease): 02:42:ce:d3:ac:00 <br>
    ![alt text](<Assets/No5_mac ino.png>)
    ![alt text](<Assets/No5_mac kusakabe.png>)
    ![alt text](<Assets/No5_mac miwa.png>)

2. update script tengen lagi
   ```bash
   nano /root/script.sh
   ```
   ```bash
    #!/bin/bash

    # Setup DNS resolver sementara
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

    # Atur dhcpd.conf Soal 4 & Soal 5
    echo 'default-lease-time 600;' > /etc/dhcp/dhcpd.conf
    echo 'max-lease-time 3600;' >> /etc/dhcp/dhcpd.conf
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
    echo 'subnet 10.92.4.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '}' >> /etc/dhcp/dhcpd.conf
    echo '' >> /etc/dhcp/dhcpd.conf
    echo 'subnet 10.92.2.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '    option routers 10.92.2.1;' >> /etc/dhcp/dhcpd.conf
    echo '    option subnet-mask 255.255.255.0;' >> /etc/dhcp/dhcpd.conf
    echo '    option domain-name-servers 192.168.122.1;' >> /etc/dhcp/dhcpd.conf
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
    echo 'subnet 10.92.3.0 netmask 255.255.255.0 {' >> /etc/dhcp/dhcpd.conf
    echo '    option routers 10.92.3.1;' >> /etc/dhcp/dhcpd.conf
    echo '    option subnet-mask 255.255.255.0;' >> /etc/dhcp/dhcpd.conf
    echo '    option domain-name-servers 10.92.1.2, 10.92.1.3;' >> /etc/dhcp/dhcpd.conf
    echo '    option domain-search "jujutsu.com";' >> /etc/dhcp/dhcpd.conf
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

3. test di ino, kusakabe dan miwa
   ```bash
   ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```
    ino kusakabe - 10.92.3.10 – 10.92.3.89 dengan lease time 1800 <br>
    miwa - 10.92.3.100 – 10.92.3.199 dengan lease time 43200

    ![alt text](<Assets/No5_test ino.png>)
    ![alt text](<Assets/No5_test kusakabe.png>)
    ![alt text](<Assets/No5_test miwa.png>)