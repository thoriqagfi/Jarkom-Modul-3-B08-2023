# Praktikum Jarkom Modul 3

|Nama|NRP|Kelas|
|:--:|:-:|:---:|
|Adrian Karuna Soetikno|5025211019|Jarkom B|
|Thariq Agfi Hermawan|5025211215|Jarkom B|
------------------------------------------

## Soal 0

*Aura (DHCP Relay)*

Pertama, kita set iptables untuk menghubungkan ke NAT agar bisa mendistribusikan dah koneksi internet

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE -s 192.182.0.0/16

echo nameserver 192.168.122.1 > /etc/resolv.conf
```

*HEITER (DNS Server)*

Kita membuat konfigurasi NGINX dengan konfigurasi sebagai berikut

- ```canyon.B08.com```

```nginx
; ; BIND data file for local loopback interface ; $TTL 604800

@ IN SOA canyon.B08.com. root.canyon.B08.com. (
                        2023141101      ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      riegel.canyon.B08.com.
@       IN      A       192.182.4.1     ; (ip lawine)
```

- ```channel.B08.com```

```nginx
; ; BIND data file for local loopback interface ; $TTL 604800

@ IN SOA channel.B08.com. root.channel.B08.com. (
                        2023111402      ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

@       IN      NS      granz.channel.B08.com.
@       IN      A       192.182.3.1     ; (ip frieren)
```

- ```named.conf.local```

```local
zone "riegel.canyon.B08.com" {
        type master;
        file "/etc/bind/jarkom/canyon.B08.com";
};

zone "granz.channel.B08.com" {
        type master;
        file "/etc/bind/jarkom/channel.B08.com";
};
```

- ```named.conf.options```

```options
options {
        directory "/var/cache/bind";

        forwarders {
                192.168.122.1;
        };

      // dnssec-validation auto;
        allow-query{any;};
        auth-nxdomain no;
        listen-on-v6 { any; };
};
```

Seluruh konfigurasi diatas kita taruh didalam root directory, kemudian kita jalankan `script.sh` di `.bashrc`. Berikut adalah konfigurasi `script.sh` :

```sh
echo nameserver 192.168.122.1 > /etc/resolv.conf

apt-get update
wait
apt-get install bind9 -y

mkdir /etc/bind/jarkom

cp /root/named.conf.local /etc/bind/
cp /root/canyon.B08.com /etc/bind/jarkom/
cp /root/channel.B08.com /etc/bind/jarkom/
cp /root/named.conf.options /etc/bind/

service bind9 restart
```

Selanjutnya, kita melakukan ping melalui client server. Berikut hasilnya :

[Image]

## Soal 1

Lakukan konfigurasi sesuai dengan peta yang sudah diberikan. Berikut hasilnya :

[Image]

## Soal 2 - 5

*Aura (DHCP Relay)*

```bash
echo nameserver 192.168.122.1 > /etc/resolv.conf
apt-get update
apt-get install isc-dhcp-relay -y
```

Selanjutnya, kita edit `/etc/default/isc-dhcp-relay.conf`

```conf
SERVERS = “192.182.1.1” (IP DHCP Server)
INTERFACES = “eth1 eth2 eth3 eth4” (SEMUA eth kecuali yang ke NAT)
```

Kemudian, kita edit `/etc/sysctl.conf`

```conf
net.ipv4.ip_forward=1
service isc-dhcp-relay restart
```

Semua konfigurasi diatas kita simpan dalam root directory yang akan dijalankan melalui `script.sh`.

```bash
echo nameserver 192.168.122.1 > /etc/resolv.conf
apt-get update
apt-get install isc-dhcp-relay -y
cp ./isc-dhcp-relay /etc/default/isc-dhcp-relay
cp ./sysctl.conf /etc/sysctl.conf
```

*Himmel (DHCP Server)*

Kita membuat `script.sh` yang berisi :

```bash
echo nameserver 192.168.122.1 > /etc/resolv.conf

apt-get update
apt-get install isc-dhcp-server -y

cp ./isc-dhcp-server /etc/default/isc-dhcp-server
cp ./dhcpd.conf /etc/dhcp/dhcpd.conf

service isc-dhcp-server restart
```

- `isc-dhcp-server`

```conf
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#       Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#       Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="eth0"
INTERFACESv6=""
```

- `dhcpd.conf`

```conf
subnet 192.182.1.0 netmask 255.255.255.0 {
}

subnet 192.182.2.0 netmask 255.255.255.0 {
}

subnet 192.182.3.0 netmask 255.255.255.0 {
    range 192.182.3.16 192.182.3.32;
    range 192.182.3.64 192.182.3.80;
    option routers 192.182.3.0;
    option broadcast-address 192.182.3.255;
    option domain-name-servers 192.182.1.2;
    default-lease-time 180;
    max-lease-time 5760;
}

subnet 192.182.4.0 netmask 255.255.255.0 {
    range 192.182.4.12 192.182.4.32;
    range 192.182.4.160 192.182.4.168;
    option routers 192.182.4.0;
    option broadcast-address 192.182.4.255;
    option domain-name-servers 192.182.1.2;
    default-lease-time 720;
    max-lease-time 5760;
}

host Lawine {
        hardware ethernet 3a:ff:68:ce:d5:cf;
        fixed-address 192.182.3.1;
}


host Linie {
        hardware ethernet 76:a0:85:61:16:3e;
        fixed-address 192.182.3.2;
}

host Lugner {
        hardware ethernet 1e:9d:8e:94:86:46;
        fixed-address 192.182.3.1;
}
```

Selanjutnya, kita melakukan setup database

[Need To Fix]

```sql
CREATE USER 'kelompokB08'@'%' IDENTIFIED BY 'passwordB08';
CREATE USER 'kelompokB08'@'localhost' IDENTIFIED BY 'passwordB08';
CREATE DATABASE dbkelompokB08;
GRANT ALL PRIVILEGES ON *.* TO 'kelompokB08'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'kelompokB08'@'localhost';
FLUSH PRIVILEGES;
```


## Soal 6

## Soal 7

## Soal 8

## Soal 9

## Soal 10

## Soal 11

## Soal 11

## Soal 12

## Soal 13

## Soal 14

## Soal 15

## Soal 16

## Soal 17

## Soal 18

## Soal 19

## Soal 20