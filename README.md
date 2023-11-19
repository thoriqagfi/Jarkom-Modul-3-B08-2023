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

```conf
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

```conf
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

```conf
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

```conf
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
```conf
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

Sebelum melakukan konfiguras autentikasi dari nginx, kita akan membuat username dan password dengan

```bash
htpasswd -b -c /etc/nginx/rahasiakita/htpasswd netics ajkB08
```

kita perlu menambahkan beberapa line pada konfigurasi nginx dari granz_channel

```conf
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
#### Lalu buat untuk setiap request yang mengandung /its akan di proxy passing menuju halaman https://www.its.ac.id. (11) hint: (proxy_pass)

Untuk melakukan proxy pass ke its.ac.id melalui granz.channel.B08.com/its, kita perlu menambahkan konfigurasi pada load balancer granz.channel.B08.com

```conf
location ^~ /its {
        proxy_pass https://www.its.ac.id;
}
```

Kita lakukan pada load balancer karena semua client akan mengarah ke 192.182.1.2 (IP Web Server) yang akan di redirect ke 192.182.2.2 (IP Load Balancer). Artinya, semua akses dari Client akan diproses melalui Web Server dan Load Balancer terlebih dahulu.

Selanjutnya, kita lakukan testing pada Client Sein. Hasilnya :

![WhatsApp Image 2023-11-19 at 15 27 55_f4b67d3c](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/3c80cb0b-59d3-4274-85d2-c906d99d87a0)


## Soal 12
#### Selanjutnya LB ini hanya boleh diakses oleh client dengan IP [Prefix IP].3.69, [Prefix IP].3.70, [Prefix IP].4.167, dan [Prefix IP].4.168. (12) hint: (fixed in dulu clinetnya)

Untuk membatasi akses hanya oleh IP tertentu, kita menambahkan konfigurasi pada load balancer.

```conf
        allow 192.182.3.69;
        allow 192.182.3.70;
        allow 192.182.4.167;
        allow 192.182.4.168;
        deny all;
```


## Soal 13
#### Semua data yang diperlukan, diatur pada Denken dan harus dapat diakses oleh Frieren, Flamme, dan Fern. (13)

Agar data hanya dapat diakses oleh Frieren, Flamme dan Fern, kita melakukan konfigurasi :

```bash
mysql << EOF

CREATE USER 'kelompokB08'@'192.182.4.%' IDENTIFIED BY 'passwordB08';
CREATE USER 'kelompokB08'@'localhost' IDENTIFIED BY 'passwordB08';
CREATE DATABASE dbkelompokB08;
GRANT ALL PRIVILEGES ON *.* TO 'kelompokB08'@'192.182.4.%';
GRANT ALL PRIVILEGES ON *.* TO 'kelompokB08'@'localhost';
FLUSH PRIVILEGES;

EOF

cp -r /root/mysql /etc

service mysql restart
```

Selanjutnya, kita lakukan konfigurasi untuk mengizinkan mysql agar dapat diakses oleh luar IP Database (Denken) dengan menambahkan pada `mariadb.conf.d/50-server.cnf` :

```conf
bind-address            = 0.0.0.0
```


## Soal 14
#### Frieren, Flamme, dan Fern memiliki Riegel Channel sesuai dengan quest guide berikut. Jangan lupa melakukan instalasi PHP8.0 dan Composer (14)

Kita melakukan setup untuk setiap worker laravel.

Pertama, kita lakukan instalasi semua dependency yang diperlukan
`nginx.sh`:
```bash
apt-get update

wait
apt-get install mariadb-client -y
apt-get install -y lsb-release ca-certificates apt-transport-https software-properties-common gnupg2

curl -sSLo /usr/share/keyrings/deb.sury.org-php.gpg https://packages.sury.org/php/apt.gpg

sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/so$

apt-get update

apt-get install php8.0-mbstring php8.0-xml php8.0-cli php8.0-common php8.0-intl php8.0-opcache php8.0-readline php8.0-mysql php8.0-fpm php8$
apt-get install nginx -y
apt-get install apache2-utils -y

wget https://getcomposer.org/download/2.0.13/composer.phar
chmod +x composer.phar
mv composer.phar /usr/bin/composer

apt-get install git -y
```

Kemudian, kita clone repository dan setup laravel

`laravel.sh`:
```bash
git clone https://github.com/martuafernando/laravel-praktikum-jarkom.git

wait

cp -r laravel-praktikum-jarkom /var/www
cd /var/www/laravel-praktikum-jarkom

composer install
cp .env.example .env

composer update

php artisan migrate:fresh
php artisan db:seed --class=AiringsTableSeeder
php artisan key:generate
php artisan jwt:secret
```

Tak lupa, kita sesuaikan .env database pada laravel worker
```conf
DB_CONNECTION=mysql
DB_HOST=192.182.2.1
DB_PORT=3306
DB_DATABASE=dbkelompokB08
DB_USERNAME=kelompokB08
DB_PASSWORD=passwordB08
```

Setelah semua setup laravel berhasil, kita konfigurasi nginx

`/sites-enabled/default` :
```conf
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/laravel-praktikum-jarkom/public;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html index.php;

        server_name riegel.canyon.B08.com;

        location / {
                try_files $uri $uri/ /index.php$query_string;
        }

        # pass PHP scripts to FastCGI server
        #
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
        #
        #       # With php-fpm (or other unix sockets):
                fastcgi_pass unix:/run/php/php8.0-fpm.sock;
        #       # With php-cgi (or other tcp sockets):
        #       fastcgi_pass 127.0.0.1:9000;
        }

        location ~ /\.ht {
                deny all;
        }
}
```

`nginx.sh` :
```bash
service php8.0-fpm start

cp default /etc/nginx/sites-available/

chown -R www-data.www-data /var/www/laravel-praktikum-jarkom/storage

service nginx restart
```

```bash
bash /root/script.sh

wait

bash /root/laravel.sh

wait

bash /root/nginx.sh
```

Ketika kita lakukan lynx ke localhost masing-masing worker :

![WhatsApp Image 2023-11-19 at 15 27 55_669cae59](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/7e7e69d6-8e6a-4a8e-9672-8b61736e3465)


## Soal 15 - 17
#### Riegel Channel memiliki beberapa endpoint yang harus ditesting sebanyak 100 request dengan 10 request/second. Tambahkan response dan hasil testing pada grimoire.
#### a. POST /auth/register (15)
#### b. POST /auth/login (16)
#### c. GET /me (17)

Sebelum melakukan testing pada client, kita coba testing di localhost terlebih dahulu :

![WhatsApp Image 2023-11-19 at 15 55 37_c879910a](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/e106be51-d60e-460c-a860-f35978281651)

Untuk melakukan testing api, kita melakukan testing api menggunakan curl

- `/api/auth/register`
```bash
curl -XPOST -H "Content-type: application/json" -d '{

"username":"thoriq",
"password":"123456789"

}' 'riegel.canyon.B08.com/api/auth/register'
```
Hasil :

![WhatsApp Image 2023-11-19 at 16 13 27_e0f6d906](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/7e73e3f0-59c8-4a4a-b266-66590cb51316)


- `/api/auth/login`
```bash
curl -XPOST -H "Content-type: application/json" -d '{

"username":"thoriq",
"password":"123456789"

}' 'riegel.canyon.B08.com/api/auth/login'
```

Hasil :

![WhatsApp Image 2023-11-19 at 16 13 28_ba34cea2](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/56d056a7-3be5-405f-8389-0fcc0e18a095)


- `/api/me`
```bash
curl -XGET -H "Content-type: application/json" -d '{

"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vbGFyYXZlbC9hcGkvYXV0aC9sb2dpbiIsImlhdCI6MTcwMDM4NTI5OSwiZXhwIjoxNzAwMzg4ODk5LCJuYmYiOjE3MDAzODUyOTksImp0aSI6Ik9XalBZUmYzNHFmSERySVQiLCJzdWIiOiIzIiwicHJ2IjoiMjNiZDVjODk0OWY2MDBhZGIzOWU3MDFjNDAwODcyZGI3YTU5NzZmNyJ9.yZx6nxXWfhji4L-jPvbPy9Jt-OeuOqULIfzeYdtzK0w"

}' 'riegel.canyon.B08.com/api/me'
```

Hasil :

![WhatsApp Image 2023-11-19 at 16 17 53_792442bf](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/14585ce5-811f-4878-b825-4507b13c78da)


Untuk mengecek apakah sudah masuk ke database atau belum, kita cek di database :

![WhatsApp Image 2023-11-19 at 16 13 35_d2bee916](https://github.com/thoriqagfi/Jarkom-Modul-3-B08-2023/assets/92865110/50f16b0e-5775-44c1-b2a4-db454ca97468)



## Soal 18

```conf
upstream laravel {

#       least_conn;
        server 192.182.4.1; # IP Frieren
        server 192.182.4.2; # IP Flamme
        server 192.182.4.3; # IP Fern
}


server {
        listen 80;
        server_name riegel.canyon.B08.com;

        location ^~ /its {
                proxy_pass https://www.its.ac.id;
        }

        location /api/auth/register {
                proxy_bind 192.182.4.1;
                proxy_pass http://riegel.canyon.B08.com/api/auth/register;
        }

        location /api/auth/login {
                proxy_bind 192.182.4.2;
                proxy_pass http://riegel.canyon.B08.com/api/auth/login;
        }

        location /api/me {
                proxy_bind 192.182.4.3;
                proxy_pass http://riegel.canyon.B08.com/api/me;
        }

        location / {
                proxy_pass http://laravel;
        }

}
```

## Soal 19

## Soal 20
#### Nampaknya hanya menggunakan PHP-FPM tidak cukup untuk meningkatkan performa dari worker maka implementasikan Least-Conn pada Eisen. Untuk testing kinerja dari worker tersebut dilakukan sebanyak 100 request dengan 10 request/second. (20)

Untuk mengimplementasikan `least conn`, kita hanya perlu menambahkan `least_conn;` pada load balancer laravel :

```conf
upstream laravel {

        least_conn;
        server 192.182.4.1; # IP Frieren
        server 192.182.4.2; # IP Flamme
        server 192.182.4.3; # IP Fern
}


server {
        listen 80;
        server_name riegel.canyon.B08.com;

        location ^~ /its {
                proxy_pass https://www.its.ac.id;
        }

        location /api/auth/register {
                proxy_bind 192.182.4.1;
                proxy_pass http://riegel.canyon.B08.com/api/auth/register;
        }

        location /api/auth/login {
                proxy_bind 192.182.4.2;
                proxy_pass http://riegel.canyon.B08.com/api/auth/login;
        }

        location /api/me {
                proxy_bind 192.182.4.3;
                proxy_pass http://riegel.canyon.B08.com/api/me;
        }

        location / {
                proxy_pass http://laravel;
        }

}
```




## Tools
Test API : [Online Curl](https://tools.w3cub.com/curl-builder)
