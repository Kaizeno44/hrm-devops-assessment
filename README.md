# HRM Labs DevOps Assessment

Bài assessment đầu vào vị trí DevOps Engineer tại HRM Labs.

Repo này chứa:
- Ứng dụng web demo (Node.js + Express)
- Cấu hình Docker & Docker Compose
- Cấu hình Nginx reverse proxy
- CI pipeline với GitHub Actions
- Câu trả lời lý thuyết cho 40 câu hỏi (xem ANSWERS.md)

## Yêu cầu hệ thống

- Docker >= 24.x
- Docker Compose >= 2.x
- Node.js >= 20.x (cho dev local)
- Git >= 2.x

## Cài đặt & chạy

### 1. Clone repository

git clone https://github.com/<Kaizeno44>/hrm-devops-assessment.git
cd hrm-devops-assessment

### 2. Cấu hình biến môi trường

cp .env.example .env

Mở file .env và chỉnh các giá trị (DB_USER, DB_PASSWORD, ...).

Lưu ý: KHÔNG commit file .env — file này đã được thêm vào .gitignore.

### 3. Chạy toàn bộ stack (app + MySQL + Nginx)

docker compose up -d --build

### 4. Kiểm tra

curl http://localhost:3000/health
curl http://localhost/health
curl http://localhost:3000/

Kết quả mong đợi:

{"status":"healthy"}

## Các lệnh thường dùng

- Build image: docker compose build
- Start stack: docker compose up -d
- Stop stack: docker compose stop
- Restart app: docker compose restart app
- Xem log app: docker compose logs -f app
- Xem log mysql: docker compose logs -f mysql
- Xem log nginx: docker compose logs -f nginx
- Vào shell container: docker compose exec app sh
- Xóa hết (kể cả volume): docker compose down -v
- Xem trạng thái: docker compose ps

## Troubleshooting

### 1. App không kết nối được MySQL

Triệu chứng: Container app exit với log "Error: Unable to connect to database".

Kiểm tra:

docker compose ps -a
docker compose logs mysql
docker compose exec app getent hosts mysql
docker compose exec app nc -zv mysql 3306
docker compose exec app env | grep DB_

Nguyên nhân thường gặp:
- MySQL chưa ready → chờ healthcheck pass
- Sai DB_HOST/DB_PORT trong .env
- App và MySQL khác network

### 2. Container exit ngay sau khi start

docker compose ps -a
docker inspect <container_id> --format '{{.State.ExitCode}}'
docker compose logs <service> --tail 100

### 3. Port đã bị chiếm (3000 hoặc 80)

sudo lsof -i :3000
sudo lsof -i :80

Đổi port mapping trong docker-compose.yml nếu cần.

### 4. Disk đầy

df -h
docker system df
docker system prune -a --volumes

### 5. Permission denied khi mount volume

docker compose exec app id

Đảm bảo quyền trên host khớp với UID trong container.

## CI Pipeline

GitHub Actions tự động chạy khi:
- Push lên nhánh main
- Mở Pull Request vào main

Các bước trong pipeline:
1. Checkout source code
2. Setup Node.js 20
3. npm ci — cài dependencies
4. npm test — chạy test
5. Build Docker image
6. Smoke test container (gọi /health)
7. Báo cáo kết quả

Xem chi tiết tại .github/workflows/ci.yml

## Cấu trúc repo

hrm-devops-assessment/
├── README.md
├── ANSWERS.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── application/
│   ├── Dockerfile
│   ├── package.json
│   ├── server.js
│   └── .dockerignore
├── nginx/
│   └── nginx.conf
├── .github/
│   └── workflows/
│       └── ci.yml
└── docs/
    ├── part-a-linux.md
    ├── part-b-git.md
    ├── part-c-docker.md
    ├── part-d-cicd.md
    ├── part-e-nginx.md
    ├── part-f-troubleshooting.md
    ├── part-g-monitoring.md
    └── part-h-security.md

## Bảo mật

- Không commit .env hoặc bất kỳ secret nào
- Dùng .env.example làm template
- Container chạy với user non-root
- HTTPS + security headers qua Nginx
- Secrets trong CI dùng GitHub Secrets

## Tài liệu tham khảo

- Docker Docs: https://docs.docker.com/
- Nginx Docs: https://nginx.org/en/docs/
- GitHub Actions: https://docs.github.com/en/actions
- Node.js Best Practices: https://github.com/goldbergyoni/nodebestpractices

## Tác giả

[Đoàn Trần Hải Nguyên] — Ứng viên DevOps Engineer
Email: [nguyensbo@gmail.com]