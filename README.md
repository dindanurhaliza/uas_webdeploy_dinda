# uas_webdeploy_dinda# 🚀 [Nama Aplikasi] — Production Deployment

> UAS Sistem Operasi (MITI.202) + Jaringan Komputer (MITI.203)  
> Kelas Sentul · Semester Genap 2025/2026 · STMIK Tazkia

---

## 👥 Anggota Kelompok

| Nama | NIM | Peran |
|------|-----|-------|
| [Nama 1] | [NIM] | [Misal: DevOps / Backend] |
| [Nama 2] | [NIM] | [Misal: Networking / DNS] |
| [Nama 3] | [NIM] | [Misal: Monitoring / CI-CD] |

---

## 📌 Tentang Aplikasi

> _Jelaskan singkat aplikasi yang di-deploy: apa fungsinya, dibuat dengan apa (ExpressJS / static HTML), dan tujuannya._

**Tech Stack:**
- Runtime: Node.js / ExpressJS _(sesuaikan)_
- Server: Nginx (reverse proxy) + Certbot (SSL)
- Container: Docker + Docker Compose
- CI/CD: GitHub Actions
- Monitoring: Uptime Kuma
- Backup: Cron job → Cloud Storage (S3 / R2 / B2)

---

## 🏗️ Architecture Diagram

```
Developer (git push)
        │
        ▼
  GitHub Repo (main branch)
        │
        ▼
  GitHub Actions CI/CD
  ┌─────────────────────────────┐
  │  1. Build Docker image      │
  │  2. Run tests               │
  │  3. SSH ke VPS              │
  │     docker compose pull     │
  │     docker compose up -d    │
  └─────────────────────────────┘
        │
        ▼  (SSH Deploy)
  ┌─────────────────────────────────────────┐
  │              VPS Server                  │
  │                                          │
  │  ┌──────────┐     ┌─────────────────┐   │
  │  │  Uptime  │────▶│   nginx         │   │
  │  │  Kuma    │     │  :80 / :443     │   │
  │  │(monitor) │     │  SSL via certbot│   │
  │  └──────────┘     └────────┬────────┘   │
  │                            │ proxy_pass  │
  │                    ┌───────▼────────┐   │
  │                    │ App Container  │   │
  │                    │ (Docker Compose│   │
  │                    └───────┬────────┘   │
  │                            │            │
  │             ┌──────────────┘            │
  │             ▼                           │
  │     Cron daily backup                   │
  └─────────────┬───────────────────────────┘
                │
                ▼
        Cloud Storage (S3/R2/B2)

User Browser → DNS (A Record) → IP VPS → HTTPS nginx → App
```

---

## ⚙️ Konfigurasi & Infrastruktur

| Komponen | Detail |
|----------|--------|
| Domain | `https://[domain-kamu.com]` |
| VPS Provider | [Contoh: DigitalOcean / Vultr / dll] |
| IP VPS | `[xxx.xxx.xxx.xxx]` |
| SSL | Let's Encrypt via Certbot |
| Port App | `[3000]` _(sesuaikan)_ |
| Monitoring URL | `http://[domain]:3001` _(Uptime Kuma)_ |

---

## 📁 Struktur Repository

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions CI/CD pipeline
├── app/
│   └── index.js                # Aplikasi utama (ExpressJS / static)
├── docker-compose.yml          # Container orchestration
├── Dockerfile                  # Build image aplikasi
├── nginx/
│   └── [domain].conf           # Konfigurasi nginx reverse proxy
├── .env.example                # Contoh environment variables (JANGAN commit .env asli)
├── scripts/
│   └── backup.sh               # Script backup harian
└── README.md
```

---

## 🔄 CI/CD Pipeline

File: `.github/workflows/deploy.yml`

**Alur pipeline saat `git push` ke `main`:**

1. GitHub Actions terpicu otomatis
2. Build Docker image dari `Dockerfile`
3. Jalankan test (jika ada)
4. SSH ke VPS menggunakan secret `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`
5. Eksekusi `docker compose pull && docker compose up -d`
6. Aplikasi versi terbaru berjalan di production

**GitHub Secrets yang dibutuhkan:**

| Secret | Keterangan |
|--------|------------|
| `VPS_HOST` | IP address VPS |
| `VPS_USER` | Username SSH (misal: `root` / `ubuntu`) |
| `VPS_SSH_KEY` | Private key SSH untuk akses VPS |
| `DOCKER_USERNAME` | Docker Hub username _(jika pakai registry)_ |
| `DOCKER_PASSWORD` | Docker Hub password / token |

---

## 📊 Monitoring

**Tool:** Uptime Kuma  
**Dashboard:** `http://[domain]:3001`

Uptime Kuma dikonfigurasi untuk polling endpoint aplikasi setiap **1 menit**.  
Alert dikirim via [Telegram / Email / dll] jika downtime terdeteksi.

**Cara cek status:**
```bash
# Cek container monitoring
docker ps | grep uptime-kuma

# Cek log uptime kuma
docker logs uptime-kuma
```

---

## 📋 Log Aplikasi

Aplikasi menulis log terstruktur ke stdout, dapat dibaca via:

```bash
# Via Docker logs
docker logs [nama-container-app] --tail 100 -f

# Via docker compose
docker compose logs app --tail 100 -f

# Filter log by waktu
docker logs [nama-container] --since "2025-01-01T14:00:00"
```

**Lokasi log nginx:**
```bash
# Access log
tail -f /var/log/nginx/access.log

# Error log
tail -f /var/log/nginx/error.log
```

---

## 💾 Backup

**Jadwal:** Setiap hari pukul 02:00 WIB  
**Tujuan:** Cloud Storage ([S3 / R2 / B2])

**Cek cron job:**
```bash
crontab -l
# Output: 0 2 * * * /home/[user]/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**Menjalankan backup manual:**
```bash
bash /home/[user]/scripts/backup.sh
```

**Verifikasi backup:**
```bash
# Cek log backup
tail -f /var/log/backup.log

# List file backup di cloud storage (contoh rclone)
rclone ls remote:bucket-name
```

**Restore from backup:**
```bash
# 1. Download backup dari cloud storage
rclone copy remote:bucket-name/backup-YYYY-MM-DD.tar.gz /tmp/

# 2. Ekstrak
tar -xzf /tmp/backup-YYYY-MM-DD.tar.gz -C /home/[user]/app/

# 3. Restart container
docker compose restart
```

---

## 📖 Runbook Operasional

### ▶️ Restart Aplikasi

```bash
# SSH ke VPS
ssh [user]@[IP-VPS]

# Restart container aplikasi
docker compose restart app

# Atau restart semua service
docker compose down && docker compose up -d

# Verifikasi
docker ps
curl -I https://[domain-kamu.com]
```

### ⏪ Rollback Deployment

**Opsi 1 — Git Revert (recommended):**
```bash
# Di local machine
git log --oneline          # Lihat commit history
git revert HEAD            # Revert commit terakhir
git push origin main       # Trigger CI/CD deploy versi sebelumnya
```

**Opsi 2 — Manual di VPS:**
```bash
# SSH ke VPS
ssh [user]@[IP-VPS]

# Pull image versi sebelumnya (jika pakai tag)
docker pull [username]/[image-name]:[previous-tag]

# Update docker-compose.yml image tag, lalu:
docker compose up -d
```

### 🔒 Renew SSL Certificate

```bash
# Cek status sertifikat
certbot certificates

# Renew manual
certbot renew --nginx

# Cek systemd timer auto-renew
systemctl status certbot.timer
```

### 🔥 Recovery Skenario Umum

<details>
<summary><strong>Container app down</strong></summary>

```bash
docker ps -a                        # Lihat semua container termasuk yang stop
docker compose start app            # Start ulang container
docker logs app --tail 50           # Cek log jika gagal start
```
</details>

<details>
<summary><strong>Nginx error / 502 Bad Gateway</strong></summary>

```bash
# Cek error log nginx
tail -f /var/log/nginx/error.log

# Verifikasi konfigurasi nginx
nginx -t

# Reload nginx
nginx -s reload

# Pastikan port app berjalan
docker ps | grep app
```
</details>

<details>
<summary><strong>Firewall memblokir akses</strong></summary>

```bash
ufw status                          # Cek status firewall
ufw allow 80                        # Buka port HTTP
ufw allow 443                       # Buka port HTTPS
ufw reload
```
</details>

<details>
<summary><strong>Disk full</strong></summary>

```bash
df -h                               # Cek penggunaan disk
du -sh /var/log/*                   # Cek ukuran log
# Hapus file junk
rm -f /var/log/junk
# Bersihkan docker images lama
docker system prune -a
```
</details>

<details>
<summary><strong>SSL Certificate hilang / error</strong></summary>

```bash
# Cek error di nginx
tail -f /var/log/nginx/error.log

# Re-issue sertifikat
certbot --nginx -d [domain-kamu.com]

# Reload nginx
nginx -s reload
```
</details>

<details>
<summary><strong>DNS tidak resolve / salah IP</strong></summary>

```bash
# Diagnosa
dig [domain-kamu.com]               # Lihat IP yang dikembalikan DNS
nslookup [domain-kamu.com]

# Jika IP salah → login ke registrar domain
# Ubah A Record ke IP VPS yang benar: [xxx.xxx.xxx.xxx]
# Tunggu propagasi DNS (bisa 5 menit - 24 jam)
```
</details>

<details>
<summary><strong>App env var corrupted / app error</strong></summary>

```bash
# Cek log aplikasi
docker logs app --tail 50

# Edit .env di VPS
nano /home/[user]/app/.env

# Restart container setelah fix
docker compose restart app
```
</details>

---

## 🌐 Environment Variables

File `.env` di VPS (`/home/[user]/app/.env`):

| Variable | Keterangan | Sensitif? |
|----------|------------|-----------|
| `PORT` | Port aplikasi berjalan | ❌ |
| `NODE_ENV` | Environment (production) | ❌ |
| `DATABASE_URL` | Connection string database | ✅ Ya |
| `API_KEY` | API key eksternal | ✅ Ya |

> ⚠️ **JANGAN pernah commit file `.env` ke repository!** Pastikan `.env` ada di `.gitignore`.

---

## 📚 Referensi

- [Docker Documentation](https://docs.docker.com)
- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Certbot (Let's Encrypt)](https://certbot.eff.org/)
- [Uptime Kuma](https://github.com/louislam/uptime-kuma)
- [LPI DevOps Tools Engineer](https://www.lpi.org/our-certifications/devops-overview)