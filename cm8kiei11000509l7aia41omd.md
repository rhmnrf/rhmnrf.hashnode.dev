---
title: "High Availability (HA) untuk mendukung Disaster Recovery (DRC) - CentOS 7 Edition #Part1"
datePublished: Sat Mar 22 2025 17:55:01 GMT+0000 (Coordinated Universal Time)
cuid: cm8kiei11000509l7aia41omd
slug: high-availability-ha-untuk-mendukung-disaster-recovery-drc-centos-7-edition-part1
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/Z3_uSvERPfM/upload/eedb938937f0b2b22fd77b32215b87d6.jpeg
tags: server

---

Jika server menggunakan **CentOS 7** dengan **httpd** (Apache) sebagai web server, konfigurasi skema **High Availability (HA)** untuk mendukung **Disaster Recovery (DRC)** dapat dilakukan dengan memanfaatkan tools dan teknologi berbasis Linux. Tujuannya tetap sama: ketika server utama down, server cadangan dapat secara otomatis mengambil alih fungsi domain utama. Berikut langkah-langkahnya:

---

## 1\. **Arsitektur Dasar HA**

* **Dua Server**: Siapkan dua server CentOS 7 (Server Utama dan Server Cadangan) dengan instalasi httpd.
    
* **IP Floating**: Gunakan IP virtual (VIP) yang dapat berpindah antar server untuk mengarahkan lalu lintas ke server yang aktif.
    
* **Replikasi Konten**: Sinkronkan konten web agar kedua server memiliki data yang identik.
    

---

## 2\. **Konfigurasi Keepalived untuk Failover Otomatis**

**Keepalived** adalah solusi populer untuk HA di Linux yang menyediakan IP virtual dan failover otomatis.

Langkah-langkah:

* **Instal Keepalived**: Pada kedua server, instal Keepalived:
    
    ```text
    yum install -y keepalived
    ```
    
* **Konfigurasi Keepalived**:
    
    * Edit file konfigurasi di /etc/keepalived/keepalived.conf.
        
    * Contoh konfigurasi untuk Server Utama (IP: 192.168.1.10):
        
        ```text
        vrrp_instance VI_1 {
            state MASTER
            interface eth0          # Ganti dengan nama interface jaringan Anda
            virtual_router_id 51
            priority 100            # Prioritas lebih tinggi untuk MASTER
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass mysecret  # Password untuk sinkronisasi
            }
            virtual_ipaddress {
                192.168.1.100       # IP Virtual (VIP) untuk domain
            }
        }
        ```
        
    * Contoh konfigurasi untuk Server Cadangan (IP: 192.168.1.11):
        
        ```text
        vrrp_instance VI_1 {
            state BACKUP
            interface eth0
            virtual_router_id 51
            priority 90             # Prioritas lebih rendah untuk BACKUP
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass mysecret
            }
            virtual_ipaddress {
                192.168.1.100       # IP Virtual yang sama
            }
        }
        ```
        
* **Jalankan Keepalived**:
    
    ```text
    systemctl enable keepalived
    systemctl start keepalived
    ```
    
* **Cara Kerja**:
    
    * Keepalived menggunakan protokol VRRP untuk mendeteksi kegagalan server utama.
        
    * Jika Server Utama down, VIP (192.168.1.100) akan otomatis beralih ke Server Cadangan.
        
* **Arahkan Domain ke VIP**:
    
    * Pada DNS, buat record A untuk domain Anda (misalnya, [domain.com](http://domain.com)) yang menunjuk ke VIP (192.168.1.100).
        

---

## 3\. **Sinkronisasi Konten Web**

Agar Server Cadangan dapat melayani konten yang sama, replikasi data httpd diperlukan.

### Opsi 1: Rsync + Cron

* **Konfigurasi Rsync**: Sinkronkan direktori web (default: /var/www/html) dari Server Utama ke Cadangan:
    
    ```text
    rsync -avz -e ssh /var/www/html/ user@192.168.1.11:/var/www/html/
    ```
    
* **Otomatisasi dengan Cron**: Tambahkan ke crontab di Server Utama untuk sinkronisasi setiap 5 menit:
    
    ```text
    */5 * * * * rsync -avz -e ssh /var/www/html/ user@192.168.1.11:/var/www/html/
    ```
    

### Opsi 2: DRBD (Distributed Replicated Block Device)

* **Instal DRBD**: Untuk replikasi real-time pada level block:
    
    ```text
    yum install -y drbd84-utils kmod-drbd84
    ```
    
* **Konfigurasi DRBD**:
    
    * Edit /etc/drbd.d/global\_common.conf dan buat resource (misalnya, webdata):
        
        ```text
        resource webdata {
            on server-utama {
                device    /dev/drbd1;
                disk      /dev/sdb1;    # Partisi untuk data web
                address   192.168.1.10:7789;
                meta-disk internal;
            }
            on server-cadangan {
                device    /dev/drbd1;
                disk      /dev/sdb1;
                address   192.168.1.11:7789;
                meta-disk internal;
            }
        }
        ```
        
    * Inisialisasi dan aktifkan:
        
        ```text
        drbdadm create-md webdata
        drbdadm up webdata
        drbdadm primary --force webdata  # Di Server Utama
        ```
        
    * Mount DRBD di /var/www/html pada server aktif.
        
* **Integrasi dengan Keepalived**: Tambahkan skrip ke Keepalived untuk mengalihkan DRBD primary saat failover.
    

---

## 4\. **Konfigurasi Apache (httpd)**

Pastikan httpd di kedua server siap melayani:

* **Instal httpd**:
    
    ```text
    yum install -y httpd
    ```
    
* **Konfigurasi Virtual Host**: Edit /etc/httpd/conf/httpd.conf atau buat file di /etc/httpd/conf.d/[domain.com](http://domain.com).conf:
    
    ```text
    <VirtualHost 192.168.1.100:80>
        ServerName domain.com
        DocumentRoot /var/www/html
        <Directory /var/www/html>
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>
    ```
    
* **Jalankan httpd**:
    
    ```text
    systemctl enable httpd
    systemctl start httpd
    ```
    

---

## 5\. **Firewall dan SELinux**

* **Izinkan Trafik**:
    
    ```text
    firewall-cmd --permanent --add-service=http
    firewall-cmd --reload
    ```
    
* **Atur SELinux** (jika aktif):
    
    ```text
    setsebool -P httpd_can_network_connect 1
    chcon -R -t httpd_sys_content_t /var/www/html/
    ```
    

---

## 6\. **Pengujian**

* Matikan Server Utama:
    
    ```text
    systemctl stop keepalived
    ```
    
* Periksa apakah VIP beralih ke Server Cadangan:
    
    ```text
    ip addr show
    ```
    
* Akses domain Anda untuk memastikan Server Cadangan melayani konten.
    

---

## 7\. **Pemantauan**

* Gunakan **Nagios**, **Zabbix**, atau skrip sederhana untuk memantau status server:
    
    ```text
    #!/bin/bash
    ping -c 2 192.168.1.10 || echo "Server Utama Down!"
    ```
    

---

### Catatan Tambahan

* **DNS TTL**: Jika Anda mengandalkan DNS eksternal, set TTL rendah (misalnya, 300 detik) agar perubahan IP cepat tersebar.
    
* **Load Balancer Alternatif**: Jika tidak ingin menggunakan Keepalived, pertimbangkan **HAProxy**:
    
    ```text
    frontend http_front
        bind 192.168.1.100:80
        default_backend http_back
    backend http_back
        server server1 192.168.1.10:80 check
        server server2 192.168.1.11:80 check backup
    ```
    

Dengan konfigurasi ini, ketika Server Utama down, Keepalived akan mengalihkan VIP ke Server Cadangan, dan httpd di server tersebut akan melayani domain utama secara otomatis, dengan konten yang sudah tersinkronisasi.