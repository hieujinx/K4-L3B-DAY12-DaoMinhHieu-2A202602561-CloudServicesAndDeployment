# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dao Minh Hieu — Mã học viên: 2A202602561

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu chạy service mà quên `AGENT_API_KEY`, Pydantic Settings sẽ báo lỗi
`Field required` ngay lúc khởi động thay vì cho app chạy với một khóa mặc
định. Ví dụ, Render sẽ đánh dấu deploy thất bại ngay ở bước start và mình biết
phải cấu hình secret, thay vì tưởng service đang bảo vệ API bằng khóa
`changeme`. Điều này tránh việc đưa một secret công khai lên production và
cho phép phát hiện lỗi cấu hình trước khi nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log JSON sau khi gọi `/ask` có dạng:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-09-29T04:39:16+00:00","user_id":"cp5-test","tokens_in":3,"tokens_out":12,"cost_usd":0.0001}
```

Mỗi event nằm trên một dòng và có các trường cố định. Từ đó mình có thể (1)
lọc riêng các request của `user_id` hoặc đếm số event `ask_completed` trong
một khoảng thời gian, và (2) tính tổng `cost_usd` rồi tạo cảnh báo khi chi phí
vượt ngưỡng. Một câu `print("đã trả lời xong")` không có timestamp, user,
chi phí hay cấu trúc để hệ thống log truy vấn được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Image multi-stage `day12-agent:prod` mình đã build và đo được khoảng **280 MB**
với `python:3.11-slim-bookworm`. Bản Dockerfile một stage ban đầu không còn
tag image riêng trong máy để đo lại sau khi đã thay Dockerfile, nên mình không
ghi một con số ước đoán cho bản đó. Chênh lệch của bản một stage thường đến từ
các file tạm/cache và toàn bộ dependency/build artifacts cần trong lúc cài
package; multi-stage chỉ chép phần package cần chạy sang runtime và bỏ phần
build. Điều quan trọng kiểm chứng được là image cuối dưới giới hạn 500 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Dockerfile hiện tại chép `requirements.txt` và chạy `pip install` trước khi
chép `app/` và `utils/`. Vì vậy khi chỉ sửa một ký tự trong `app/main.py`,
layer cài dependency và layer base được dùng lại; Docker chỉ phải chạy lại
layer copy source và các layer sau đó. Nếu đặt `COPY . .` trước `RUN pip
install`, mọi thay đổi trong source sẽ làm layer `COPY` đổi, khiến Docker
không dùng lại layer `pip install` và phải cài lại dependency, build lâu hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu tiến trình Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công có thể
thoát khỏi giới hạn của ứng dụng và chạy lệnh trong container. Nếu container
đang là root, tiến trình đó có quyền đọc/sửa nhiều file và khai thác container
runtime với mức quyền cao hơn, làm tăng rủi ro ảnh hưởng tới host. Dockerfile
của mình tạo `appuser` rồi dùng `USER appuser`; lệnh này cắt chuỗi ở điểm
tiến trình ứng dụng không còn chạy với UID 0. Đây không thay thế việc vá lỗ
hổng hay sandbox của Docker, nhưng giảm đáng kể quyền nếu ứng dụng bị khai
thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với cách reset theo phút đồng hồ và hạn mức 10/phút, người dùng có thể gửi
**20 request trong khoảng 2 giây**: 10 request ngay trước giây 00, rồi thêm
10 request ngay sau giây 00. Hai nhóm cùng được tính ở hai phút khác nhau
dù thời gian thực giữa request đầu và cuối chỉ khoảng 2 giây. Sliding window
giữ đúng 60 giây tính từ từng request, nên không có khoảng biên này và nhóm
request thứ hai sẽ bị giới hạn nếu 10 request trước đó vẫn còn trong cửa sổ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ gọi, còn cost guard giới hạn tổng chi phí ước tính
theo user trong tháng UTC. Ví dụ một user mới chỉ gọi 1 request/phút nên rate
limit cho qua, nhưng nếu request đó có chi phí ước tính làm tổng tháng vượt
`MONTHLY_BUDGET_USD` thì cost guard trả 402 trước khi gọi LLM. Ngược lại,
user có ngân sách còn nhiều có thể gửi 11 request trong một phút: cost guard
có thể cho qua vì tổng tiền vẫn thấp, nhưng rate limiter chặn request vượt
quá 10/phút bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp `/health` và `/ready` và endpoint đó kiểm tra Redis, khi Redis mất kết
nối thì cả ba container sẽ lần lượt trả lỗi probe. Orchestrator/load balancer
có thể coi cả cụm là unhealthy, restart các process hoặc ngừng gửi traffic
đồng thời, dù Python process vẫn còn sống. Trong 30 giây đó việc restart hàng
loạt làm gián đoạn dịch vụ và không sửa được nguyên nhân Redis. Với cách tách
hiện tại, `/health` vẫn trả 200 để liveness biết process còn sống, còn
`/ready` trả 503 để rút instance khỏi traffic cho đến khi Redis hoạt động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi state nằm trong Redis, các replica cùng đọc một history theo user nên
`history_length` tăng nhất quán dù request được chuyển sang container khác.
Nếu mỗi container giữ một dict Python riêng, request vào container A chỉ thấy
history của A, còn request vào B có thể lại thấy `history_length` nhỏ hơn
(thường quay về 0 hoặc 2 cho lượt đầu ở B). Khi container restart, dict đó
cũng mất hoàn toàn. Redis List, `ltrim` và TTL 7 ngày giúp history dùng chung
giữa các instance và tồn tại qua restart tạm thời.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lần build Docker đầu tiên trên Docker Desktop, dùng tag `python:3.11-slim`,
container gặp lỗi `/bin/sh: exec format error` khi chạy các lệnh build. Mình
kiểm tra bằng cách chạy image thử và xác nhận Docker engine vẫn hoạt động qua
`hello-world`/Alpine, nên lỗi không phải do app hay Redis. Sau đó mình đổi
base image sang tag Debian rõ ràng `python:3.11-slim-bookworm`, build lại
thành công và đo image khoảng 280 MB. Compose sau đó chạy được agent và Redis;
Render cũng deploy thành công với `/health` 200, `/ready` 200 và `/ask` yêu
cầu API key đúng 401/200.
