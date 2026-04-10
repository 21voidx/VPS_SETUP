# 📋 Dokumentasi Setup VPS — Lengkap

> **Sistem:** Ubuntu 24.04 LTS  
> **Dibuat:** Februari 2026  
> **Status:** Ready

---

## Daftar Isi

1. [Update & Install Paket Dasar](#1-update--install-paket-dasar)
2. [Tambah User & Hak Akses](#2-tambah-user--hak-akses)
3. [Setup SSH Public Key](#3-setup-ssh-public-key)
4. [Konfigurasi OpenSSH](#4-konfigurasi-openssh)
5. [Tailscale VPN Tunnel](#5-tailscale-vpn-tunnel)
6. [Install Docker](#6-install-docker)
7. [Install k3s (Kubernetes)](#7-install-k3s-kubernetes)
8. [Setup GitLab Runner](#8-setup-gitlab-runner)
9. [Setup GitHub Actions Runner](#9-setup-github-actions-runner)
10. [Setup NFS — WSL2 Server & VPS Client](#10-setup-nfs--wsl2-server--vps-client)
11. [Referensi Perintah](#11-referensi-perintah)

---

## 1. Update & Install Paket Dasar

Update sistem dan install tools yang diperlukan sebelum konfigurasi apapun.

```bash
# Update package list dan upgrade semua paket
sudo apt update && sudo apt upgrade -y

# Install paket-paket dasar
sudo apt install -y curl ufw nano
```

**Verifikasi:**
```bash
curl --version
ufw --version
nano --version
```

---

## 2. Tambah User & Hak Akses

Membuat user baru `void` dan memberikan akses sudo agar tidak selalu login sebagai root.

```bash
# Buat user baru
adduser void

# Tambahkan ke grup sudo (akses root)
usermod -aG sudo void

# Verifikasi user berhasil dibuat
id void
```

**Pindah ke user void:**
```bash
su - void
# atau login langsung via SSH dengan: ssh void@IP_VPS
```

> ⚠️ **Catatan:** Setelah setup SSH key selesai, disarankan menonaktifkan login root langsung.

---

## 3. Setup SSH Public Key

Menggunakan key-based authentication agar lebih aman dibanding password.

### Di Laptop (Local Machine)

```bash
# Generate SSH key pair (jika belum ada)
ssh-keygen -t ed25519 -C "laptop-void-vps"

# Lihat public key yang akan di-copy ke VPS
cat ~/.ssh/id_ed25519.pub
```

### Di VPS — Untuk User root

```bash
# Login sebagai root
mkdir -p /root/.ssh
chmod 700 /root/.ssh

# Paste public key dari laptop
nano /root/.ssh/authorized_keys
# → paste isi id_ed25519.pub, save

chmod 600 /root/.ssh/authorized_keys
```

### Di VPS — Untuk User void

```bash
# Login sebagai void (atau su - void)
mkdir -p /home/void/.ssh
chmod 700 /home/void/.ssh

# Paste public key dari laptop
nano /home/void/.ssh/authorized_keys
# → paste isi id_ed25519.pub, save

chmod 600 /home/void/.ssh/authorized_keys
chown -R void:void /home/void/.ssh
```

**Test koneksi dari laptop:**
```bash
ssh root@IP_VPS
ssh void@IP_VPS
```

---

## 4. Konfigurasi OpenSSH

Mengamankan SSH server agar tidak mudah di-brute force.

### Edit konfigurasi SSH

```bash
sudo nano /etc/ssh/sshd_config
```

**Ubah/tambahkan baris berikut:**

```conf
# Nonaktifkan login password (gunakan key saja)
PasswordAuthentication no

# Nonaktifkan login root via password (root hanya bisa via key)
PermitRootLogin prohibit-password

# Batasi auth attempts
MaxAuthTries 3

# Port default (bisa diganti untuk keamanan tambahan)
Port 22

# Aktifkan public key auth
PubkeyAuthentication yes
```

### Restart SSH

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

> ⚠️ **Penting:** Jangan close terminal yang sedang aktif sebelum test koneksi baru berhasil!

**Test dari terminal baru:**
```bash
ssh void@IP_VPS
```

---

## 5. Tailscale VPN Tunnel

Tailscale membuat VPN mesh antar device, sehingga VPS bisa diakses via IP Tailscale tanpa expose port ke internet publik.

### Install Tailscale

```bash
# Download & install via script resmi
curl -fsSL https://tailscale.com/install.sh | sh
```

### Aktifkan & Login

```bash
# Mulai Tailscale dan buka link untuk login
sudo tailscale up

# Akan muncul URL → buka di browser → login dengan akun Tailscale
```

### Verifikasi

```bash
# Cek IP Tailscale yang diberikan ke VPS
tailscale ip -4

# Cek status koneksi
tailscale status
```

### Akses VPS via Tailscale

```bash
# Dari laptop (yang juga terhubung Tailscale)
ssh void@100.x.x.x   # IP Tailscale VPS

# Atau tambahkan ke ~/.ssh/config di laptop:
# Host vps-tailscale
#   HostName 100.x.x.x
#   User void
#   IdentityFile ~/.ssh/id_ed25519
```

### Auto-start saat boot

```bash
sudo systemctl enable tailscaled
sudo systemctl status tailscaled
```

---

## 6. Install Docker

### Tambah Repository Resmi Docker

```bash
# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Tambah GPG key Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Tambah repository Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### Konfigurasi Post-Install

```bash
# Jalankan Docker tanpa sudo
sudo usermod -aG docker $USER
newgrp docker

# Aktifkan Docker saat boot
sudo systemctl enable docker
sudo systemctl start docker
```

### Verifikasi

```bash
docker --version
docker compose version
docker run hello-world
```

---

## 7. Install k3s (Kubernetes)

k3s adalah distribusi Kubernetes yang ringan, cocok untuk single-node VPS.

### Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

k3s otomatis berjalan sebagai systemd service setelah instalasi.

### Setup kubectl tanpa sudo

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

### Verifikasi

```bash
sudo systemctl status k3s
kubectl get nodes
kubectl get pods -A
```

### Akses dari Laptop (Remote kubectl)

```bash
# Di laptop, copy kubeconfig dari VPS
scp void@IP_VPS:/etc/rancher/k3s/k3s.yaml ~/.kube/config-vps

# Ganti IP 127.0.0.1 dengan IP VPS atau IP Tailscale
sed -i 's/127.0.0.1/IP_VPS_ATAU_TAILSCALE/g' ~/.kube/config-vps

# Set sebagai default config
export KUBECONFIG=~/.kube/config-vps

# Test
kubectl get nodes
```

### Komponen Bawaan k3s

| Komponen | Keterangan |
|---|---|
| Traefik | Ingress controller default |
| CoreDNS | DNS internal cluster |
| Local Path Provisioner | Persistent storage lokal |
| Flannel | CNI networking |
| metrics-server | Resource monitoring |

### Contoh Deploy Pertama

```yaml
# nginx-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f nginx-test.yaml
kubectl get svc          # lihat port yang di-assign
kubectl get pods         # pastikan status Running
```

---

## 8. Setup GitLab Runner

GitLab Runner adalah agent yang menjalankan CI/CD pipeline dari GitLab. Runner akan berjalan di VPS dan mendengarkan job dari GitLab.com atau GitLab self-hosted.

### Install GitLab Runner

```bash
# Tambah repository resmi GitLab
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# Install GitLab Runner
sudo apt install -y gitlab-runner

# Verifikasi
gitlab-runner --version
sudo systemctl status gitlab-runner
```

### Register Runner ke GitLab

#### Dapatkan Registration Token

Masuk ke GitLab → project/group → **Settings → CI/CD → Runners → New project runner**, lalu copy token yang diberikan.

#### Register Runner

```bash
sudo gitlab-runner register
```

Isi prompt yang muncul:

```
Enter the GitLab instance URL:
→ https://gitlab.com   (atau URL GitLab self-hosted)

Enter the registration token:
→ glrt-xxxxxxxxxxxx   (token dari GitLab)

Enter a description for the runner:
→ vps-runner

Enter tags for the runner (comma-separated):
→ vps, docker, production   (opsional, untuk filter job)

Enter optional maintenance note for the runner:
→ (enter / kosongkan)

Enter an executor:
→ docker   (pilih executor yang sesuai)

Enter the default Docker image:
→ docker:latest
```

#### Atau Register Langsung via Flag (Non-Interactive)

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com" \
  --token "glrt-xxxxxxxxxxxx" \
  --executor "docker" \
  --docker-image "docker:latest" \
  --description "vps-runner" \
  --tag-list "vps,docker"
```

### Konfigurasi Runner (config.toml)

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Contoh konfigurasi dengan Docker executor yang bisa build Docker image:

```toml
concurrent = 4

[[runners]]
  name = "vps-runner"
  url = "https://gitlab.com"
  token = "glrt-xxxxxxxxxxxx"
  executor = "docker"

  [runners.docker]
    image = "docker:latest"
    privileged = true               # Diperlukan untuk Docker-in-Docker
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    pull_policy = "if-not-present"  # Hemat bandwidth, pakai cache jika ada
```

> ⚠️ **Catatan `privileged = true`:** Diperlukan jika pipeline perlu build Docker image. Jangan aktifkan jika tidak perlu untuk keamanan.

### Restart & Verifikasi

```bash
sudo gitlab-runner restart
sudo gitlab-runner status

# Lihat semua runner yang terdaftar
sudo gitlab-runner list

# Verifikasi runner terhubung ke GitLab
sudo gitlab-runner verify
```

### Auto-start saat Boot

```bash
sudo systemctl enable gitlab-runner
sudo systemctl status gitlab-runner
```

### Contoh `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — contoh pipeline yang berjalan di VPS runner

stages:
  - build
  - deploy

build-image:
  stage: build
  tags:
    - vps          # pastikan pakai runner VPS
  script:
    - docker build -t myapp:$CI_COMMIT_SHORT_SHA .
    - docker tag myapp:$CI_COMMIT_SHORT_SHA myapp:latest

deploy:
  stage: deploy
  tags:
    - vps
  only:
    - main
  script:
    - docker compose pull
    - docker compose up -d --force-recreate
```

### Mengelola Runner

```bash
# Stop runner sementara (pause)
sudo gitlab-runner stop

# Hapus runner dari VPS (tidak menghapus dari GitLab)
sudo gitlab-runner unregister --name "vps-runner"

# Hapus semua runner
sudo gitlab-runner unregister --all-runners
```

---

## 9. Setup GitHub Actions Runner

GitHub Actions Self-Hosted Runner memungkinkan workflow GitHub Actions berjalan langsung di VPS, bukan di server GitHub.

### Buat Direktori Runner

```bash
# Buat user khusus runner (opsional tapi direkomendasikan)
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner
sudo usermod -aG sudo github-runner

# Buat direktori kerja
sudo mkdir -p /opt/github-runner
sudo chown github-runner:github-runner /opt/github-runner
sudo su - github-runner
cd /opt/github-runner
```

### Download Runner

Buka GitHub → repository → **Settings → Actions → Runners → New self-hosted runner** → pilih **Linux x64**, lalu ikuti perintah yang ditampilkan. Contohnya:

```bash
# Download versi terbaru (cek versi aktual di halaman GitHub)
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.323.0/actions-runner-linux-x64-2.323.0.tar.gz

# Extract
tar xzf actions-runner-linux-x64.tar.gz
```

### Register Runner ke GitHub

```bash
# Token didapat dari halaman GitHub Settings → Actions → Runners
./config.sh \
  --url https://github.com/USERNAME/REPO \
  --token YOUR_TOKEN_FROM_GITHUB
```

Isi prompt yang muncul:

```
Enter the name of the runner group:
→ (enter / default)

Enter the name of runner:
→ vps-runner

Enter any additional labels:
→ vps, self-hosted, linux   (opsional)

Enter name of work folder:
→ (enter / default: _work)
```

### Jalankan sebagai Systemd Service

Agar runner otomatis berjalan saat boot, install sebagai service:

```bash
# Keluar dari user github-runner dulu
exit

# Install service (jalankan sebagai sudo dari user void)
sudo /opt/github-runner/svc.sh install github-runner

# Start service
sudo /opt/github-runner/svc.sh start

# Verifikasi
sudo /opt/github-runner/svc.sh status
```

Atau cara manual membuat systemd service:

```bash
sudo nano /etc/systemd/system/github-runner.service
```

```ini
[Unit]
Description=GitHub Actions Runner
After=network.target

[Service]
ExecStart=/opt/github-runner/run.sh
User=github-runner
WorkingDirectory=/opt/github-runner
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=5min
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable github-runner
sudo systemctl start github-runner
sudo systemctl status github-runner
```

### Verifikasi di GitHub

Buka **GitHub → Settings → Actions → Runners** — runner VPS akan muncul dengan status **Idle** jika berhasil terhubung.

### Contoh Workflow GitHub Actions

```yaml
# .github/workflows/deploy.yml

name: Deploy to VPS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted    # pakai VPS runner

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Deploy dengan Docker Compose
        run: |
          docker compose pull
          docker compose up -d --force-recreate

      - name: Verifikasi deploy
        run: docker compose ps
```

### Mengelola Runner

```bash
# Cek status
sudo systemctl status github-runner

# Restart runner
sudo systemctl restart github-runner

# Stop runner
sudo systemctl stop github-runner

# Hapus runner dari VPS
cd /opt/github-runner
./config.sh remove --token YOUR_REMOVE_TOKEN
```

### Update Runner

```bash
# Stop service dulu
sudo systemctl stop github-runner

# Masuk ke direktori runner
cd /opt/github-runner

# Download versi terbaru
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz

# Extract (timpa file lama)
tar xzf actions-runner-linux-x64.tar.gz

# Start kembali
sudo systemctl start github-runner
```

---

## Perbandingan GitLab Runner vs GitHub Actions Runner

| Aspek | GitLab Runner | GitHub Actions Runner |
|---|---|---|
| Platform | GitLab.com / self-hosted | GitHub.com |
| Config file | `/etc/gitlab-runner/config.toml` | `/opt/github-runner/` |
| Pipeline file | `.gitlab-ci.yml` | `.github/workflows/*.yml` |
| Executor | Docker, Shell, k8s, dll | Shell (Docker via actions) |
| Tag runner | `tags: [vps]` | `runs-on: self-hosted` |
| Service | `gitlab-runner` | `github-runner` |
| Concurrent jobs | Bisa dikonfigurasi | Per runner = 1 job |

---

## 10. Setup NFS — WSL2 Server & VPS Client

NFS (Network File System) memungkinkan VPS me-mount direktori dari WSL2 di laptop lokal sebagai shared storage. Koneksi menggunakan jaringan Tailscale sehingga tidak perlu expose port NFS ke internet publik.

**Topologi:**
```
WSL2 (laptop)  ──── Tailscale ────  VPS
nfs-kernel-server                   nfs-common
100.x.x.x (server)                  100.y.y.y (client)
```

> ⚠️ **Prasyarat:** Tailscale sudah aktif dan terhubung di kedua sisi (WSL2 & VPS). Jalankan `tailscale ip -4` di masing-masing sisi untuk mendapatkan IP Tailscale sebelum mulai.

---

### A. Setup di WSL2 (NFS Server)

#### 1. Aktifkan Systemd di WSL2

NFS server membutuhkan systemd. Pastikan sudah diaktifkan di WSL2.

```bash
# Cek apakah systemd sudah aktif
ps -p 1 -o comm=
# → jika output "systemd", sudah aktif. Jika "init", perlu diaktifkan.
```

Jika belum aktif, edit konfigurasi WSL:

```bash
sudo nano /etc/wsl.conf
```

Tambahkan:

```ini
[boot]
systemd=true
```

Kemudian **restart WSL2** dari PowerShell/CMD Windows:

```powershell
wsl --shutdown
wsl
```

#### 2. Install nfs-kernel-server

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

**Verifikasi:**

```bash
sudo systemctl status nfs-kernel-server
nfsstat --version
```

#### 3. Buat Direktori yang Akan Di-share

```bash
# Buat direktori shared (sesuaikan path dengan kebutuhan)
mkdir -p ~/nfs-share

# Contoh: share direktori project
# mkdir -p ~/projects/shared
```

#### 4. Konfigurasi /etc/exports

```bash
sudo nano /etc/exports
```

Tambahkan baris berikut (ganti `100.y.y.y` dengan **IP Tailscale VPS**):

```exports
# Format: <path> <client_ip>(<options>)
/home/<user>/nfs-share  100.y.y.y(rw,sync,no_subtree_check,no_root_squash)
```

**Penjelasan opsi:**

| Opsi | Keterangan |
|---|---|
| `rw` | Client bisa baca dan tulis |
| `sync` | Tulis ke disk sebelum konfirmasi ke client (lebih aman) |
| `no_subtree_check` | Nonaktifkan subtree checking, meningkatkan stabilitas |
| `no_root_squash` | Root di client diperlakukan sebagai root di server |

> ⚠️ **Keamanan:** Gunakan `root_squash` (default) jika VPS tidak sepenuhnya dipercaya. `no_root_squash` mempermudah operasi file tapi memberikan akses root penuh dari client.

#### 5. Terapkan Konfigurasi Exports

```bash
# Export semua direktori dari /etc/exports
sudo exportfs -arv

# Verifikasi direktori yang di-export
sudo exportfs -v
```

#### 6. Jalankan & Aktifkan NFS Server

```bash
sudo systemctl enable nfs-kernel-server
sudo systemctl start nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

#### 7. Cek IP Tailscale WSL2

```bash
# Catat IP ini — dibutuhkan di sisi VPS
tailscale ip -4
# Contoh output: 100.x.x.x
```

---

### B. Setup di VPS (NFS Client)

#### 1. Install nfs-common

```bash
sudo apt update
sudo apt install -y nfs-common
```

**Verifikasi:**

```bash
showmount --version
```

#### 2. Buat Mount Point

```bash
# Buat direktori tempat NFS akan di-mount
sudo mkdir -p /mnt/wsl-share

# Sesuaikan ownership jika perlu
sudo chown void:void /mnt/wsl-share
```

#### 3. Test Mount Manual

Sebelum setup otomatis, pastikan koneksi berhasil:

```bash
# Cek export yang tersedia dari WSL2 (ganti dengan IP Tailscale WSL2)
showmount -e 100.x.x.x

# Mount manual
sudo mount -t nfs 100.x.x.x:/home/<user>/nfs-share /mnt/wsl-share

# Verifikasi mount berhasil
df -h | grep wsl-share
ls /mnt/wsl-share
```

#### 4. Mount Otomatis via /etc/fstab

Agar NFS ter-mount otomatis saat VPS boot:

```bash
sudo nano /etc/fstab
```

Tambahkan baris berikut di akhir file:

```fstab
# NFS mount dari WSL2 via Tailscale
100.x.x.x:/home/<user>/nfs-share  /mnt/wsl-share  nfs  defaults,_netdev,nofail,soft,timeo=30  0  0
```

**Penjelasan opsi fstab:**

| Opsi | Keterangan |
|---|---|
| `_netdev` | Tunggu jaringan sebelum mount saat boot |
| `nofail` | Boot tetap lanjut meski mount gagal |
| `soft` | Return error jika server tidak merespons (tidak hang) |
| `timeo=30` | Timeout 3 detik per retry (unit: 1/10 detik) |

**Test fstab tanpa reboot:**

```bash
sudo mount -a
df -h | grep wsl-share
```

#### 5. Unmount

```bash
# Unmount manual jika diperlukan
sudo umount /mnt/wsl-share

# Paksa unmount jika sedang digunakan
sudo umount -l /mnt/wsl-share
```

---

### C. Troubleshooting NFS

**Mount gagal — "Connection refused" atau timeout:**
```bash
# Di WSL2: pastikan NFS server berjalan
sudo systemctl status nfs-kernel-server

# Di WSL2: pastikan port NFS (2049) terbuka
sudo ss -tlnp | grep 2049

# Test koneksi dari VPS ke WSL2
nc -zv 100.x.x.x 2049
```

**Mount gagal — "Access denied":**
```bash
# Di WSL2: cek apakah IP VPS sudah benar di /etc/exports
sudo exportfs -v

# Di WSL2: reload exports setelah perubahan
sudo exportfs -arv
sudo systemctl restart nfs-kernel-server
```

**WSL2 restart menyebabkan IP Tailscale berubah:**
```bash
# Di WSL2: cek IP Tailscale terkini
tailscale ip -4

# Update /etc/exports di WSL2 dengan IP baru
sudo nano /etc/exports
sudo exportfs -arv

# Update /etc/fstab di VPS dengan IP baru
sudo nano /etc/fstab
sudo umount /mnt/wsl-share
sudo mount -a
```

> 💡 **Tips:** Gunakan **Tailscale MagicDNS** untuk menghindari masalah IP berubah. Ganti IP Tailscale dengan hostname Tailscale (misal `mylaptop.tail12345.ts.net`) di `/etc/exports` dan `/etc/fstab`.

---

## 11. Referensi Perintah

### NFS (WSL2 — Server)

```bash
sudo systemctl status nfs-kernel-server   # Cek status NFS server
sudo systemctl restart nfs-kernel-server  # Restart NFS server
sudo exportfs -arv                         # Reload & tampilkan semua exports
sudo exportfs -v                           # Lihat exports yang aktif
sudo exportfs -u 100.y.y.y:/path          # Unexport direktori tertentu
```

### NFS (VPS — Client)

```bash
showmount -e 100.x.x.x                    # Lihat exports dari server
sudo mount -t nfs 100.x.x.x:/path /mnt/point  # Mount manual
sudo umount /mnt/point                    # Unmount
sudo mount -a                             # Mount semua entri di fstab
df -h | grep nfs                          # Verifikasi NFS ter-mount
```

### GitLab Runner

```bash
sudo gitlab-runner status              # Cek status
sudo gitlab-runner restart             # Restart runner
sudo gitlab-runner list                # Daftar runner terdaftar
sudo gitlab-runner verify              # Verifikasi koneksi ke GitLab
sudo gitlab-runner unregister --all-runners  # Hapus semua runner
```

### GitHub Actions Runner

```bash
sudo systemctl status github-runner   # Cek status
sudo systemctl restart github-runner  # Restart runner
sudo systemctl stop github-runner     # Stop runner
cat /opt/github-runner/.runner        # Lihat info runner
```

---


```bash
ssh void@IP_VPS                    # Login via IP publik
ssh void@100.x.x.x                 # Login via Tailscale
sudo systemctl restart ssh         # Restart SSH service
```

### UFW Firewall

```bash
sudo ufw status                    # Cek status firewall
sudo ufw allow 22/tcp              # Izinkan SSH
sudo ufw allow 80/tcp              # Izinkan HTTP
sudo ufw allow 443/tcp             # Izinkan HTTPS
sudo ufw allow 6443/tcp            # Izinkan k3s API
sudo ufw enable                    # Aktifkan firewall
sudo ufw disable                   # Nonaktifkan firewall
```

### Tailscale

```bash
tailscale status                   # Cek semua device
tailscale ip -4                    # Lihat IP Tailscale
sudo tailscale up                  # Koneksikan ke jaringan
sudo tailscale down                # Disconnect
```

### Docker

```bash
docker ps                          # Container yang berjalan
docker ps -a                       # Semua container
docker images                      # Daftar image
docker compose up -d               # Jalankan compose (background)
docker compose down                # Stop & hapus container
docker compose logs -f             # Lihat log realtime
docker system prune -a             # Bersihkan resource tidak terpakai
```

### k3s / kubectl

```bash
kubectl get nodes                  # Lihat semua node
kubectl get pods -A                # Lihat semua pod
kubectl get svc -A                 # Lihat semua service
kubectl apply -f file.yaml         # Deploy dari file
kubectl delete -f file.yaml        # Hapus resource
kubectl logs <pod-name>            # Log pod
kubectl describe pod <pod-name>    # Detail pod
kubectl exec -it <pod-name> -- sh  # Masuk ke dalam pod
sudo systemctl restart k3s         # Restart k3s
sudo systemctl status k3s          # Status k3s
```

---

## Arsitektur Setup

```
LAPTOP (WSL2)              GITLAB.COM / GITHUB.COM
  │  nfs-kernel-server            │
  │  Tailscale: 100.x.x.x         │
  │                               │
  ├─── SSH (port 22) ─────────────────────────────► VPS PUBLIC IP
  │                               │ (CI/CD job trigger)
  └─── Tailscale VPN (mesh) ─────►│◄──────────────► VPS TAILSCALE IP (100.y.y.y)
         │  NFS (port 2049)        │                       │
         └────────────────────────────────────────────────►│
                                  │ Pipeline trigger   ┌───┴─────────────────┐
                                  └───────────────────►│       VPS           │
                                                       │─────────────────────│
                                                       │  Docker             │
                                                       │  k3s                │
                                                       │  GitLab Runner      │
                                                       │  GitHub Runner      │
                                                       │  UFW                │
                                                       │  nfs-common         │
                                                       │  /mnt/wsl-share ◄── │── NFS mount
                                                       └─────────────────────┘
```

---

*Dokumentasi ini mencakup seluruh setup yang telah dilakukan. Simpan sebagai referensi untuk konfigurasi ulang atau onboarding ke VPS baru.*
