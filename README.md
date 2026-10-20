# Hướng dẫn tự host GitLab Server cá nhân trên VPS

Toàn bộ quy trình này được thực hiện thành công trên:

| Hạng mục | Thông số ví dụ |
|---|---|
| OS | Ubuntu 22.04 LTS |
| VPS | 4 vCPU / 8GB RAM / 60GB disk |
| Domain | `gitlab.example.com` (đã trỏ DNS) |
| GitLab | CE 19.5 (Docker image `gitlab/gitlab-ce`) |
| Reverse proxy | Traefik v3 + Let's Encrypt (SSL tự động) |

> ⚠️ **Lưu ý bảo mật**: mọi giá trị trong guide dùng placeholder. Tuyệt đối không commit
> mật khẩu, token (PAT), IP VPS thật hay private key vào repo công khai.

---

## Mục lục

1. [Yêu cầu tiền đề](#1-yêu-cầu-tiền-đề)
2. [Chuẩn bị SSH](#2-chuẩn-bị-ssh)
3. [Tạo Swap (bắt buộc với VPS 4GB)](#3-tạo-swap)
4. [Cài Docker](#4-cài-docker)
5. [Reverse proxy Traefik + SSL](#5-reverse-proxy-traefik--ssl)
6. [Deploy GitLab](#6-deploy-gitlab)
7. [Kiểm tra GitLab hoạt động](#7-kiểm-tra-gitlab-hoạt-động)
8. [Cấu hình client: SSH key + push source](#8-cấu hình-client)
9. [Backup, cập nhật, dọn dẹp](#9-backup-cập-nhật-dọn-dẹp)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Yêu cầu tiền đề

- **VPS** ít nhất **4GB RAM** (khuyến nghị 8GB), Ubuntu 22.04/24.04 LTS
- **Domain** đã trỏ A record về IP VPS (vd: `gitlab.example.com` → IP VPS)
- Truy cập `root` qua SSH từ máy cá nhân
- Mở port: `22` (SSH), `80` và `443` (web/SSL) trên firewall của nhà cung cấp VPS

Kiểm tra nhanh:

```bash
ssh root@YOUR_VPS_IP
uname -a && lsb_release -d
free -h && df -h /
```

> 💡 **Kinh nghiệm thật**: nếu `~/.ssh/config` có nhiều block `Host` trùng tên,
> block **đầu tiên** quyết định `HostName`/`User`, các option khác lấy từ block
> sau → rất dễ kết nối nhầm port. Kiểm tra file này nếu SSH unexpected timeout!

---

## 2. Chuẩn bị SSH

### 2.1. Tạo key (nếu chưa có)

```bash
ssh-keygen -t ed25519 -C "bạn@domain-của-bạn"
```

### 2.2. Thêm key vào VPS

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@YOUR_VPS_IP
# nhập mật khẩu root một lần

# xác nhận login không cần mật khẩu:
ssh -o BatchMode=yes root@YOUR_VPS_IP "echo OK"
```

### 2.3. (Tùy chọn) Ghi chú trong `~/.ssh/config`

```sshconfig
Host myvps
  HostName YOUR_VPS_IP
  User root
  IdentityFile ~/.ssh/id_ed25519
```

---

## 3. Tạo Swap

GitLab Omnibus "ngốn" RAM lúc boot. VPS 4GB **nên** có thêm swap 4GB:

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab   # giữ lại sau reboot

free -h   # xác nhận Swap: 4Gi
```

---

## 4. Cài Docker

```bash
# Cách nhanh (Ubuntu)
curl -fsSL https://get.docker.com | sh

# Cho phép chạy docker không cần sudo (bắt buộc logout lại)
usermod -aG docker $USER

docker --version
```

---

## 5. Reverse proxy Traefik + SSL

GitLab **không nên** tự chiếm port 80/443 nếu VPS đã có reverse proxy.
Pattern phổ biến: **Traefik đọc Docker label** → tự cấp Let's Encrypt theo Host header.

### 5.1. Nếu VPS chưa có Traefik

Tạo `/root/traefik-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v3
    container_name: traefik
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=proxy
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.email=your@email.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge=true
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      - --api.dashboard=true
      - --api.insecure=true
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    restart: unless-stopped
    networks:
      - proxy

networks:
  proxy:
    external: true
```

```bash
mkdir -p /root/traefik && cd /root
docker network create proxy
docker compose -f traefik-compose.yml up -d
```

### 5.2. Nếu đã có Traefik sẵn

Chỉ cần đảm bảo network `proxy` tồn tại — container GitLab sẽ "ghép" vào bằng label:

```bash
docker network inspect proxy >/dev/null 2>&1 || docker network create proxy
```

---

## 6. Deploy GitLab

### 6.1. Tạo thư mục & compose file

```bash
mkdir -p /root/gitlab && cd /root/gitlab
nano docker-compose.yml   # dán nội dung dưới vào
```

**`/root/gitlab/docker-compose.yml`**:

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: unless-stopped
    hostname: gitlab.example.com
    environment:
      GITLAB_ROOT_EMAIL: "admin@example.com"
      GITLAB_ROOT_PASSWORD: "DoiMatKhauManhNgaySauLanDau_123!"   # ≥16 ký tự
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'

        # Nginx trong container chỉ serve HTTP, Traefik terminate TLS ở cổng 443
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        nginx['real_ip_trusted_addresses'] = ['127.0.0.1', '172.16.0.0/12', '10.0.0.0/8']
        nginx['real_ip_header'] = 'X-Forwarded-For'
        nginx['real_ip_recursive'] = 'on'

        # SSL do Traefik (Let's Encrypt) quản lý — tắt ACME nội bộ
        letsencrypt['enable'] = false

        # Git-over-SSH qua host port 2224 (port 22 của host giữ cho SSH system)
        gitlab_rails['gitlab_ssh_host'] = 'gitlab.example.com'
        gitlab_rails['gitlab_ssh_port'] = 2224

        # Tiết kiệm RAM (~1GB): tắt monitoring stack mặc định
        prometheus_monitoring['enable'] = false
        gitlab_rails['gitlab_email_enabled'] = false
        puma['worker_processes'] = 4
    ports:
      - "2224:22"          # git-over-SSH
    volumes:
      - ./config:/etc/gitlab
      - ./logs:/var/log/gitlab
      - ./data:/var/opt/gitlab
    networks:
      - proxy
    labels:
      - traefik.enable=true
      - traefik.docker.network=proxy
      # HTTP → redirect HTTPS
      - traefik.http.routers.gitlab-http.rule=Host(`gitlab.example.com`)
      - traefik.http.routers.gitlab-http.entrypoints=web
      - traefik.http.routers.gitlab-http.middlewares=gitlab-redirect-https
      - traefik.http.middlewares.gitlab-redirect-https.redirectscheme.scheme=https
      - traefik.http.middlewares.gitlab-redirect-https.redirectscheme.permanent=true
      # HTTPS
      - traefik.http.routers.gitlab.rule=Host(`gitlab.example.com`)
      - traefik.http.routers.gitlab.entrypoints=websecure
      - traefik.http.routers.gitlab.tls=true
      - traefik.http.routers.gitlab.tls.certresolver=letsencrypt
      - traefik.http.services.gitlab.loadbalancer.server.port=80

networks:
  proxy:
    external: true
```

### 6.2. Khởi động

```bash
cd /root/gitlab
docker compose up -d

# Theo dõi tới khi "healthy" (boot lần đầu mất 3–8 phút):
until [ "$(docker inspect -f '{{.State.Health.Status}}' gitlab)" = "healthy" ]; do
  echo "waiting..."; sleep 10
done
docker logs gitlab --tail 20
```

> ⚠️ **Lỗi thường gặp #1**: GitLab 17+ **đã gỡ một số config key cũ**
> (ví dụ `grafana['enable']` bị **removed** → `UnknownConfigOptionError` →
> container crash-loop `Up 2 seconds`). Nếu gặp, **xóa key lỗi** trong
> `GITLAB_OMNIBUS_CONFIG` rồi `docker compose up -d --force-recreate`.

### 6.3. Giải thích các config quan trọng

| Config | Tại sao cần |
|---|---|
| `external_url https://...` | GitLab tự sinh URL https, share link đúng |
| `nginx['listen_https']=false` | Tránh 2 bên cùng terminate SSL (Traefik đã làm) |
| `letsencrypt['enable']=false` | ACME nội bộ sẽ **fail** vì port 80 không thuộc GitLab |
| `gitlab_ssh_port=2224` | Host SSH system giữ port 22, GitLab SSH nhường chỗ |
| `prometheus_monitoring=false` | Tiết kiệm ~1GB RAM trên VPS nhỏ |
| `restart: unless-stopped` | Tự chạy lại khi VPS reboot |

---

## 7. Kiểm tra GitLab hoạt động

```bash
# 1. Web trả 200 (SSL từ Let's Encrypt qua Traefik):
curl -sI https://gitlab.example.com/users/sign_in | head -1

# 2. Container services:
docker exec gitlab gitlab-ctl status | grep -c "^run:"   # ~10+ services "run:"

# 3. Login lần đầu: https://gitlab.example.com
#    user: root / password: GITLAB_ROOT_PASSWORD đã đặt ở compose
```

**Dữ liệu nằm ở** (bắt buộc backup):

```
/root/gitlab/config   → /etc/gitlab      (gitlab.rb, secrets)
/root/gitlab/data     → /var/opt/gitlab  (database, repos, artifacts)
/root/gitlab/logs     → /var/log/gitlab
```

---

## 8. Cấu hình client

### 8.1. Thêm SSH key vào GitLab

**Cách A — qua UI**: đăng nhập → Avatar → **Edit profile** → **SSH Keys** → dán `cat ~/.ssh/id_ed25519.pub`.

**Cách B — qua API** (tiện cho script):

```bash
# 1. Tạo PAT trong UI: User Settings → Access Tokens → scope: api, write_repository
export GLT="glpat-xxxxxxxxxxxxxxxx"

# 2. Thêm public key
curl -s --request POST --header "PRIVATE-TOKEN: $GLT" \
  --data-urlencode "title=my-laptop" \
  --data-urlencode "key=$(cat ~/.ssh/id_ed25519.pub)" \
  https://gitlab.example.com/api/v4/user/keys

# 3. Tạo project
curl -s --request POST --header "PRIVATE-TOKEN: $GLT" \
  --data "name=my-project&visibility=private" \
  https://gitlab.example.com/api/v4/projects
```

### 8.2. Push source từ máy cá nhân

```bash
cd ~/projects/my-project
git remote add gitlab https://gitlab.example.com/group/my-project.git
git push gitlab main
```

**Lưu ý xác thực HTTPS**: lần đầu push, GitLab hỏi tài khoản —

- Username: `root` (hoặc username của bạn)
- Password: **KHÔNG phải mật khẩu login** → dùng **Personal Access Token (PAT)**

Lưu credential vào Keychain (macOS) để không nhập lại:

```bash
git credential-osxkeychain store <<'EOF'
protocol=https
host=gitlab.example.com
username=bạn
password=glpat-xxxxxxxxxxxxxxxx
EOF
```

### 8.3. Git over SSH (port 2224)

```bash
git remote add gitlab-ssh ssh://git@gitlab.example.com:2224/group/my-project.git
git push gitlab-ssh main
```

> ⚠️ **Kinh nghiệm thật**: một số mạng ISP **filter kết nối TCP tới port lạ**
> (chỉ cho 80/443/22 qua) → SSH 2224 có thể **timeout/RST** dù server hoàn
> toàn bình thường. Triệu chứng: `nc -zv IP 2224` OK nhưng
> `ssh -p 2224 -T git@IP` treo ở "banner exchange".
>
> → **Fallback chắc chắn: dùng git over HTTPS (443)** như mục 8.2.
> Test thử bằng hotspot 4G nếu muốn dùng SSH.

---

## 9. Backup, cập nhật, dọn dẹp

### Backup (nên chạy định kỳ + thêm cron)

```bash
docker exec gitlab gitlab-backup create
# File backup: /root/gitlab/data/backups/
# đồng thời backup /root/gitlab/config (secrets!) riêng

# Cron example: 2h sáng chủ nhật
# 0 2 * * 0 docker exec gitlab gitlab-backup create GITLAB_BACKUP_DIR=/var/opt/gitlab/backups
```

### Cập nhật GitLab

```bash
cd /root/gitlab
docker compose pull
docker compose up -d          # recreate với image mới
docker exec gitlab gitlab-ctl reconfigure
```

### Dọn dẹp disk (khi chật)

```bash
# Xem dung lượng
df -h / && du -sh /var/log/* | sort -rh | head

# Journal logs (thường phình nhất)
journalctl --vacuum-size=300M

# Docker cache
docker builder prune -af
docker system prune -f          # KHÔNG dùng -a để giữ image dự phòng
# Không prune volume nếu có dữ liệu quan trọng!
```

---

## 10. Troubleshooting

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| Container `Up 2 seconds` lặp lại | Config key không tồn tại trong GitLab version hiện tại (`UnknownConfigOptionError`) | Xóa key lỗi trong `GITLAB_OMNIBUS_CONFIG`, `docker compose up -d --force-recreate` |
| Web trả **502** | Puma/nginx chưa boot xong hoặc reconfigure đang chạy | Chờ 2–3 phút, xem `docker exec gitlab gitlab-ctl status puma` |
| Web trả **502 lâu dài** | Traefik sai network/port backend | Kiểm tra label `traefik.docker.network=proxy` và `loadbalancer.server.port=80` |
| **Không có cert** / ACME 403 | DNS chưa trỏ về VPS, hoặc Cloudflare proxy chặn HTTP-01 challenge | `dig gitlab.example.com` phải ra IP VPS; thử tắt proxy DNS (grey cloud) |
| `Permission denied (publickey)` khi push SSH | Key chưa thêm vào GitLab / sai port | Kiểm tra key trong UI GitLab, port `2224` |
| SSH 2224 treo "banner exchange" | ISP filter port lạ (xem 8.3) | Dùng git over HTTPS |
| Container chiếm port 80/443 | Đã có reverse proxy | GitLab **không** map 80/443 (chỉ qua Traefik labels) |
| RAM đầy lúc boot | Thiếu swap | Làm mục 3 |

---

## Kiến trúc tham khảo

```
Internet
   │
   ├─ :80/:443 ──► Traefik (SSL Let's Encrypt)
   │                    │  Host(`gitlab.example.com`)
   │                    ▼
   │              GitLab container (nginx:80 → puma)
   │
   ├─ :2224 ─────► GitLab container (sshd, git-over-SSH)
   │
   └─ :22 ───────► Host SSH (quản trị VPS)
```

---

## Tham khảo

- [GitLab Omnibus configuration](https://docs.gitlab.com/omnibus/settings/configuration/)
- [GitLab Docker image](https://docs.gitlab.com/ee/install/docker.html)
- [Traefik & Docker provider](https://doc.traefik.io/traefik/providers/docker/)
- [Let's Encrypt HTTP-01 challenge](https://letsencrypt.org/docs/challenge-types/)

> Guide dựa trên quy trình deploy thực tế: Ubuntu 22.04 + Traefik v3 +
> GitLab CE 19.5 Docker. Nếu bạn gặp lỗi mới, hãy mở Issue/PR!
