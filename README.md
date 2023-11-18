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
#### Pada masing-masing worker PHP, lakukan konfigurasi virtual host untuk website berikut dengan menggunakan php 7.3. 


Download terlebih dahulu modules yang diperlukan, lalu kita download file modul-3 yang tertera di soal

```bash
apt-get update && apt-get install mariadb-client -y
apt-get install nginx -y
apt-get install php php-fpm -y

apt-get install wget -y
apt-get install unzip -y
wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1ViSkRq7SmwZgdK64eRbr5Fm1EGCTPrU1' -O modul-3.zip 
unzip modul-3.zip -d .

service php7.3-fpm start

cp -r /root/modul-3 /var/www
cp -r /root/nginx /etc

ln -s /etc/nginx/sites-available /etc/sites-enabled

service nginx restart	
```

lalu dilakukan konfigurasi nginx 
dimana diganti rootnya menjadi 

```
root /var/www/modul-3
```
dan jangan lupa menambahkan index.php pada list nya 
![image](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/86884506/66852636-077e-47aa-a248-15ad2bd6a4dc)

dilakukan pada ketiga worker PHP yaitu Lawine, Linie, Lugner

## Soal 7
#### aturlah agar Eisen dapat bekerja dengan maksimal, lalu lakukan testing dengan 1000 request dan 100 request/second. 

pertama install modules yang diperlukan 

```bash

echo nameserver 192.168.122.1 > /etc/resolv.conf

apt-get update && apt-get install apache2-utils -y
apt-get install nginx -y

```

lalu untuk konfigurasi lb granz_channel (default : round robin)

```bash
upstream myweb {
        hash $request_uri consistent;
        server 192.182.3.1; # IP lawine
        server 192.182.3.2; # IP linie
        server 192.182.3.3; # IP Lugner
}


server {
        listen 80;
        server_name granz.channel.B08.com;


        location / {
                proxy_pass http://myweb;
        }

}
```

konfigurasi riegel_canyon (default round-robin)

```bash
upstream laravel {
        server 192.182.4.1; # IP Frieren
        server 192.182.4.2; # IP Flamme
        server 192.182.4.3; # IP Fern
}


server {
        listen 80;
        server_name riegel.canyon.B08.com;

        location / {
                proxy_pass http://laravel;
        }

}

```

jangan lupa melakukan 
``` service nginx restart```
setiap melakukan perubahan pada konfigurasi nginx

lalu dapat dilakukan test pada salah satu client, yang telah mendapatkan lease dan menginstall apache2utils

```bash
ab -n 1000 -c 100 granz.channel.B08.com/myweb
```
![image](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/86884506/28e280d4-17f2-4a8c-8889-a88ebca5434f)


## Soal 8
#### melakukan testing dengan load 200 request dan 10 request per second menggunakan tiap algoritma, maka kita perlu mengganti konfigurasi nginx nya tiap melakukan testing. pada defaultnya load balancer akan menggunakan algoritma round robin. untuk menggantinya kita harus menambahkan 1 line pada load balancenrnya yaitu 

```bash
upstream myweb {

        #tambahkan ini
        least_conn;

        server 192.182.3.1; # IP lawine
        server 192.182.3.2; # IP linie
        server 192.182.3.3; # IP Lugner
}


server {
        listen 80;
        server_name granz.channel.B08.com;


        location / {
                proxy_pass http://myweb;
        }

}
```

lalu dapat dilakukan testing kembali, dan dimasukkan pada grimoire, berikut hasil tabel yang telah kami masukkan kedalam Grimoire 

<img width="685" alt="Screenshot 2023-11-18 at 16 20 55" src="https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/86884506/82a2099a-4f48-4663-863a-b76f80c477e0">

## Soal 9
#### dengan algoritma round robin lakukan testing dengan menggunakan 3, 2, 1 worker.

untuk ini, kita tinggal mengurangi jumlah workernya dengan meng-comment IP Worker masing masing

```bash
upstream myweb {

        #Comment yang tidak diperlukan
        #server 192.182.3.1; # IP lawine
        server 192.182.3.2; # IP linie
        server 192.182.3.3; # IP Lugner
}


server {
        listen 80;
        server_name granz.channel.B08.com;


        location / {
                proxy_pass http://myweb;
        }

}
```

lalu hasil nya adalah : 
```bash
upstream myweb {

        #tambahkan ini
        least_conn;

        server 192.182.3.1; # IP lawine
        server 192.182.3.2; # IP linie
        server 192.182.3.3; # IP Lugner
}


server {
        listen 80;
        server_name granz.channel.B08.com;


        location / {
                proxy_pass http://myweb;
        }

}
```


## Soal 10
#### Selanjutnya coba tambahkan konfigurasi autentikasi di LB dengan dengan kombinasi username: “netics” dan password: “ajkyyy”, dengan yyy merupakan kode kelompok. Terakhir simpan file “htpasswd” nya di /etc/nginx/rahasisakita/

kita perlu menambahkan beberapa line pada konfigurasi nginx dari granz_channel 

```bash
upstream myweb {
        server 192.182.3.1; # IP lawine
        server 192.182.3.2; # IP linie
        server 192.182.3.3; # IP Lugner
}


server {
        listen 80;
        server_name granz.channel.B08.com;

        location / {
                auth_basic "Restricted Access";
                auth_basic_user_file /etc/nginx/rahasiakita/htpasswd;
                proxy_pass http://myweb;
        }


}
```


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
