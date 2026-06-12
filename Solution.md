# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. **Hardcoded API Key & Database URL**: Gán trực tiếp `OPENAI_API_KEY` và `DATABASE_URL` trong mã nguồn. Khi đưa lên kho lưu trữ code (GitHub/GitLab), các thông tin nhạy cảm này sẽ bị lộ.
2. **Cấu hình trực tiếp trong code (No Config Management)**: Biến `DEBUG` và `MAX_TOKENS` được định nghĩa cứng, không thể thay đổi linh hoạt giữa các môi trường (dev, staging, production) mà không phải sửa code.
3. **Thiếu Health Check Endpoint**: Ứng dụng không cung cấp bất kỳ API nào để giám sát trạng thái hoạt động (Liveness/Readiness). Nếu tiến trình bị treo hoặc lỗi kết nối database, các hệ thống Cloud/Kubernetes không thể tự động phát hiện để restart container.
4. **Sử dụng print() thay vì Structured Logging**: Dùng `print()` để log thông tin dạng plain text thô sơ, không lưu thời gian rõ ràng và nguy hiểm nhất là in cả API Key bí mật ra console log (`print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`).
5. **Gán cứng Host và Port (Hardcoded binding)**: Thiết lập `host="localhost"` và `port=8000`. Điều này làm cho ứng dụng chỉ chấp nhận kết nối nội bộ từ máy đó, không thể truy cập từ ngoài container hoặc internet. Môi trường cloud yêu cầu lắng nghe trên `0.0.0.0` và cổng `PORT` được cung cấp động qua biến môi trường.
6. **Không xử lý Graceful Shutdown**: Khi có tín hiệu dừng tiến trình (SIGTERM), ứng dụng sẽ bị ngắt đột ngột, dẫn đến việc các request đang xử lý dở dang bị lỗi và các kết nối cơ sở dữ liệu không được giải phóng đúng cách.

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| **Config** | Hardcode trực tiếp trong code | Đọc từ Environment Variables (12-Factor App) | Bảo mật thông tin nhạy cảm, dễ dàng cấu hình và triển khai qua nhiều môi trường mà không cần sửa code. |
| **Health check** | Không có | Có `/health` (Liveness) và `/ready` (Readiness) | Giúp Cloud Platform giám sát trạng thái của container để tự động restart nếu app bị treo hoặc tạm ngừng route traffic khi app đang quá tải. |
| **Logging** | Dùng `print()` đơn giản | Structured JSON Logging | Dễ dàng quản lý, lọc và phân tích log tự động bằng các công cụ tập trung (Datadog, Loki, ELK stack). Không rò rỉ secret. |
| **Shutdown** | Tắt đột ngột (Abort) | Graceful Shutdown (SIGTERM handling) | Giúp hoàn thành nốt các request đang xử lý dở dang và đóng kết nối (Database, Redis) an toàn trước khi tắt hoàn toàn. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base image**: `python:3.11`. Đây là phiên bản phân phối Python đầy đủ chạy trên nền Debian Linux (~1GB), chứa đầy đủ các công cụ build.
2. **Working directory**: `/app`. Tất cả các câu lệnh tiếp theo (COPY, RUN, CMD) sẽ được thực thi tại thư mục này trong container.
3. **Tại sao COPY requirements.txt trước?**: Để tối ưu hóa cơ chế Docker layer caching. Nếu chúng ta chỉ thay đổi source code mà không thêm bớt thư viện mới, Docker sẽ tái sử dụng layer cache cài đặt dependencies (`RUN pip install...`) đã build từ trước, giúp thời gian build sau này nhanh hơn rất nhiều.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**:
   - `ENTRYPOINT` định nghĩa câu lệnh chính và cố định sẽ luôn được thực hiện khi container khởi chạy.
   - `CMD` định nghĩa các đối số (arguments) mặc định truyền vào cho `ENTRYPOINT`. Người dùng có thể dễ dàng ghi đè (override) nội dung của `CMD` khi chạy container bằng lệnh `docker run <image> <new_arguments>`.

### Exercise 2.3: Image size comparison
- **Develop (Single-stage)**: ~1.02 GB
- **Production (Multi-stage)**: ~145 MB
- **Difference**: ~85% (Kích thước giảm đi đáng kể)

*(Giải thích: Multi-stage build chia quá trình build làm 2 stage. Stage 1 sử dụng đầy đủ các công cụ compiler/build (gcc, pip cache, v.v.). Stage 2 chỉ copy thư viện đã được build xong từ Stage 1 sang một base image dạng cực gọn `python:3.11-slim`, giúp loại bỏ hoàn toàn các file rác không cần thiết khi chạy).*

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **URL**: https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/
- **Ý nghĩa của các biến môi trường**:
  - `PORT`: Thiết lập cổng lắng nghe cho API Gateway. Các Cloud Provider sẽ inject biến này tự động để định tuyến traffic.
  - `AGENT_API_KEY`: Khóa bí mật dùng để xác thực client khi gọi API endpoint `/ask`.
  - `REDIS_URL`: Địa chỉ kết nối đến Redis dùng để quản lý state, lưu session history, rate limit, cost tracking tập trung.

---

## Part 4: API Security

### Exercise 4.1-4.3: Test results

#### Lệnh test không có X-API-Key:
```bash
curl -X POST https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
**Output mong muốn**: `401 Unauthorized` kèm thông báo:
```json
{"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
```

#### Lệnh test với API Key đúng:
```bash
curl -X POST https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ask -H "X-API-Key: dev-key-change-me" -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
**Output mong muốn**: `200 OK`
```json
{
  "question": "Hello",
  "answer": "Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T08:56:00.123456+00:00"
}
```

#### Lệnh test Rate Limiting (gọi liên tục vượt quá 20 requests/minute):
```bash
for i in {1..25}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ask -H "X-API-Key: dev-key-change-me" -H "Content-Type: application/json" -d '{"question":"Hello"}'; done
```
**Output**: Trả về một chuỗi mã `200` và cuối cùng xuất hiện mã lỗi `429` (Too Many Requests).
```json
{"detail": "Rate limit exceeded: 20 req/min"}
```

### Exercise 4.4: Cost guard implementation
- **Giải pháp tiếp cận (Approach)**:
  - Tích hợp một cấu trúc quản lý ngân sách hàng ngày (Daily budget) trong cấu hình (`daily_budget_usd`).
  - Hệ thống tính toán chi phí thực tế dựa trên số lượng token của Input (câu hỏi) và Output (câu trả lời) sử dụng các hệ số chi phí mô phỏng (ví dụ: GPT-4o-mini là `$0.00015 / 1K Input Tokens` và `$0.0006 / 1K Output Tokens`).
  - Trước mỗi request, Cost Guard thực hiện kiểm tra xem tổng chi phí tích lũy trong ngày đã vượt mức ngân sách cho phép chưa (`check_and_record_cost()`). Nếu vượt quá, lập tức chặn request mới và trả về mã lỗi `503 Service Unavailable` hoặc `402 Payment Required`, bảo vệ chủ tài khoản khỏi việc hóa đơn LLM tăng vọt ngoài tầm kiểm soát.

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes

#### 1. Liveness & Readiness Checks
- **Liveness Probe (`/health`)**: Trả về `200 OK` để thông báo tiến trình app vẫn đang hoạt động bình thường trong container.
- **Readiness Probe (`/ready`)**: Kiểm tra xem app đã nạp xong cấu hình, khởi tạo xong kết nối đến dependencies (như Redis) chưa. Nếu có bất kỳ kết nối nào bị gián đoạn, `/ready` trả về `503 Service Unavailable`, báo cho Load Balancer tạm thời dừng định tuyến traffic vào container lỗi này.

#### 2. Graceful Shutdown
- Khi nhận được tín hiệu dừng tiến trình `SIGTERM` từ Cloud Orchestrator, biến trạng thái `_is_ready` được đặt thành `False` để lập tức ngắt nhận thêm traffic mới.
- Ứng dụng sẽ chờ (với một thời gian timeout xác định, thường là 30 giây) để hoàn thành tất cả các request đang xử lý dở dang (in-flight requests) thông qua middleware đếm request.
- Sau khi số lượng in-flight requests giảm về 0 hoặc hết thời gian timeout, ứng dụng đóng an toàn các kết nối và dừng tiến trình sạch sẽ.

#### 3. Stateless Design (Redis)
- **Vấn đề**: Lưu trữ conversation history, rate limit và cost tracking trong memory (RAM của server) sẽ bị mất khi container restart, đồng thời không thể đồng bộ hóa dữ liệu khi scale ra nhiều replica chạy song song sau Load Balancer.
- **Giải pháp**: Đưa toàn bộ các dữ liệu state này lưu trữ tại một instance Redis tập trung bên ngoài. Nhờ đó, bất kỳ instance ứng dụng (replica) nào cũng có thể đọc/ghi trạng thái giống nhau, đảm bảo hệ thống có khả năng mở rộng không giới hạn (horizontal scaling).

