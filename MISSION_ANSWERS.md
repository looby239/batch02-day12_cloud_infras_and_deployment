# Day 12 Lab - Mission Answers

> **Student Name:** Nguyễn Thành Lộc  
> **Student ID:** 2A202600817  
> **Date:** 13/06/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found (Các lỗi thiết kế/Anti-pattern được tìm thấy)

Khi xem xét file `01-localhost-vs-production/develop/app.py`, 6 lỗi thiết kế (anti-pattern) sau đây đã được phát hiện:
1. **Lộ bí mật & thông tin xác thực trực tiếp (Hardcoded Secrets & Credentials):** API key OpenAI (`OPENAI_API_KEY`) và thông tin kết nối cơ sở dữ liệu (`DATABASE_URL`) được ghi trực tiếp trong mã nguồn. Điều này gây rủi ro bảo mật nghiêm trọng vì nếu mã nguồn được đẩy lên GitHub, các thông tin nhạy cảm này sẽ bị lộ ngay lập tức.
2. **Thiếu quản lý cấu hình (Lack of Configuration Management):** Các hằng số như `DEBUG` và `MAX_TOKENS` được viết cứng (hardcoded) thay vì tải động từ biến môi trường. Điều này làm cho ứng dụng trở nên cứng nhắc và khó cấu hình cho các môi trường chạy khác nhau (chẳng hạn như dev, staging, và production).
3. **Ghi log không đúng chuẩn (Inappropriate Logging Practices):** Ứng dụng phụ thuộc vào các lệnh in (`print()`) thay vì sử dụng hệ thống ghi log có cấu trúc. Hơn nữa, nó in cả thông tin xác thực nhạy cảm (ghi lại API key), dẫn đến việc rò rỉ bí mật ra đầu ra tiêu chuẩn (standard output).
4. **Không có Endpoint kiểm tra sức khỏe hệ thống (No Health / Readiness Check Endpoints):** Ứng dụng thiếu các endpoint `/health` (kiểm tra liveness) và `/ready` (kiểm tra readiness). Trong môi trường production, các bộ điều phối cloud không có cách nào tự động kiểm tra xem ứng dụng hoạt động bình thường hay đang bị treo, khiến việc tự động khởi động lại container bị lỗi không khả thi.
5. **Cấu hình mạng cứng nhắc và Port cố định (Fixed Network Binding and Hardcoded Port):** Máy chủ được ràng buộc chạy trên `localhost` (`host="localhost"`) và port cố định (`port=8000`), kèm theo bật chế độ `reload=True`. Việc chạy trên `localhost` ngăn ứng dụng nhận traffic từ bên ngoài container, và việc ghi cứng port ngăn cản các nền t trạng đám mây (Railway/Render) chèn port động vào.
6. **Không xử lý tắt ứng dụng an toàn (No Graceful Shutdown Handling):** Không có cơ chế bắt các tín hiệu tắt ứng dụng (như `SIGTERM`), dẫn đến việc máy chủ dừng đột ngột. Việc này có thể làm ngắt quãng các yêu cầu đang xử lý của client hoặc gây rò rỉ pool kết nối khi tắt hệ thống.

### Exercise 1.3: Comparison table (Bảng so sánh)

| Tính năng | Cơ bản (Develop) | Nâng cao (Production) | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Cấu hình (Config)** | Ghi trực tiếp (hardcoded) trong mã nguồn. | Tải từ biến môi trường (sử dụng Pydantic `BaseSettings` hoặc `os.getenv`). | Ngăn chặn rò rỉ thông tin bảo mật, tuân thủ nguyên tắc 12-Factor App, và cho phép thay đổi cấu hình theo từng môi trường dễ dàng mà không cần sửa code. |
| **Kiểm tra sức khỏe (Health check)** | Không có. | Cung cấp endpoint `/health` (liveness) và `/ready` (readiness). | Cho phép các nền tảng cloud giám sát trạng thái container, thực hiện kiểm tra sức khỏe tự động, khởi động lại container bị crash và tạm ngừng định tuyến traffic nếu các dịch vụ phụ thuộc gặp sự cố. |
| **Ghi log (Logging)** | Sử dụng lệnh `print()` thông thường (không có cấu trúc và đồng bộ). | Sử dụng log JSON có cấu trúc (định dạng key-value, không ghi log thông tin nhạy cảm). | Giúp các công cụ quản lý log tập trung (như Datadog, ELK, Grafana Loki) dễ dàng index, phân tích cú pháp, tìm kiếm và thiết lập cảnh báo. |
| **Tắt ứng dụng (Shutdown)** | Tắt đột ngột / dừng ngay lập tức. | Xử lý tắt ứng dụng an toàn (bắt tín hiệu `SIGTERM`, dừng xử lý qua vòng đời lifespan). | Giúp máy chủ dừng nhận yêu cầu mới, hoàn thành xử lý các yêu cầu đang chạy dang dở, đóng kết nối database/cache một cách sạch sẽ trước khi thoát hoàn toàn. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (Các câu hỏi về Dockerfile)

1. **Base image:** Base image được sử dụng là `python:3.11`. Đây là một image đầy đủ dựa trên Debian bao gồm thư viện Python tiêu chuẩn, các công cụ biên dịch (compilers) và thư viện phát triển (~1 GB kích thước).
2. **Working directory:** Thư mục làm việc là `/app`. Nó cách ly các file ứng dụng vào một thư mục chuyên dụng trong hệ thống tệp của container.
3. **Tại sao nên COPY requirements.txt trước?** Docker sử dụng cơ chế cache các layer khi build. Bằng cách copy `requirements.txt` và cài đặt dependencies trước khi copy toàn bộ mã nguồn ứng dụng, Docker có thể bỏ qua bước cài đặt thư viện tốn thời gian này trong các lần build sau nếu danh sách thư viện không có gì thay đổi.
4. **Sự khác biệt giữa CMD và ENTRYPOINT:**
   - `CMD` cung cấp lệnh mặc định và các tham số sẽ chạy khi container khởi động. Nếu có tham số bổ sung truyền vào lúc chạy lệnh `docker run`, chúng sẽ ghi đè hoàn toàn lên `CMD`.
   - `ENTRYPOINT` định cấu hình container chạy như một file thực thi. Các tham số truyền vào lúc runtime sẽ được thêm nối tiếp vào sau `ENTRYPOINT` chứ không ghi đè nó. Thông thường chúng được kết hợp: `ENTRYPOINT` đóng vai trò là lệnh thực thi chính còn `CMD` chứa danh sách tham số mặc định.

### Exercise 2.3: Image size comparison (So sánh kích thước Image)

- **Develop (Single-stage):** ~800 MB
- **Production (Multi-stage):** ~160 MB
- **Chênh lệch:** Giảm ~80%
- **Tại sao image production lại nhỏ hơn?**
  1. Bản build production sử dụng `python:3.11-slim` làm base image, phiên bản này có dung lượng cơ bản rất nhẹ.
  2. Nó áp dụng kỹ thuật build Docker đa tầng (multi-stage build): Tầng 1 (`AS builder`) tiến hành biên dịch các gói thư viện và cài đặt công cụ hỗ trợ (`gcc`, `libpq-dev`). Tầng 2 (`AS runtime`) chỉ sao chép các thư viện đã được biên dịch hoàn chỉnh (từ `/root/.local`) cùng mã nguồn ứng dụng, loại bỏ hoàn toàn các trình biên dịch không cần thiết và bộ nhớ đệm gói tải về khỏi image cuối cùng.

### Exercise 2.4: Docker Compose stack architecture & communication (Kiến trúc & Giao tiếp stack Docker Compose)

- **Các service được khởi động:**
  1. `agent`: Dịch vụ backend AI agent viết bằng FastAPI.
  2. `redis`: Cơ sở dữ liệu bộ nhớ đệm session và giới hạn tần suất yêu cầu (rate limiting).
  3. `qdrant`: Cơ sở dữ liệu vector dùng cho việc tìm kiếm/RAG.
  4. `nginx`: Máy chủ reverse proxy và cân bằng tải (Load Balancer), được expose ra máy host.
- **Sơ đồ kiến trúc (Architecture Diagram):**
  ```mermaid
  graph TD
      Client([Client]) -->|Port 80/443| Nginx[Nginx Load Balancer / Reverse Proxy]
      subgraph Internal Network ["Docker Internal Bridge Network (internal)"]
          Nginx -->|Round-Robin| Agent1[FastAPI Agent Replica 1: Port 8000]
          Nginx -->|Round-Robin| Agent2[FastAPI Agent Replica 2: Port 8000]
          Nginx -->|Round-Robin| Agent3[FastAPI Agent Replica 3: Port 8000]
          Agent1 & Agent2 & Agent3 -->|Port 6379| Redis[(Redis Cache / Rate Limiter)]
          Agent1 & Agent2 & Agent3 -->|Port 6333| Qdrant[(Qdrant Vector Database)]
      end
  ```
- **Luồng giao tiếp (Communication Flow):**
  - Tất cả dịch vụ được kết nối vào một mạng bridge dùng chung và nội bộ tên là `internal`.
  - Nginx expose port `80` ra máy host để tiếp nhận các yêu cầu từ client. Nó đóng vai trò là load balancer/reverse proxy để chuyển tiếp các cuộc gọi từ client tới các instance của `agent`.
  - Các `agent` giao tiếp với `redis` (qua tên máy chủ `redis` tại cổng 6379) và `qdrant` (qua tên máy chủ `qdrant` tại cổng 6333) trực tiếp thông qua mạng nội bộ Docker. Các kết nối bên ngoài không thể truy cập trực tiếp vào agent, Redis, hay Qdrant.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment (Triển khai lên Railway)

- **URL:** `https://distinguished-reverence-production-a4e9.up.railway.app`
- **Screenshot:** *Mặc dù không có thư mục hình ảnh trực tiếp trong repository này, triển khai đã được xác minh thành công qua kiểm tra endpoint hoạt động.*

### Exercise 3.2: Deploy Render & Comparison (So sánh render.yaml và railway.toml)

- **So sánh Render.yaml và Railway.toml:**
  - **`render.yaml` (Cơ sở hạ tầng dạng code - IaC):** File cấu hình dạng blueprint được Render sử dụng để khai báo toàn bộ các dịch vụ liên kết với nhau (như web server, database, Redis cache), ổ đĩa lưu trữ, khu vực máy chủ (regions), đồng bộ hóa biến môi trường và tự động co giãn (autoscaling) thành một cụm hạ tầng thống nhất.
  - **`railway.toml` (Cấu hình triển khai dịch vụ):** File cấu hình nhẹ ở cấp độ dịch vụ nhằm hướng dẫn Railway cách build (ví dụ dùng Nixpacks hoặc Dockerfile) và deploy một dịch vụ ứng dụng cụ thể, định nghĩa lệnh thực thi, kiểm tra sức khỏe tùy chỉnh (health check) và chính sách khởi động lại container.

### Exercise 3.3: (Optional/Bonus) GCP Cloud Run CI/CD (Triển khai nâng cao lên Google Cloud Run)

Khi xem xét cấu hình triển khai Cloud Run trong thư mục `03-cloud-deployment/production-cloud-run/`:
- **`cloudbuild.yaml` (Cấu hình CI/CD Pipeline):** Định nghĩa quy trình build tự động với 4 bước chính:
  1. `test`: Chạy bộ unit test của ứng dụng trong môi trường container `python:3.11-slim`.
  2. `build`: Build Docker image từ source code và đánh tag `ai-agent:$COMMIT_SHA` (có sử dụng cache từ bản build `latest` trước đó để tăng tốc).
  3. `push`: Đẩy image đã build lên Google Container Registry (GCR).
  4. `deploy`: Triển khai image lên Cloud Run tại region `asia-southeast1`, cấu hình các tài nguyên phần cứng (CPU, Memory), thiết lập tự động co giãn (`min-instances=1`, `max-instances=10`) và liên kết các thông tin nhạy cảm (như `OPENAI_API_KEY`) một cách an toàn từ Secret Manager.
- **`service.yaml` (Định nghĩa hạ tầng Cloud Run - IaC):** Khai báo các thông số của dịch vụ Knative:
  - Cấu hình auto-scaling (`minScale: 1` tránh cold start, `maxScale: 10`), giới hạn concurrency tối đa 80 request/instance để tối ưu hóa hiệu suất.
  - Giới hạn tài nguyên phần cứng (limits: CPU 1, RAM 512Mi).
  - Tải tự động các biến môi trường và tham chiếu key bí mật từ Google Secret Manager.
  - Định nghĩa cơ chế kiểm tra sức khỏe thông qua `livenessProbe` (`/health`) và `startupProbe` (`/ready`).

---

## Part 4: API Security

### Exercise 4.1: API Key authentication (Xác thực API Key)

- **API key được kiểm tra ở đâu?**
  - API Key được kiểm tra trong hàm dependency `verify_api_key` (trong file `app/auth.py` hoặc trực tiếp trong file code `develop/app.py`). Hàm này được định nghĩa như một FastAPI dependency sử dụng class `APIKeyHeader` để trích xuất API key từ header của request (cụ thể là header `X-API-Key`).
- **Điều gì xảy ra nếu sai key hoặc thiếu key?**
  - **Thiếu key:** Hệ thống quăng ra lỗi `HTTPException` với mã trạng thái `401 Unauthorized` và thông báo `"Missing API key..."` (hoặc `"Invalid or missing API key..."`).
  - **Sai key:** Hệ thống quăng ra lỗi `HTTPException` với mã trạng thái `403 Forbidden` (hoặc `401 Unauthorized` tùy phiên bản cấu hình) và thông báo `"Invalid API key."`.
- **Làm sao để xoay vòng (rotate) API key?**
  - API key được cấu hình thông qua biến môi trường `AGENT_API_KEY` (được đọc vào hệ thống qua Pydantic `Settings` ở file `config.py` hoặc `os.getenv`). Để xoay vòng key, chỉ cần cập nhật giá trị của biến môi trường `AGENT_API_KEY` trong file cấu hình `.env` hoặc trên dashboard của Cloud Provider (Railway/Render/etc.) và khởi động lại/triển khai lại container, không cần phải chỉnh sửa hay thay đổi bất kỳ dòng mã nguồn nào.

### Exercise 4.2: JWT authentication (Xác thực JWT - Nâng cao)

- **Đọc và hiểu JWT flow trong `auth.py`:**
  - Quy trình xác thực JWT (JSON Web Token) được thiết kế theo mô hình không trạng thái (stateless):
    1. **Đăng nhập và cấp Token (`POST /auth/token`):** Người dùng gửi username và password lên API. Hệ thống xác thực thông tin qua hàm `authenticate_user`. Nếu chính xác, hệ thống gọi `create_token` để mã hóa thông tin user (`sub` là username, `role` là vai trò của user, thời gian tạo `iat` và thời gian hết hạn `exp` - mặc định là 60 phút) bằng thuật toán `HS256` cùng một khóa bí mật `JWT_SECRET` được cấu hình từ biến môi trường.
    2. **Gửi Token trong các request tiếp theo:** Client nhận JWT và lưu trữ lại. Với mọi request gọi API cần bảo mật `/ask`, client gửi token này trong header dưới dạng `Authorization: Bearer <token>`.
    3. **Xác thực Token (`verify_token`):** Phía máy chủ, FastAPI sử dụng dependency `verify_token` để giải mã JWT bằng `JWT_SECRET`. Nếu chữ ký hợp lệ và token chưa hết hạn, thông tin user (username và role) được trích xuất để tiếp tục xử lý request. Nếu hết hạn, hệ thống trả về `401 Unauthorized` (Token expired); nếu chữ ký sai, hệ thống trả về `403 Forbidden` (Invalid token).

### Exercise 4.3: Rate limiting (Giới hạn tần suất yêu cầu)

- **Thuật toán được sử dụng:**
  - Thuật toán **Sliding Window Counter** (Cửa sổ trượt) được triển khai thông qua cấu trúc dữ liệu `deque` trong Python (hoặc lưu trữ trong Redis ở môi trường production).
- **Cách thức hoạt động và giới hạn (Limits):**
  - Mỗi khi user gửi một request, hệ thống sẽ thêm timestamp hiện tại vào `deque` tương ứng của user đó. Đồng thời, hệ thống duyệt và loại bỏ tất cả các timestamp cũ hơn cửa sổ thời gian (60 giây trước đó).
  - Hệ thống đếm số lượng timestamp còn lại trong `deque`. Nếu vượt quá giới hạn cấu hình, hệ thống sẽ quăng ra lỗi `HTTPException` với mã trạng thái `429 Too Many Requests`.
  - **Giới hạn cụ thể:** Trong file `rate_limiter.py`, hệ thống định nghĩa các hạn mức khác nhau tùy thuộc vào vai trò (role) của người dùng:
    - Người dùng thông thường (`rate_limiter_user`): Giới hạn **10 requests/phút**.
    - Người quản trị (`rate_limiter_admin`): Giới hạn **100 requests/phút**.
- **Làm sao để bypass hoặc tăng giới hạn cho admin?**
  - Trong hàm endpoint `/ask`, hệ thống trích xuất thông tin `role` từ JWT token đã được xác thực thành công. Dựa vào vai trò đó, ứng dụng sẽ quyết định sử dụng bộ rate limiter nào để kiểm tra:
    ```python
    limiter = rate_limiter_admin if role == "admin" else rate_limiter_user
    ```
    Nhờ cơ chế kiểm tra động này, tài khoản admin tự động được áp dụng hạn mức cao hơn (100 req/min) so với tài khoản thông thường, hoặc có thể tùy chỉnh thêm logic để bỏ qua (bypass) hoàn toàn bước kiểm tra rate limit đối với một số tài khoản admin đặc biệt.

### Exercise 4.1-4.3: Test results (Kết quả kiểm thử)

#### 1. API Request - Thiếu API Key
```json
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Invalid or missing API key. Include header: X-API-Key: <key>"
}
```

#### 2. API Request - Đã xác thực & Thành công
```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "question": "Hello",
  "answer": "Mock LLM Response to: Hello",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T04:55:00Z"
}
```

#### 3. API Request - Vượt quá giới hạn Rate Limiting (Người dùng tiêu chuẩn)
```json
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60

{
  "detail": "Rate limit exceeded: 20 req/min"
}
```

#### 4. API Request - Vượt quá hạn mức ngân sách chi phí
```json
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "detail": "Daily budget exhausted. Try tomorrow."
}
```

### Exercise 4.4: Cost guard implementation (Triển khai bảo vệ ngân sách/chi phí)

- **Phương pháp tiếp cận:**
  - Định cấu hình một thành phần giám sát chi phí (budget guard) để theo dõi số lượng token tiêu thụ trên mỗi người dùng (hoặc toàn cục) trong chu kỳ 24 giờ liên tục.
  - Chi phí sử dụng được tính theo thời gian thực dựa trên đơn giá mô hình (ví dụ: $0.00015 / 1K input tokens và $0.0006 / 1K output tokens).
  - Trước khi gọi LLM, hệ thống chạy hàm `check_budget` để kiểm tra xem tổng chi tiêu trong ngày của người gửi yêu cầu đã vượt quá mức quy định `daily_budget_usd` chưa. Nếu vượt quá, ứng dụng sẽ quăng ra lỗi `HTTPException` với mã trạng thái `503` (hoặc `402 Payment Required`), ngăn chặn phát sinh các khoản phí LLM không mong muốn.
  - Sau khi phản hồi từ LLM được tạo ra, hệ thống gọi `check_and_record_cost` để tính toán chính xác lượng token đầu ra thực tế, cộng dồn vào bộ đếm của client và lưu lại chi phí phiên làm việc. Trong môi trường production, các bộ đếm tích lũy này được lưu trữ tập trung tại cơ sở dữ liệu Redis để đảm bảo tính nhất quán và chia sẻ chung giữa các replica được cân bằng tải.

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes (Ghi chú triển khai thực tế)

- **Exercise 5.1 (Health & Readiness Probes):**
  - **Liveness probe (`/health`):** Kiểm tra xem tiến trình chạy ứng dụng bên trong container còn hoạt động hay không. Nếu phản hồi trả về mã lỗi hoặc hết thời gian chờ (do rò rỉ bộ nhớ hoặc deadlock), bộ điều phối container sẽ tự động khởi động lại container đó.
  - **Readiness probe (`/ready`):** Kiểm tra ứng dụng đã hoàn toàn sẵn sàng nhận lưu lượng truy cập bên ngoài hay chưa (kiểm tra kết nối tới Redis và các thành phần phụ thuộc). Nếu trả về trạng thái lỗi, bộ cân bằng tải sẽ ngừng chuyển tiếp lưu lượng của người dùng tới instance này.
- **Exercise 5.2 (Graceful Shutdown - Tắt ứng dụng an toàn):**
  - Hệ thống lắng nghe tín hiệu `SIGTERM` được gửi từ bộ điều phối container. Khi bắt được tín hiệu này, ứng dụng sẽ chuyển trạng thái `/ready` thành unhealthy (không sẵn sàng) để bộ cân bằng tải loại bỏ instance này khỏi danh sách nhận traffic, chờ các yêu cầu hiện tại xử lý xong xuôi (sử dụng bộ đếm yêu cầu nội bộ), đóng an toàn các kết nối Redis/DB và thoát ra một cách sạch sẽ.
- **Exercise 5.3 (Thiết kế Stateless - Không lưu trạng thái):**
  - Khi cấu hình cân bằng tải, các yêu cầu của một người dùng có thể được định tuyến tới các instance khác nhau. Nếu lịch sử hội thoại được lưu trong bộ nhớ cục bộ (in-memory) của từng máy chủ, trạng thái sẽ bị mất. Để đạt được thiết kế stateless, chúng tôi lưu toàn bộ metadata phiên người dùng và lịch sử trò chuyện trong một cơ sở dữ liệu cache dùng chung có tốc độ truy xuất nhanh như Redis.
- **Exercise 5.4 (Cân bằng tải với Nginx):**
  - Nginx hoạt động như một reverse proxy load balancer giúp phân phối lưu lượng truy cập đến theo thuật toán Round-Robin qua các replica đang hoạt động của dịch vụ agent, đảm bảo tính sẵn sàng cao và phân chia tải đều đặn.
- **Exercise 5.5 (Xác minh cơ chế Stateless):**
  - Đã được kiểm chứng thông qua file `test_stateless.py`. Các yêu cầu gửi liên tục với cùng `session_id` được xử lý bởi các backend instance khác nhau (thể hiện qua các mã nhận diện `served_by` khác nhau), tuy nhiên ngữ cảnh hội thoại vẫn được lưu giữ và khôi phục thành công nhờ lưu trữ tập trung trong Redis.
