# HRM Labs DevOps Assessment — Câu trả lời lý thuyết

Ứng viên: [Đoàn Trần Hải Nguyên]
Ngày nộp: [25-9-2026]

============================================================
PHẦN A — Linux & Quản trị Server
============================================================

Câu 1: Các lệnh Linux để điều tra vấn đề

top                    # Xem CPU, memory, process real-time
htop                   # Giống top nhưng trực quan hơn
uptime                 # Xem load average
vmstat 1 5             # Thống kê CPU, memory, I/O mỗi 1s, 5 lần
iostat -x 1 5          # I/O disk
df -h                  # Dung lượng các phân vùng
du -sh /var/*          # Dung lượng từng thư mục trong /var
free -h                # RAM và swap
cat /proc/meminfo      # Chi tiết memory
ps aux --sort=-%cpu | head -10   # Top 10 process dùng CPU
ps aux --sort=-%mem | head -10   # Top 10 process dùng RAM
journalctl -xe         # Log hệ thống gần đây
tail -f /var/log/syslog
dmesg | tail           # Log kernel (kiểm tra OOM killer)

Giải thích: Khi CPU 92%, RAM 87%, Disk 94% và response time tăng từ 200ms lên 5-10s, cần điều tra theo thứ tự: tài nguyên → process → I/O → log.

Câu 2: Xác định process ngốn CPU nhiều nhất

top -o %CPU
ps aux --sort=-%cpu | head -10
pidstat -u 1 5

Process đứng đầu danh sách là thủ phạm chính.

Câu 3: Kiểm tra memory usage

free -h
cat /proc/meminfo
ps aux --sort=-%mem | head -10
smem -rs memory

Câu 4: Kiểm tra thư mục/file ngốn disk nhiều nhất

df -h
du -sh /* 2>/dev/null | sort -rh | head -10
du -ah /var | sort -rh | head -20
ncdu /
find / -type f -size +100M -exec ls -lh {} \;

Câu 5: Xử lý khi disk gần đầy

du -sh /* | sort -rh
lsof +L1
journalctl --vacuum-size=200M
apt clean
docker system prune -a
find /var/log -name "*.log" -mtime +30 -delete
truncate -s 0 /var/log/large.log
logrotate -f /etc/logrotate.conf

Câu 6: Logs cần kiểm tra

journalctl -xe
journalctl -u <service> --since "30 min ago"
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log
tail -f /var/log/syslog
dmesg | grep -i "oom\|error"
docker logs <container>

============================================================
PHẦN B — Git
============================================================

Quy trình đưa feature/login vào môi trường development

git checkout develop
git pull origin develop
git checkout feature/login
git rebase develop
git push origin feature/login
# Tạo Pull Request: feature/login → develop
# Review, CI pass, approve
git checkout develop
git merge --no-ff feature/login
git push origin develop
git branch -d feature/login
git push origin --delete feature/login

Câu 7: git merge vs git rebase

git merge:
- Tạo merge commit, giữ nguyên history
- An toàn với branch đã push
- History phức tạp, nhiều nhánh
- Dùng khi merge feature vào main

git rebase:
- Viết lại history, commit phẳng
- Không nên dùng với branch public
- History tuyến tính, sạch
- Dùng khi cập nhật feature branch với main

Câu 8: Pull Request là gì?

Là yêu cầu merge code từ branch này sang branch khác trên nền tảng Git (GitHub/GitLab). Cho phép review code, chạy CI/CD, thảo luận comment, kiểm soát chất lượng trước khi merge.

Câu 9: Tại sao tránh push trực tiếp lên main?

- main là branch production-ready, cần ổn định
- Bypass review, dễ đưa bug lên production
- Không có CI/CD check
- Không có audit trail
- Khó rollback
- Giải pháp: Branch protection rules, required reviews, required status checks

Câu 10: Xử lý merge conflict

git checkout develop
git merge feature/login
# CONFLICT...
git status
# Mở file, tìm <<<<<<< ======= >>>>>>>
# Sửa thủ công, chọn code đúng
git add <resolved-file>
git commit
# hoặc
git merge --abort

Câu 11: Mục đích của .gitignore

Loại trừ file không cần track khỏi Git:

node_modules/
.env
*.log
dist/
build/
.DS_Store
*.pem
__pycache__/

Câu 12: Có nên lưu password/API key/DB credentials trong Git?

KHÔNG. Lý do:
- Git history vĩnh viễn, xóa file không xóa khỏi history
- Repo có thể bị leak
- Không thể rotate dễ dàng
- Vi phạm compliance (PCI-DSS, GDPR)
- Giải pháp: .env (gitignored), secret manager, GitHub Secrets, environment variables

============================================================
PHẦN C — Docker
============================================================

Dockerfile:

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 3000
ENV NODE_ENV=production PORT=3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 CMD node -e "require('http').get('http://127.0.0.1:3000/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) }).on('error', () => process.exit(1))"
CMD ["node", "server.js"]

docker-compose.yml: xem file gốc trong repo

Các lệnh Docker:

docker build -t hrm-app:latest ./application
docker run -d --name hrm-app -p 3000:3000 -e NODE_ENV=production hrm-app:latest
docker compose up -d --build
docker compose ps
docker compose logs -f app
docker compose stop
docker compose restart app
docker compose down
docker compose down -v

============================================================
PHẦN D — CI/CD
============================================================

Câu 13: CI là gì?

Continuous Integration — Tích hợp liên tục. Tự động build, test, kiểm tra code mỗi khi có commit/push. Phát hiện lỗi sớm.

Câu 14: CD là gì?

Continuous Delivery: Code luôn sẵn sàng deploy, deploy thủ công.
Continuous Deployment: Tự động deploy lên production sau khi pass pipeline.

Câu 15: Tại sao test tự động trước deploy?

- Phát hiện bug sớm, chi phí sửa thấp
- Đảm bảo chất lượng nhất quán
- Tăng tốc release, giảm rủi ro
- Tự tin refactor
- Ngăn bug lên production

Câu 16: Khi pipeline fail?

- Dừng pipeline, không deploy
- Thông báo cho người commit
- Block merge PR
- Developer fix và push lại
- Log chi tiết để debug

Câu 17: Xử lý secrets?

- GitHub Secrets
- HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
- Không hardcode, không commit
- Rotate định kỳ
- Least privilege
- Mã hóa at-rest và in-transit

Bonus: Pipeline mở rộng

Build → Test → Docker Image → Deploy Staging → Manual Approval → Production

Sử dụng GitHub Environments với required reviewers cho bước production.

============================================================
PHẦN E — Nginx / Reverse Proxy
============================================================

Câu 18: Mục đích của server

Định nghĩa một virtual host — lắng nghe trên port/IP cụ thể, xử lý request cho một domain. Có thể có nhiều server block cho nhiều domain.

Câu 19: Mục đích của location

Match URL path và định nghĩa cách xử lý (proxy, serve static, redirect, deny). Ví dụ location /api xử lý các request bắt đầu bằng /api.

Câu 20: Mục đích của proxy_pass

Chuyển tiếp request từ Nginx đến backend server (app Node.js ở localhost:3000). Nginx đóng vai reverse proxy.

Câu 21: Cấu hình HTTPS

ssl_certificate     /etc/nginx/ssl/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/privkey.pem;
ssl_protocols       TLSv1.2 TLSv1.3;

Dùng Let's Encrypt (certbot) để lấy cert miễn phí:
certbot --nginx -d hrmlabs.example.com

Câu 22: Redirect HTTP → HTTPS

server {
    listen 80;
    server_name localhost hrmlabs.example.com;
    return 301 https://$host$request_uri;
}

Câu 23: Security headers cần thiết

- Strict-Transport-Security (HSTS)
- X-Frame-Options
- X-Content-Type-Options: nosniff
- Referrer-Policy
- Content-Security-Policy
- X-XSS-Protection
- Permissions-Policy

============================================================
PHẦN F — Troubleshooting
============================================================

Câu 24: Điều gì đang xảy ra?

Container abc123 exit với code 1. Log báo không kết nối được MySQL (mysql:3306 connection refused). Nguyên nhân có thể:
- Container MySQL đã chết
- MySQL chưa ready khi app khởi động (race condition)
- Sai hostname/port trong env
- Network Docker bị lệch
- Firewall chặn
- MySQL hết connection slots hoặc bị crash

Server vẫn chạy, tài nguyên bình thường → không phải vấn đề tài nguyên.

Câu 25: Kiểm tra gì đầu tiên?

1. Trạng thái tất cả container: docker ps -a
2. Log container MySQL
3. Network giữa app và mysql
4. Env variables của app container
5. DNS resolution trong container

Câu 26: Các lệnh sử dụng

docker ps -a
docker inspect abc123 | grep -A5 NetworkSettings
docker logs abc123 --tail 100
docker logs <mysql-container> --tail 100
docker network ls
docker network inspect <network-name>
docker exec -it abc123 sh
  ping mysql
  nc -zv mysql 3306
  getent hosts mysql
  env | grep DB_
docker inspect <mysql-container> | grep -i health
docker stats --no-stream

Câu 27: Xác định MySQL có đang chạy?

docker ps | grep mysql
docker inspect <mysql-container> --format '{{.State.Status}}'
docker inspect <mysql-container> --format '{{.State.Health.Status}}'
docker logs <mysql-container> | tail -50
docker exec -it <mysql-container> mysqladmin ping -h localhost

Câu 28: Kiểm tra app có reach được MySQL?

docker exec -it abc123 sh -c "nc -zv mysql 3306"
docker exec -it abc123 sh -c "getent hosts mysql"
nc -zv localhost 3306
mysql -h 127.0.0.1 -P 3306 -u $DB_USER -p

Câu 29: Trước khi restart, làm gì?

- Đọc log đầy đủ của app và mysql
- Kiểm tra docker inspect để xem exit code, OOMKilled, restart count
- Kiểm tra volume MySQL còn nguyên không
- Backup log (không để mất evidence)
- Kiểm tra disk/memory host
- Xác nhận nguyên nhân trước khi restart

Câu 30: Phòng ngừa tái diễn

- Healthcheck cho MySQL + depends_on: condition: service_healthy
- Restart policy (unless-stopped, on-failure)
- Connection retry logic trong app (exponential backoff)
- Monitoring & alerting: Prometheus + Grafana
- Centralized logging: ELK / Loki
- Resource limits cho container
- Backup định kỳ MySQL
- CI/CD test integration với MySQL thật
- Runbook cho sự cố tương tự

============================================================
PHẦN G — Monitoring
============================================================

Câu 31: Điều gì trigger alert?

- CPU > 80% trong 5 phút
- RAM > 85%
- Disk > 85%
- Load average > số CPU × 2
- HTTP 5xx rate > 1%
- Response time P95 > 1s
- Container down / restart nhiều lần
- DB connection pool exhausted
- DB unavailable
- SSL cert sắp hết hạn (< 30 ngày)
- Error log tăng đột biến

Câu 32: Ai nhận alert?

- Critical (P1): on-call DevOps/SRE qua PagerDuty/Opsgenie + điện thoại
- Warning (P2): team Slack/Teams channel, email
- Info: dashboard, không alert
- Business impact: notify cả Product/Manager

Câu 33: Monitoring vs Logging

Monitoring:
- Metrics số (CPU, RAM, RPS)
- Trả lời "hệ thống khỏe không?"
- Aggregated, time-series
- Prometheus, Grafana, CloudWatch
- Alert theo ngưỡng

Logging:
- Sự kiện text chi tiết
- Trả lời "chuyện gì đã xảy ra?"
- Raw events, có thể search
- ELK, Loki, Splunk
- Debug, forensics, audit

Câu 34: Quản lý 20 servers không cần SSH từng cái

- Centralized monitoring: Prometheus + Node Exporter, Zabbix, Datadog
- Configuration management: Ansible, Puppet, Chef
- Centralized logging: ELK/Loki + Filebeat/Promtail
- Dashboards: Grafana
- Alerting tự động
- Infrastructure as Code: Terraform
- SSH qua bastion host với Ansible playbook
- Auto-remediation scripts

============================================================
PHẦN H — Security
============================================================

Câu 35: Production DB passwords lưu ở đâu?

- Secret Manager: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
- Kubernetes Secrets (mã hóa at-rest)
- Docker Swarm Secrets
- Environment variables inject runtime từ secret store
- KHÔNG trong Git, không trong image

Câu 36: Sai gì khi để DB_PASSWORD trong public Git?

- Ai cũng đọc được, compromise ngay
- Git history vĩnh viễn, khó xóa hoàn toàn
- Bot scan GitHub 24/7, bị khai thác trong vài phút
- Vi phạm compliance (GDPR, PCI-DSS, SOC2)
- Phải rotate ngay nếu đã lỡ commit

Câu 37: SSH key authentication là gì?

Xác thực bằng cặp khóa public/private thay vì password:
- Private key giữ bí mật trên máy client
- Public key đặt trên server (~/.ssh/authorized_keys)
- An toàn hơn password
- Có thể dùng passphrase
- ssh-keygen -t ed25519 -C "email"

Câu 38: Tại sao tránh dùng root cho app?

- Root có toàn quyền, nếu bị compromise mất cả hệ thống
- Vi phạm principle of least privilege
- Dễ vô tình phá hệ thống
- Không audit được ai làm gì
- Giải pháp: user riêng, USER trong Dockerfile, sudo có kiểm soát

Câu 39: Mục đích của firewall?

Kiểm soát traffic vào/ra theo rule (port, IP, protocol):
- Chặn port không cần thiết
- Chỉ cho phép IP tin cậy
- Ngăn tấn công mạng
- Công cụ: ufw, iptables, firewalld, Security Groups

Câu 40: Port 80, 443, 22

Port 22: SSH — quản trị server, nên giới hạn IP, đổi port, dùng key
Port 80: HTTP — web không mã hóa, nên redirect sang 443
Port 443: HTTPS — web mã hóa SSL/TLS, port chính cho production