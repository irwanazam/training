Baik, berikut adalah langkah-langkah untuk menyediakan high availability (HA) untuk kluster RKE2 dengan menggunakan HAProxy atau Nginx sebagai external load balancer:

### 1. **Persediaan Awal:**
   - Anda perlu mempunyai 6 pelayan: 3 untuk master dan 3 untuk worker.
   - Pastikan semua pelayan telah dipasang dengan sistem operasi yang sesuai (contohnya, Ubuntu) dan telah dikemas kini.
   - Pasang RKE2 pada setiap pelayan.

### 2. **Pasang dan Konfigurasi HAProxy:**

**Pasang HAProxy:**
```bash
sudo apt-get update
sudo apt-get install -y haproxy
```

**Konfigurasi HAProxy:**
Edit fail konfigurasi HAProxy (`/etc/haproxy/haproxy.cfg`) seperti berikut:
```plaintext
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend kubernetes-frontend
    bind *:6443
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    balance roundrobin
    option tcp-check
    server master1 <MASTER1_IP>:6443 check
    server master2 <MASTER2_IP>:6443 check
    server master3 <MASTER3_IP>:6443 check
```
Gantikan `<MASTER1_IP>`, `<MASTER2_IP>`, dan `<MASTER3_IP>` dengan alamat IP pelayan master anda.

**Mulakan semula HAProxy:**
```bash
sudo systemctl restart haproxy
```

### 3. **Pasang dan Konfigurasi Nginx:**

**Pasang Nginx:**
```bash
sudo apt-get update
sudo apt-get install -y nginx
```

**Konfigurasi Nginx:**
Edit fail konfigurasi Nginx (`/etc/nginx/nginx.conf`) seperti berikut:
```plaintext
http {
    upstream kubernetes {
        least_conn;
        server <MASTER1_IP>:6443;
        server <MASTER2_IP>:6443;
        server <MASTER3_IP>:6443;
    }

    server {
        listen 6443;
        proxy_pass kubernetes;
    }
}
```
Gantikan `<MASTER1_IP>`, `<MASTER2_IP>`, dan `<MASTER3_IP>` dengan alamat IP pelayan master anda.

**Mulakan semula Nginx:**
```bash
sudo systemctl restart nginx
```

### 4. **Pasang RKE2 pada Pelayan Master dan Worker:**

**Pada setiap pelayan master dan worker:**
```bash
curl -sfL https://get.rke2.io | sh -
```

**Mulakan kluster pada pelayan master pertama:**
```bash
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml <<EOF
server: https://<LOAD_BALANCER_IP>:6443
token: <CLUSTER_TOKEN>
EOF

sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

Gantikan `<LOAD_BALANCER_IP>` dengan alamat IP load balancer anda dan `<CLUSTER_TOKEN>` dengan token kluster anda.

**Tambahkan pelayan master dan worker lain ke kluster:**
Ulangi langkah di atas pada setiap pelayan master dan worker lain, tetapi pastikan menggunakan `rke2-agent.service` untuk pelayan worker:
```bash
sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

### 5. **Semak Status Kluster:**
Pada pelayan master pertama, semak status kluster dengan:
```bash
sudo kubectl get nodes
```

Anda sepatutnya melihat semua pelayan master dan worker disenaraikan dan dalam keadaan `Ready`.

Ini adalah panduan asas untuk menyediakan high availability (HA) untuk kluster RKE2 menggunakan HAProxy atau Nginx sebagai load balancer. Anda mungkin perlu menyesuaikan beberapa konfigurasi berdasarkan keperluan khusus persekitaran anda.
