# Fixed Sister (Load Balancing)

# NGINX Load Balancer Configuration for Voting App

Konfigurasi ini digunakan untuk membangun **load balancer** menggunakan **NGINX** yang mendistribusikan trafik ke beberapa backend server aplikasi voting.

---

## Isi File `nginx.conf`

```nginx
worker_processes auto;
events {
    worker_connections 2048;
}

http {
    upstream voting_backend {
        least_conn;
        server 192.168.138.35:9031 max_fails=3 fail_timeout=30s;
        server 192.168.138.140:9032 max_fails=3 fail_timeout=30s;
        server 192.168.138.61:9033 max_fails=3 fail_timeout=30s;
        keepalive 64;
    }

    server {
        listen 9030;
        location / {
            proxy_pass http://voting_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Connection "";

            proxy_connect_timeout 10s;
            proxy_read_timeout 30s;
            proxy_send_timeout 30s;

            proxy_buffering on;
            proxy_buffers 32 64k;
            proxy_busy_buffers_size 128k;
            proxy_temp_file_write_size 128k;
        }
    }
}
```

---

## Penjelasan Konfigurasi

### Load Balancer (Upstream Block)

```nginx
upstream voting_backend {
    least_conn;
    server 192.168.138.35:9031 ...;
    server 192.168.138.140:9032 ...;
    server 192.168.138.61:9033 ...;
    keepalive 64;
}
```

* **`least_conn`**: Metode pembagian beban berdasarkan jumlah koneksi aktif paling sedikit (lebih adil untuk server yang sedang sibuk).
* **`server`**: Tiga server backend dengan port berbeda.
* **`max_fails` dan `fail_timeout`**: Jika backend gagal diakses 3 kali dalam 30 detik, akan dianggap *down* sementara.
* **`keepalive 64`**: Mempertahankan hingga 64 koneksi terbuka ke backend untuk efisiensi.

---

### Reverse Proxy (Server Block)

```nginx
server {
    listen 9030;
    location / {
        proxy_pass http://voting_backend;
        ...
    }
}
```

* **`listen 9030`**: NGINX akan menerima request dari klien di port ini.
* **`proxy_pass`**: Meneruskan permintaan ke `voting_backend` yang didefinisikan di atas.

#### Forwarding Header

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

* Mengirim informasi asli dari klien ke backend, seperti alamat IP dan protokol.

#### Timeout dan Buffering

```nginx
proxy_connect_timeout 10s;
proxy_read_timeout 30s;
proxy_send_timeout 30s;
proxy_buffering on;
proxy_buffers 32 64k;
proxy_busy_buffers_size 128k;
proxy_temp_file_write_size 128k;
```

* Menyesuaikan waktu tunggu dan buffer agar performa stabil meski trafik tinggi.

---

