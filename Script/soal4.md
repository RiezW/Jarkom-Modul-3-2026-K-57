1. edit dhcp.conf milik tengen, sebelum itu cek mac addres dari itadori, fushiguro, zenin, dan okkotsu
   ```bash
   ip link show eth0
   ```
    ![alt text](<Assets/No4_mac fushiguro.png>)
    ![alt text](<Assets/No4_mac itadori.png>)
    ![alt text](<Assets/No4_mac okkotsu.png>)
    ![alt text](<Assets/No4_mac zenin.png>)

2. edit script tengen
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

    # Atur dhcpd.conf
    cat > /etc/dhcp/dhcpd.conf << 'EOF'
    default-lease-time 600;
    max-lease-time 3600;

    # Deklarasi Class Utama
    class "Dev" {
        match hardware;
    }

    class "Support" {
        match hardware;
    }

    # Subclass anggota dengan prefix 1: (Ethernet)
    subclass "Dev" 1:02:42:b2:4f:49:00;  # itadori
    subclass "Dev" 1:f2:dd:ec:51:c9:cb;  # fushiguro

    subclass "Support" 1:02:42:54:1c:11:00;  # zenin
    subclass "Support" 1:02:42:0d:ec:d7:00;  # okkotsu

    # Network 4 (Local Subnet tengen)
    subnet 10.92.4.0 netmask 255.255.255.0 {
    }

    # Network 2 (Divisi A)
    subnet 10.92.2.0 netmask 255.255.255.0 {
        option routers 10.92.2.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;

        # Pool 1: Dev (itadori & fushiguro)
        pool {
            allow members of "Dev";
            range 10.92.2.10 10.92.2.35;
        }

        # Pool 2: Support (zenin & okkotsu)
        pool {
            allow members of "Support";
            range 10.92.2.70 10.92.2.95;
        }
    }

    # Network 3 (Divisi B)
    subnet 10.92.3.0 netmask 255.255.255.0 {
        range 10.92.3.10 10.92.3.50;
        option routers 10.92.3.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;
    }
    EOF

    # Validasi & Restart Service
    dhcpd -t
    service isc-dhcp-server restart
    ```
    kasih izin dan jalankan
    ```bash
    chmod +x /root/script.sh
    ```
    ```bash
    bash /root/script.sh
    ```
    pastikan service jalan
    ```bash
    service isc-dhcp-server status
    ```

3. Pembuktian <br>
    - itadori & fushiguro: Harus mendapatkan IP di rentang 10.92.2.10 – 10.92.2.35.
    - zenin & okkotsu: Harus mendapatkan IP di rentang 10.92.2.70 – 10.92.2.95.
    - Rentang 10.92.2.36 – 10.92.2.69 terbukti dilewati / tidak dipakai.

    ```bash
    ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```

4. troubleshoot <br>
   edit dhcp.conf milik tengen
   ```bash
   nano /etc/dhcp/dhcpd.conf
   ```
   ```bash
    default-lease-time 600;
    max-lease-time 3600;

    # Class Tim Dev (UPPERCASE)
    class "Dev" {
        match if (hardware = 1:02:42:B2:4F:49:00) or (hardware = 1:F2:DD:EC:51:C9:CB);
    }

    # Class Tim Support (UPPERCASE)
    class "Support" {
        match if (hardware = 1:02:42:54:1C:11:00) or (hardware = 1:02:42:0D:EC:D7:00);
    }

    # Subnet 4 (Local Tengen)
    subnet 10.92.4.0 netmask 255.255.255.0 {
    }

    # Subnet 2 (Network 2 - Divisi A)
    subnet 10.92.2.0 netmask 255.255.255.0 {
        option routers 10.92.2.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;

        # Pool 1: Tim Dev
        pool {
            allow members of "Dev";
            range 10.92.2.10 10.92.2.35;
        }

        # Pool 2: Tim Support
        pool {
            allow members of "Support";
            range 10.92.2.70 10.92.2.95;
        }
    }

    # Subnet 3 (Network 3 - Divisi B)
    subnet 10.92.3.0 netmask 255.255.255.0 {
        range 10.92.3.10 10.92.3.50;
        option routers 10.92.3.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 192.168.122.1;
    }
    ```
    restart service
    ```bash
    service isc-dhcp-server restart
    ```

    check lagi di client
    ```bash
    ifdown eth0 2>/dev/null && ifup eth0
    ip a
    ```


    note: 
    pastikan dulu di jichi jalan
    ```bash
    pkill dhcrelay 2>/dev/null || true
    dhcrelay -i eth2 -i eth3 -i eth4 10.92.4.2
    ```
    ```bash
    ps aux | grep dhcrelay
    ```
    ![alt text](<Assets/No4_test fushiguro.png>)
    ![alt text](<Assets/No4_test itadori.png>)
    ![alt text](<Assets/No4_test okkotsu.png>)
    ![alt text](<Assets/No4_test Zenin.png>)