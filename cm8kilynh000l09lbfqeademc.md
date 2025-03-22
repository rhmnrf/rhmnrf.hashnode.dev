---
title: "High Availability (HA) untuk mendukung Disaster Recovery (DRC) - CentOS 7 Edition #Part2"
datePublished: Sat Mar 22 2025 18:00:50 GMT+0000 (Coordinated Universal Time)
cuid: cm8kilynh000l09lbfqeademc
slug: high-availability-ha-untuk-mendukung-disaster-recovery-drc-centos-7-edition-part2
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/BsAzhHIIwxg/upload/e32a724a7e23d23c47c4b86e68923163.jpeg
tags: server, haproxy, high-availability

---

Berikut adalah penjelasan lebih detail tentang cara mengkonfigurasi **HAProxy** sebagai solusi **High Availability (HA)** untuk web server berbasis **httpd** pada **CentOS 7**, sehingga ketika server utama down, lalu lintas otomatis dialihkan ke server cadangan. HAProxy bertindak sebagai load balancer yang mendistribusikan lalu lintas dan mendukung failover otomatis dengan memantau status server backend.

---

## Prasyarat

* **Server HAProxy**: Satu server terpisah (misalnya, IP: 192.168.1.5) untuk menjalankan HAProxy.
    
* **Server Web**:
    
    * Server Utama (misalnya, IP: 192.168.1.10) dengan httpd.
        
    * Server Cadangan (misalnya, IP: 192.168.1.11) dengan httpd.
        
* **IP Virtual (VIP)**: IP yang digunakan oleh HAProxy untuk menerima lalu lintas (misalnya, 192.168.1.100).
    
* **Domain**: Arahkan domain (misalnya, [domain.com](http://domain.com)) ke VIP.
    

---

**Langkah-langkah Konfigurasi HAProxy**

## 1\. **Instal HAProxy**

Di server HAProxy, instal HAProxy:

```bash
yum install -y haproxy
```

## 2\. **Konfigurasi File HAProxy**

Edit file konfigurasi HAProxy di /etc/haproxy/haproxy.cfg. Berikut adalah contoh konfigurasi lengkap:

```plaintext
# Konfigurasi global
global
    log /dev/log local0
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 4000
    user haproxy
    group haproxy
    daemon
    stats socket /var/lib/haproxy/stats

# Pengaturan default
defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    option redispatch
    retries 3
    timeout http-request 10s
    timeout queue 1m
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    timeout http-keep-alive 10s
    timeout check 10s
    maxconn 3000

# Frontend: Menerima lalu lintas dari klien
frontend http_front
    bind 192.168.1.100:80           # VIP dan port untuk domain
    mode http
    option httplog
    default_backend http_back       # Arahkan ke backend

# Backend: Kelompok server web
backend http_back
    mode http
    balance roundrobin              # Algoritma distribusi (opsional: leastconn, source)
    option httpchk HEAD / HTTP/1.1\r\nHost:domain.com  # Health check
    server web1 192.168.1.10:80 check  # Server Utama
    server web2 192.168.1.11:80 check backup  # Server Cadangan
```

**Penjelasan Konfigurasi**:

* **global**: Pengaturan umum seperti logging, user, dan batas koneksi.
    
* **defaults**: Parameter default untuk semua frontend dan backend (timeout, retries, dll.).
    
* **frontend http\_front**:
    
    * bind 192.168.1.100:80: HAProxy mendengarkan di VIP pada port 80.
        
    * default\_backend http\_back: Mengarahkan lalu lintas ke backend bernama http\_back.
        
* **backend http\_back**:
    
    * balance roundrobin: Mendistribusikan lalu lintas secara bergilir (opsional, bisa diganti dengan leastconn untuk memilih server dengan koneksi paling sedikit).
        
    * option httpchk: Memeriksa kesehatan server dengan mengirimkan request HEAD ke root (/). Jika server tidak merespons, dianggap down.
        
    * server web1: Server Utama, ditandai dengan check untuk pemantauan.
        
    * server web2: Server Cadangan, ditandai dengan backup agar hanya aktif jika web1 down.
        

## 3\. **Jalankan dan Aktifkan HAProxy**

```bash
systemctl enable haproxy
systemctl start haproxy
```

Periksa status:

```bash
systemctl status haproxy
```

## 4\. **Konfigurasi httpd di Server Web**

Pastikan httpd di Server Utama (192.168.1.10) dan Cadangan (192.168.1.11) berjalan:

* **Instal httpd**:
    
    ```bash
    yum install -y httpd
    systemctl enable httpd
    systemctl start httpd
    ```
    
* **Konfigurasi Virtual Host** (opsional): Edit /etc/httpd/conf.d/[domain.com](http://domain.com).conf di kedua server:
    
    ```plaintext
    <VirtualHost *:80>
        ServerName domain.com
        DocumentRoot /var/www/html
        <Directory /var/www/html>
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>
    ```
    
* **Konten Web**: Pastikan konten di /var/www/html sama di kedua server (lihat bagian sinkronisasi di bawah).
    

## 5\. **Sinkronisasi Konten Antar Server**

Agar Server Cadangan dapat melayani konten yang sama:

* **Rsync**: Dari Server Utama:
    
    ```bash
    rsync -avz -e ssh /var/www/html/ 192.168.1.11:/var/www/html/
    ```
    
    Jadwalkan dengan cron setiap 5 menit:
    
    ```bash
    crontab -e
    */5 * * * * rsync -avz -e ssh /var/www/html/ 192.168.1.11:/var/www/html/
    ```
    
* **Alternatif**: Gunakan **GlusterFS** atau **NFS** untuk replikasi real-time jika diperlukan.
    

## 6\. **Firewall dan SELinux**

* **Firewall di Server HAProxy**:
    
    ```bash
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --reload
    ```
    
* **Firewall di Server Web**:
    
    ```bash
    firewall-cmd --permanent --add-service=http
    firewall-cmd --reload
    ```
    
* **SELinux** (jika aktif):
    
    ```bash
    setsebool -P httpd_can_network_connect 1
    chcon -R -t httpd_sys_content_t /var/www/html/
    ```
    

## 7\. **Monitoring dan Statistik**

HAProxy menyediakan halaman statistik bawaan:

* Tambahkan ke haproxy.cfg:
    
    ```plaintext
    listen stats
        bind 192.168.1.5:8404
        stats enable
        stats uri /stats
        stats realm Haproxy\ Statistics
        stats auth admin:password  # Ganti dengan username:password Anda
    ```
    
* Restart HAProxy:
    
    ```bash
    systemctl restart haproxy
    ```
    
* Akses [http://192.168.1.5:8404/stats](http://192.168.1.5:8404/stats) untuk memantau status server (web1 dan web2).
    

## 8\. **Pengujian Failover**

* Matikan httpd di Server Utama:
    
    ```bash
    systemctl stop httpd  # Di 192.168.1.10
    ```
    
* Akses [domain.com](http://domain.com) di browser. HAProxy akan mendeteksi Server Utama down (setelah timeout health check) dan mengalihkan lalu lintas ke Server Cadangan.
    
* Periksa log HAProxy:
    
    ```bash
    tail -f /var/log/haproxy.log
    ```
    

---

## Penyesuaian Tambahan

1. **Health Check Lebih Spesifik**: Jika situs Anda memiliki halaman khusus untuk pengecekan (misalnya, /health), ubah option httpchk:
    
    ```plaintext
    option httpchk GET /health HTTP/1.1\r\nHost:domain.com
    ```
    
2. **Timeout dan Interval Check**:
    
    * Kurangi timeout check menjadi 2s untuk deteksi lebih cepat:
        
        ```plaintext
        timeout check 2s
        ```
        
    * Tambahkan inter dan fall pada server:
        
        ```plaintext
        server web1 192.168.1.10:80 check inter 2000ms fall 3
        ```
        
        Artinya, server dianggap down setelah 3 kali gagal dalam interval 2 detik.
        
3. **Sticky Sessions** (Opsional): Jika Anda perlu mempertahankan sesi pengguna:
    
    ```plaintext
    backend http_back
        mode http
        balance roundrobin
        cookie SERVERID insert indirect nocache
        server web1 192.168.1.10:80 check cookie web1
        server web2 192.168.1.11:80 check cookie web2 backup
    ```
    
4. **HTTPS (Opsional)**: Tambahkan SSL dengan sertifikat:
    
    * Gabungkan sertifikat dan kunci privat ke file .pem (misalnya, /etc/haproxy/domain.pem).
        
    * Ubah frontend:
        
        ```plaintext
        frontend https_front
            bind 192.168.1.100:443 ssl crt /etc/haproxy/domain.pem
            mode http
            default_backend http_back
        ```
        

---

## Cara Kerja

* HAProxy menerima lalu lintas di VIP (192.168.1.100).
    
* Health check memastikan Server Utama (web1) aktif. Jika down, lalu lintas dialihkan ke Server Cadangan (web2).
    
* Failover otomatis tanpa perubahan DNS, selama domain menunjuk ke VIP.
    

Dengan konfigurasi ini, HAProxy memberikan solusi HA yang andal untuk httpd di CentOS 7, dengan failover cepat dan fleksibilitas tinggi.