1. update script jichi
   ```bash
   nano /root/script.sh
   ```
   ```bash
    #!/bin/bash

    # 1. Setup DNS resolver sementara untuk download paket
    echo "nameserver 192.168.122.1" > /etc/resolv.conf

    # 2. Aktifkan IP Forwarding & Source NAT (Soal 1)
    sysctl -w net.ipv4.ip_forward=1
    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

    # 3. Install ulang isc-dhcp-relay secara otomatis
    DEBIAN_FRONTEND=noninteractive apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install isc-dhcp-relay -y

    # 4. Tuliskan file konfigurasi default
    cat > /etc/default/isc-dhcp-relay << 'EOF'
    SERVERS="10.92.4.2"
    INTERFACES="eth2 eth3 eth4"
    OPTIONS=""
    EOF

    # 5. Jalankan daemon dhcrelay secara langsung
    pkill dhcrelay 2>/dev/null || true
    dhcrelay -i eth2 -i eth3 -i eth4 10.92.4.2
    ```
    beri izin dan jalankan
    ```bash
    chmod +x /root/script.sh
    ```
    ```bash
    bash /root/script.sh
    ```
    test status
    ```bash
    ps aux | grep dhcrelay
    ```

2. Pembuktian lewat itadori
   ```bash
   ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```
    lewat Ino
    ```bash
    ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```

3. tidak bisa kasih ip dari tengen, jadi update tengen
   ```bash
   nano /etc/dhcp/dhcpd.conf
   ```
   ```bash
   default-lease-time 600;
    max-lease-time 3600;

    subnet 10.92.4.0 netmask 255.255.255.0 {
    }

    subnet 10.92.2.0 netmask 255.255.255.0 {
        range 10.92.2.10 10.92.2.50;
        option routers 10.92.2.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;
    }

    subnet 10.92.3.0 netmask 255.255.255.0 {
        range 10.92.3.10 10.92.3.50;
        option routers 10.92.3.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;
    }
    ```
    restart
    ```bash
    service isc-dhcp-server restart
    ```
    uji coba lagi ke itadori dan fushiguro
    ```bash
    ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```
    ![alt text](<Assets/No3_test di itadori.png>)
    ![alt text](<Assets/No3_test fushiguro.png>)