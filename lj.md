# HƯỚNG DẪN CHI TIẾT TRIỂN KHAI, GIẢI THÍCH HỆ THỐNG VÀ CHỤP ẢNH MINH CHỨNG
## MÔN THI: DEVOPS FUNDAMENTALS – ĐỀ 004: QUẢN LÝ KHO HÀNG

---

## MỤC LỤC
1. [Thông tin sinh viên & Thông số hệ thống](#i-thông-tin-sinh-viên--thông-số-hệ-thống)
2. [Giải thích kiến trúc hệ thống & Luồng hoạt động](#ii-giải-thích-kiến-trúc-hệ-thống--luồng-hoạt-động)
3. [Giải thích chi tiết từng file cấu hình & Mã nguồn](#iii-giải-thích-chi-tiết-từng-file-cấu-hình--mã-nguồn)
   - [1. File student.env](#1-giải-thích-file-studentenv)
   - [2. File Dockerfile (Multi-stage build)](#2-giải-thích-file-dockerfile-multi-stage-build)
   - [3. File docker-compose.yml (5 TODOs)](#3-giải-thích-file-docker-composeyml-5-todos)
   - [4. File .github/workflows/ci-cd.yml (6 TODOs)](#4-giải-thích-file-githubworkflowsci-cdyml-6-todos)
   - [5. File nginx/default.conf & database/init.sql](#5-giải-thích-file-nginxdefaultconf--databaseinitsql)
4. [Hướng dẫn kích hoạt Deploy lên VPS để Pipeline xanh toàn bộ](#iv-hướng-dẫn-kích-hoạt-deploy-lên-vps-để-pipeline-xanh-toàn-bộ)
5. [Hướng dẫn chi tiết cách chụp 4 ảnh minh chứng](#v-hướng-dẫn-chi-tiết-cách-chụp-4-ảnh-minh-chứng-phần-5)
6. [Quy trình đưa ảnh vào dự án và nộp bài lên hệ thống RAIA](#vi-quy-trình-đưa-ảnh-vào-dự-án-và-nộp-bài-lên-hệ-thống-raia)

---

### I. THÔNG TIN SINH VIÊN & THÔNG SỐ HỆ THỐNG

- **Họ và tên**: Phan Thuận Thành
- **Mã số sinh viên (MSSV)**: B24DTCN169
- **Lớp**: K24CNTT2
- **Email**: Phanthuanthanh270606@gmail.com
- **GitHub Username**: Phanthuanthanh
- **GitHub Repository (Public, nhánh main)**: [https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh](https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh)
- **Địa chỉ máy chủ chung (IP VPS)**: `20.189.122.108`
- **Tài khoản SSH VPS**: `thanhpt2706`
- **Cổng cá nhân truy cập (APP_PORT)**: `8169` (Ghép từ 8000 + 169 là 3 số cuối MSSV, đảm bảo >= 8080 và không trùng với sinh viên khác)
- **Tên project Docker Compose**: `phanthuanthanh-de004` (Định dạng `<tên-viết-tắt>-de004` viết thường)
- **Cơ sở dữ liệu**:
  - Loại CSDL: MySQL 8.0
  - Tên CSDL: `ptit_inventory_db` (Mặc định của Đề 004, không sửa)
  - Tài khoản ứng dụng: `app_user`
  - Mật khẩu ứng dụng: `270606`
  - Mật khẩu root: `270606`

---

### II. GIẢI THÍCH KIẾN TRÚC HỆ THỐNG & LUỒNG HOẠT ĐỘNG

```
[Trình duyệt Web / Client]
         │
         │  HTTP Request (Cổng 8169)
         ▼
 ┌─────────────────────────────────────────────────────────────┐
 │  [nginx] Service (nginx:1.27-alpine)                        │
 │  - Lắng nghe cổng 80 trong container                        │
 │  - Reverse Proxy toàn bộ request: GET /* -> app:8080        │
 │  - Chuyển hướng "/" thành 302 sang "/api/items"             │
 └──────────────────────────────┬──────────────────────────────┘
                                │ Mạng nội bộ: proxy-net
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │  [app] Service (Spring Boot 4, JRE 21)                      │
 │  - Chạy backend Java trên cổng 8080                         │
 │  - Xử lý API: GET /api/items                                │
 │  - Chạy bằng tài khoản thường (non-root: spring)            │
 └──────────────────────────────┬──────────────────────────────┘
                                │ Mạng nội bộ: db-net
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │  [mysql] Service (MySQL 8.0)                                │
 │  - Lắng nghe cổng nội bộ 3306                               │
 │  - Database: ptit_inventory_db                              │
 │  - Dữ liệu lưu bền vững trong Volume mysql_data             │
 │  - TUYỆT ĐỐI KHÔNG mở cổng 3306 ra ngoài Internet           │
 └─────────────────────────────────────────────────────────────┘
```

#### Luồng hoạt động của hệ thống:
1. **Tiếp nhận yêu cầu**: Người dùng mở trình duyệt gõ `http://20.189.122.108:8169/api/items` (hoặc `http://20.189.122.108:8169/`).
2. **Reverse Proxy (Nginx)**: Cổng `8169` trên máy chủ ánh xạ vào cổng `80` của container Nginx. Nginx nhận request, giữ vai trò làm cổng vào duy nhất (API Gateway), che giấu hoàn toàn các dịch vụ phía sau và chuyển tiếp request (proxy_pass) tới service `app` tại địa chỉ nội bộ `http://app:8080`.
3. **Xử lý ứng dụng (Spring Boot Backend)**: Ứng dụng Spring Boot tiếp nhận request `/api/items` thông qua `ItemController`, gọi `ItemRepository` để truy vấn danh sách toàn bộ các mặt hàng trong kho.
4. **Truy vấn CSDL (MySQL)**: Spring Boot kết nối tới MySQL thông qua chuỗi JDBC `jdbc:mysql://mysql:3306/ptit_inventory_db`. MySQL trả về dữ liệu 8 mặt hàng mẫu.
5. **Phản hồi dữ liệu**: Spring Boot đóng gói dữ liệu thành mảng JSON và trả ngược về Nginx, Nginx gửi về cho trình duyệt của người dùng.

#### Tính năng bảo mật qua phân vùng mạng (Network Isolation):
Hệ thống sử dụng mô hình kiến trúc 3 tầng (3-tier Architecture) với 2 mạng Docker độc lập:
- **`proxy-net`**: Kết nối giữa Nginx và App. Người dùng chỉ có thể giao tiếp với Nginx.
- **`db-net`**: Kết nối giữa App và MySQL.
- **Ý nghĩa bảo mật**: Nginx không thể truy cập trực tiếp vào MySQL. MySQL không mở cổng (port mapping) ra ngoài máy chủ vật lý, ngăn chặn 100% nguy cơ tấn công brute-force mật khẩu hoặc đánh cắp dữ liệu từ bên ngoài internet.

---

### III. GIẢI THÍCH CHI TIẾT TỪNG FILE CẤU HÌNH & MÃ NGUỒN

---

#### 1. Giải thích file `student.env`
File này là file cấu hình môi trường duy nhất chứa thông tin sinh viên và các tham số triển khai. Cả Docker Compose và GitHub Actions đều đọc file này:

```properties
# Thông tin sinh viên nhận diện bài làm
STUDENT_NAME="Phan Thuận Thành"
STUDENT_ID="B24DTCN169"
STUDENT_CLASS="K24CNTT2"
STUDENT_EMAIL="Phanthuanthanh270606@gmail.com"
GITHUB_USERNAME="Phanthuanthanh"

# Mã đề thi: Cố định là "004". Pipeline kiểm tra nếu giá trị này khác "004" sẽ lập tức hủy bỏ quá trình build.
EXAM_CODE="004"

# Cổng truy cập của ứng dụng trên VPS:
# Đặt là 8169 (>= 8080). Giúp phân biệt cổng của bạn với các bạn khác dùng chung VPS.
APP_PORT="8169"

# Tên dự án Docker Compose:
# Đặt theo chuẩn <tên-viết-tắt>-de004. Docker Compose sẽ dùng tiền tố này để đặt tên cho containers, volumes và networks (ví dụ: phanthuanthanh-de004-app-1), tránh xung đột tài nguyên với bạn khác.
COMPOSE_PROJECT_NAME="phanthuanthanh-de004"

# Tên cơ sở dữ liệu: Cố định là "ptit_inventory_db" theo đúng nghiệp vụ kho hàng của đề 004.
MYSQL_DATABASE="ptit_inventory_db"

# Tài khoản ứng dụng và mật khẩu MySQL:
# Cung cấp cho MySQL lúc khởi tạo và cho Spring Boot lúc kết nối.
MYSQL_USER="app_user"
MYSQL_PASSWORD="270606"
MYSQL_ROOT_PASSWORD="270606"
```

---

#### 2. Giải thích file `Dockerfile` (Multi-stage build)
File này áp dụng kỹ thuật **Multi-stage build** gồm 2 giai đoạn (`build` và `runtime`), giúp tách rời môi trường biên dịch nặng nề khỏi môi trường chạy ứng dụng thực tế.

```dockerfile
# syntax=docker/dockerfile:1

# =============================================================================
# GIAI ĐOẠN 1: BUILD (Biên dịch và đóng gói JAR)
# =============================================================================
# Sử dụng base image có sẵn Gradle 8.14 và JDK 21 trên nền Alpine Linux siêu nhẹ
FROM gradle:8.14-jdk21-alpine AS build
WORKDIR /workspace

# KỸ THUẬT DOCKER LAYER CACHING:
# Chỉ copy 2 file build.gradle và settings.gradle vào trước
COPY build.gradle settings.gradle ./

# Tải trước toàn bộ thư viện dependencies về lưu vào layer cache của Docker.
# Nếu mã nguồn Java trong src/ thay đổi nhưng dependencies không đổi, Docker sẽ tái sử dụng cache này, giúp thời gian build các lần sau chỉ mất vài giây thay vì phải tải lại toàn bộ.
RUN gradle dependencies --no-daemon

# Sau khi cache dependencies xong mới copy toàn bộ mã nguồn Java vào
COPY src ./src

# Biên dịch mã nguồn và đóng gói thành file executable JAR: build/libs/app.jar
# Cờ '-x test' giúp bỏ qua chạy unit test để tối ưu thời gian đóng gói
RUN gradle bootJar -x test --no-daemon


# =============================================================================
# GIAI ĐOẠN 2: RUNTIME (Môi trường thực thi tối giản)
# =============================================================================
# Chỉ sử dụng JRE 21 (Java Runtime Environment) của Eclipse Temurin nền Alpine
# Loại bỏ hoàn toàn JDK, Gradle và mã nguồn gốc -> Giảm kích thước image từ ~800MB xuống ~200MB và loại bỏ lỗ hổng bảo mật của trình biên dịch.
FROM eclipse-temurin:21-jre-alpine AS runtime

# Nhận đối số phiên bản ứng dụng từ Docker Compose hoặc CI/CD truyền vào (mặc định là dev)
ARG APP_VERSION=dev

# Gán biến môi trường phiên bản và cấu hình cấp phát tối đa 75% RAM container cho JVM
ENV APP_VERSION=${APP_VERSION} \
    JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"

WORKDIR /app

# NGUYÊN TẮC BẢO MẬT NON-ROOT:
# Tạo nhóm và người dùng thường 'spring' (không dùng quyền root)
RUN addgroup -S spring && adduser -S spring -G spring

# Chỉ sao chép duy nhất file JAR thành phẩm từ Giai đoạn 1 (AS build) sang
COPY --from=build /workspace/build/libs/app.jar /app/app.jar

# Chuyển quyền thực thi sang user 'spring' để ngăn ngừa nguy cơ container escape
USER spring

# Khai báo cổng 8080 mà ứng dụng Spring Boot lắng nghe bên trong container
EXPOSE 8080

# Chạy ứng dụng dưới định dạng exec array (JSON array):
# Giúp tiến trình Java nhận PID 1, có thể nhận trực tiếp tín hiệu SIGTERM khi dừng container (Graceful Shutdown)
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

#### 3. Giải thích file `docker-compose.yml` (5 TODOs)
File này dùng để điều phối và liên kết 3 dịch vụ: `mysql`, `app`, và `nginx`.

```yaml
name: ${COMPOSE_PROJECT_NAME:?Chua dien COMPOSE_PROJECT_NAME trong student.env}

services:
  # ---------------------------------------------------------------- MySQL
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    command: ["--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      TZ: Asia/Ho_Chi_Minh
    volumes:
      # Lưu trữ dữ liệu MySQL bền vững ra Docker Volume
      - mysql_data:/var/lib/mysql
      # Mount script SQL mẫu vào thư mục khởi tạo tự động của MySQL
      - ./database/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      # Định kỳ kiểm tra MySQL đã thực sự phản hồi lệnh ping chưa
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u$$MYSQL_USER -p$$MYSQL_PASSWORD --silent"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s
    networks:
      - db-net

  # ---------------------------------------------------------------- Spring Boot
  app:
    # TODO 1:
    # - context: . -> Chỉ định thư mục gốc làm ngữ cảnh build cho Dockerfile
    # - args: APP_VERSION: ${APP_VERSION:-local} -> Truyền phiên bản ứng dụng vào Dockerfile, mặc định là 'local'
    build:
      context: .
      args:
        APP_VERSION: ${APP_VERSION:-local}

    # TODO 2:
    # Tên image được gắn tag linh hoạt theo biến APP_IMAGE và APP_TAG.
    # Trong CI/CD, biến này sẽ trỏ tới GHCR (ghcr.io/.../app:<commit_sha>).
    # Khi chạy máy cá nhân, tự động fallback về 'devops-final-app:latest'.
    image: ${APP_IMAGE:-devops-final-app}:${APP_TAG:-latest}
    restart: unless-stopped
    environment:
      DB_HOST: mysql
      DB_PORT: "3306"
      # TODO 3:
      # Lấy thông tin kết nối CSDL trực tiếp từ các biến trong student.env, không fix cứng vào code
      DB_NAME: ${MYSQL_DATABASE}
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      TZ: Asia/Ho_Chi_Minh

    # CƠ CHẾ SẴN SÀNG:
    # Service 'app' chỉ được khởi động khi service 'mysql' đã qua bước healthcheck thành công (Healthy).
    # Giúp loại bỏ hoàn toàn lỗi Spring Boot bị crash do MySQL chưa khởi động xong.
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - db-net
      - proxy-net

  # ---------------------------------------------------------------- Nginx
  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    # TODO 4:
    # Ánh xạ cổng cá nhân APP_PORT (8169) trên máy chủ vật lý vào cổng 80 của Nginx
    ports:
      - "${APP_PORT}:80"

    # TODO 5:
    # Mount file cấu hình nginx/default.conf từ máy chủ vào container ở chế độ chỉ đọc (read-only: ro)
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - proxy-net

networks:
  proxy-net:
  db-net:

volumes:
  mysql_data:
```

---

#### 4. Giải thích file `.github/workflows/ci-cd.yml` (6 TODOs)
File này định nghĩa quy trình tự động hóa tích hợp liên tục và triển khai liên tục (CI/CD) thông qua GitHub Actions:

- **Sự kiện kích hoạt (Triggers)**: Tự động chạy mỗi khi có commit được đẩy (`push`) lên nhánh `main`, hoặc chạy thủ công bằng nút bấm (`workflow_dispatch`).

##### Giải thích Job 1: `build` (Kiểm thử, đóng gói và xuất bản Image)
1. **Checkout mã nguồn**: Tải code từ commit mới nhất về máy ảo runner.
2. **Kiểm tra thông tin trong `student.env`**: Chạy script bash kiểm tra tất cả các biến bắt buộc, đảm bảo không có biến nào rỗng, mã đề phải đúng `004`, CSDL đúng `ptit_inventory_db`, và cổng cá nhân phải từ `8080` trở lên.
3. **Đặt tên image**: Tạo biến `image=ghcr.io/<tài-khoản-github>/devops-final-de004-phanthuanthanh/app`.
4. **TODO 1 - Build & chạy thử bằng Docker Compose**:
   ```yaml
   run: docker compose --env-file student.env up -d --build --wait --wait-timeout 300
   ```
   *Ý nghĩa*: Khởi động toàn bộ 3 container ngay trên máy ảo GitHub Actions để chạy thử nghiệm tích hợp. Cờ `--wait --wait-timeout 300` yêu cầu Docker Compose chờ cho đến khi tất cả các service đạt trạng thái healthy sẵn sàng phục vụ.
5. **Smoke test**:
   ```bash
   curl -fsS --retry 12 --retry-delay 5 --retry-all-errors "$BASE/api/items" | grep -q '"id"'
   ```
   *Ý nghĩa*: Gửi request HTTP thật tới endpoint `/api/items`. Nếu API trả về JSON chứa trường `"id"`, chứng minh backend và database hoạt động hoàn hảo. Nếu lỗi, pipeline dừng ngay lập tức.
6. **TODO 2 - Đăng nhập GitHub Container Registry (GHCR)**:
   ```yaml
   uses: docker/login-action@v3
   with:
     registry: ${{ env.REGISTRY }}
     username: ${{ github.actor }}
     password: ${{ secrets.GITHUB_TOKEN }}
   ```
   *Ý nghĩa*: Đăng nhập vào GHCR bằng chính tài khoản người thực hiện push và token tự sinh của GitHub.
7. **TODO 3 - Push image lên GHCR**:
   ```bash
   docker tag "$IMAGE:${{ github.sha }}" "$IMAGE:latest"
   docker push "$IMAGE:${{ github.sha }}"
   docker push "$IMAGE:latest"
   ```
   *Ý nghĩa*: Gắn thêm tag `latest` cho image vừa build, sau đó đẩy cả 2 tag (mã commit SHA và `latest`) lên lưu trữ an toàn trên GitHub Container Registry.
8. **Dọn dẹp**: Chạy lệnh `docker compose down -v` để giải phóng tài nguyên máy ảo.

##### Giải thích Job 2: `deploy` (Triển khai tự động lên VPS)
1. **TODO 4 - Phụ thuộc công việc**:
   ```yaml
   needs: build
   ```
   *Ý nghĩa*: Đảm bảo quy tắc an toàn DevOps: Chỉ khi job `build` và kiểm thử thành công 100% thì job `deploy` mới được phép kích hoạt.
2. **TODO 5 - Kết nối SSH qua Secrets**:
   ```yaml
   uses: appleboy/ssh-action@v1.2.0
   with:
     host: ${{ secrets.SERVER_HOST }}
     username: ${{ secrets.SERVER_USER }}
     key: ${{ secrets.SERVER_SSH_KEY }}
   ```
   *Ý nghĩa*: Kết nối an toàn vào VPS `20.189.122.108` bằng private key SSH lưu trong GitHub Secrets mà không làm lộ mật khẩu hay khóa riêng tư trong mã nguồn.
3. **Đồng bộ mã nguồn trên VPS**: Kéo mã nguồn mới nhất từ nhánh `main` về máy chủ.
4. **TODO 6 - Kéo image và tái khởi động ứng dụng trên VPS**:
   ```bash
   docker compose --env-file student.env pull app
   docker compose --env-file student.env up -d --no-build
   ```
   *Ý nghĩa*: Trên VPS không thực hiện build lại mã nguồn (để tiết kiệm CPU/RAM cho VPS chung), mà chỉ tải trực tiếp image đã được build sẵn ở job 1 từ GHCR về (`pull app`), sau đó cập nhật và chạy lại container (`up -d --no-build`).

---

#### 5. Giải thích file `nginx/default.conf` & `database/init.sql`

- **`nginx/default.conf`**:
  - `upstream spring_app { server app:8080; }`: Khai báo backend đích thông qua DNS nội bộ của Docker (`app` là tên service trong docker-compose).
  - `location = / { return 302 /api/items; }`: Khi người dùng truy cập vào trang chủ `/`, Nginx tự động chuyển hướng sang danh sách hàng tồn kho `/api/items`.
  - `location / { proxy_pass http://spring_app; ... }`: Chuyển tiếp toàn bộ request và kèm các header cần thiết (`Host`, `X-Real-IP`, `X-Forwarded-Proto`).
- **`database/init.sql`**:
  - `USE ptit_inventory_db;`: Chỉ định database thao tác.
  - `CREATE TABLE IF NOT EXISTS items (...)`: Tạo bảng hàng tồn kho với các cột: `id`, mã SKU (`sku`), tên mặt hàng (`item_name`), kho lưu trữ (`warehouse`), số lượng (`quantity`), đơn vị tính (`unit`), và thời gian tạo (`created_at`).
  - `INSERT INTO items (...) VALUES (...)`: Nạp sẵn 8 bản ghi linh kiện máy tính và thiết bị mạng mẫu (Ổ cứng SSD, RAM, Cáp mạng, Switch, Bộ lưu điện, Màn hình, Bàn phím, Tủ rack).

---

### IV. HƯỚNG DẪN KÍCH HOẠT DEPLOY LÊN VPS ĐỂ PIPELINE XANH TOÀN BỘ

Hiện tại Job 1 (`Build & Test`) đã **xanh 100%**. Để Job 2 (`Deploy lên server`) kết nối được vào VPS và chuyển sang màu xanh, bạn chỉ cần thực hiện 3 bước sau:

#### BƯỚC 1: Thêm Public Key vào file `authorized_keys` trên VPS
Mở terminal trên máy tính (PowerShell hoặc CMD), chạy lệnh kết nối SSH vào VPS:
```bash
ssh thanhpt2706@20.189.122.108
```
*(Hệ thống sẽ hỏi mật khẩu VPS, bạn nhập: `270606` hoặc mật khẩu tài khoản VPS của bạn)*

Sau khi vào được giao diện dòng lệnh của VPS (`thanhpt2706@...`), copy nguyên vẹn đoạn lệnh sau và dán vào terminal để cấp quyền cho cặp khóa SSH mà dự án đã tạo:
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILVaqL1faCsIzPncT8feLK/PXqytAfK74sbQs8EnnZPU pc@MSI" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
Gõ `exit` để đăng xuất khỏi VPS.

#### BƯỚC 2: Thêm 3 GitHub Secrets vào Repository
Mở trình duyệt, truy cập thẳng vào trang cài đặt Secrets của Repository:  
👉 **[https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/settings/secrets/actions](https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/settings/secrets/actions)**

Bấm nút **New repository secret** (màu xanh lá cây) và lần lượt tạo 3 Secret sau:

1. **Secret 1:**
   - Name: `SERVER_HOST`
   - Secret: `20.189.122.108`

2. **Secret 2:**
   - Name: `SERVER_USER`
   - Secret: `thanhpt2706`

3. **Secret 3:**
   - Name: `SERVER_SSH_KEY`
   - Secret: Copy toàn bộ nội dung khối khóa riêng tư dưới đây (chú ý lấy đầy đủ từ dòng BEGIN đến END):
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACC1Wqi9X2grCMz53E/H3iyvz16srQHyu+LG0LPBJ52T1AAAAJDiCFka4ghZ
GgAAAAtzc2gtZWQyNTUxOQAAACC1Wqi9X2grCMz53E/H3iyvz16srQHyu+LG0LPBJ52T1A
AAAEB6GUKkyfS57R54jqIqwARHPHxaxXyjX3jgWGVbO7tUD7VaqL1faCsIzPncT8feLK/P
XqytAfK74sbQs8EnnZPUAAAABnBjQE1TSQECAwQFBgc=
-----END OPENSSH PRIVATE KEY-----
```

#### BƯỚC 3: Chạy lại Pipeline (Re-run jobs)
1. Vào tab **Actions** trên GitHub: [https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/actions](https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/actions)
2. Bấm vào lần chạy gần nhất `complete devops final app de004`.
3. Bấm nút **Re-run jobs** (ở góc trên bên phải) -> Chọn **Re-run all jobs**.
4. Chờ 1 - 2 phút, cả 2 job **Build & Test** và **Deploy lên server** sẽ hiện dấu tích xanh lá cây ✔️.

---

### V. HƯỚNG DẪN CHI TIẾT CÁCH CHỤP 4 ẢNH MINH CHỨNG (PHẦN 5)

Đề thi yêu cầu lưu 4 ảnh minh chứng vào thư mục `screenshots/` với đúng định dạng tên chuẩn sau:

---

#### 1. Ảnh `01-actions.png`
- **Ý nghĩa minh chứng**: Xác nhận pipeline CI/CD tự động hóa hoạt động hoàn chỉnh, vượt qua toàn bộ các bước kiểm tra cấu hình, build image, smoke test và deploy thành công lên server.
- **Cách thực hiện & Chụp ảnh**:
  - Truy cập đường dẫn: `https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/actions`
  - Bấm vào lần chạy mới nhất trên nhánh `main`.
  - Bạn sẽ thấy sơ đồ hiển thị 2 job liên kết với nhau:
    + `Build & Test` có dấu tích xanh ✔️
    + `Deploy lên server` có dấu tích xanh ✔️
  - Dùng công cụ **Snipping Tool** (hoặc phím tắt `Windows + Shift + S`) chụp toàn màn hình trình duyệt (thấy rõ URL trên thanh địa chỉ, tên commit và 2 dấu tích xanh).
  - Đặt tên file là: **`01-actions.png`**.

---

#### 2. Ảnh `02-ghcr.png`
- **Ý nghĩa minh chứng**: Xác nhận container image đã được đóng gói và xuất bản thành công lên GitHub Container Registry với 2 tag: mã commit SHA và tag `latest`.
- **Cách thực hiện & Chụp ảnh**:
  - Truy cập trực tiếp đường dẫn package của bạn:  
    👉 `https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh/pkgs/container/devops-final-de004-phanthuanthanh%2Fapp`  
    *(Hoặc vào trang cá nhân `https://github.com/Phanthuanthanh?tab=packages`, bấm chọn package `devops-final-de004-phanthuanthanh/app`)*
  - Xem danh sách các tag ở mục **Recent tags** hoặc tab **Tags/Versions**, bạn sẽ thấy rõ:
    + Tag `latest`
    + Tag `a852326...` (chuỗi commit SHA)
  - Chụp màn hình thấy rõ tên package, URL và 2 tag này.
  - Đặt tên file là: **`02-ghcr.png`**.

---

#### 3. Ảnh `03-compose-ps.png`
- **Ý nghĩa minh chứng**: Xác nhận các container thực sự đang vận hành ổn định trên server thực tế, đúng cổng và đạt trạng thái Healthy.
- **Cách thực hiện & Chụp ảnh**:
  - Mở PowerShell / CMD trên máy tính, SSH vào VPS:
    ```bash
    ssh thanhpt2706@20.189.122.108
    ```
  - Di chuyển vào thư mục dự án mà pipeline vừa deploy tự động:
    ```bash
    cd ~/devops-final-de004-phanthuanthanh
    docker compose --env-file student.env ps
    ```
  - Bảng hiển thị kết quả sẽ có 3 container:
    + `phanthuanthanh-de004-app-1`: trạng thái `Up`, cổng `8080/tcp`
    + `phanthuanthanh-de004-mysql-1`: trạng thái `Up (healthy)`, cổng `3306/tcp`
    + `phanthuanthanh-de004-nginx-1`: trạng thái `Up`, cổng `0.0.0.0:8169->80/tcp`
  - Chụp màn hình cửa sổ Terminal thấy rõ dòng lệnh `docker compose --env-file student.env ps` và bảng thông tin 3 container trên.
  - Đặt tên file là: **`03-compose-ps.png`**.

---

#### 4. Ảnh `04-website.png`
- **Ý nghĩa minh chứng**: Xác nhận người dùng đầu cuối có thể truy cập dịch vụ qua trình duyệt và dữ liệu kho hàng từ MySQL được Spring Boot hiển thị đầy đủ, chính xác dưới dạng JSON.
- **Cách thực hiện & Chụp ảnh**:
  - Mở trình duyệt Web (Chrome, Edge hoặc Firefox).
  - Nhập địa chỉ: 👉 **`http://20.189.122.108:8169/api/items`**  
    *(Hoặc nhập `http://20.189.122.108:8169/` do Nginx đã cấu hình tự redirect sang `/api/items`)*
  - Màn hình sẽ hiển thị dữ liệu JSON của 8 bản ghi:
    ```json
    [
      {"id":1,"itemName":"Ổ cứng SSD NVMe 512GB","quantity":120,"sku":"SSD-512-NV","unit":"Cái","warehouse":"Kho Hà Nội"},
      {"id":2,"itemName":"RAM DDR4 16GB","quantity":85,"sku":"RAM-16-D4","unit":"Thanh","warehouse":"Kho Hà Nội"},
      {"id":3,"itemName":"Cáp mạng Cat6 305m","quantity":40,"sku":"CAB-CAT6-305","unit":"Cuộn","warehouse":"Kho Đà Nẵng"},
      {"id":4,"itemName":"Switch 24 port Gigabit","quantity":18,"sku":"SW-24P-GB","unit":"Cái","warehouse":"Kho TP.HCM"},
      {"id":5,"itemName":"Bộ lưu điện 1KVA","quantity":25,"sku":"UPS-1KVA","unit":"Bộ","warehouse":"Kho Hà Nội"},
      {"id":6,"itemName":"Màn hình 24 inch Full HD","quantity":60,"sku":"MON-24-FHD","unit":"Cái","warehouse":"Kho TP.HCM"},
      {"id":7,"itemName":"Bàn phím USB văn phòng","quantity":200,"sku":"KB-USB-STD","unit":"Cái","warehouse":"Kho Đà Nẵng"},
      {"id":8,"itemName":"Tủ rack 42U","quantity":6,"sku":"RACK-42U","unit":"Tủ","warehouse":"Kho Hà Nội"}
    ]
    ```
  - Chụp toàn bộ màn hình trình duyệt thấy rõ thanh địa chỉ URL (`http://20.189.122.108:8169/api/items`) và nội dung JSON.
  - Đặt tên file là: **`04-website.png`**.

---

### VI. QUY TRÌNH ĐƯA ẢNH VÀO DỰ ÁN VÀ NỘP BÀI LÊN HỆ THỐNG RAIA

Sau khi bạn đã chụp và lưu đủ 4 ảnh:

1. **Lưu trữ ảnh vào đúng thư mục dự án**:
   - Copy 4 ảnh vào thư mục `screenshots/`:
     + `screenshots/01-actions.png`
     + `screenshots/02-ghcr.png`
     + `screenshots/03-compose-ps.png`
     + `screenshots/04-website.png`
   - Copy tiếp cả 4 ảnh vào thư mục `submission-content/` (theo đúng quy định đóng gói nộp bài của đề thi):
     + `submission-content/01-actions.png`
     + `submission-content/02-ghcr.png`
     + `submission-content/03-compose-ps.png`
     + `submission-content/04-website.png`

2. **Commit và đẩy ảnh lên GitHub**:
   Mở PowerShell tại thư mục dự án trên máy tính và chạy:
   ```bash
   git add screenshots/ submission-content/
   git commit -m "add proof screenshots"
   git push origin main
   ```

3. **Nộp bài**:
   - Copy link repository GitHub của bạn:  
     👉 **`https://github.com/Phanthuanthanh/devops-final-de004-phanthuanthanh`**
   - Dán link vào ô nộp bài trên hệ thống thi RAIA.
   - *Lưu ý quan trọng*: Giữ cho hệ thống và VPS tiếp tục hoạt động cho tới khi giảng viên chấm xong điểm thi.
