# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder (in nghiêng, bắt đầu bằng "Câu trả lời
> của bạn") ở dưới mỗi câu hỏi bằng câu trả lời thật.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: mình deploy lên Railway/Render và quên set biến `AGENT_API_KEY`
> trong dashboard. Nếu `agent_api_key` có mặc định `"changeme"`, app vẫn khởi
> động bình thường, health check pass, mình tưởng deploy thành công — nhưng
> thực chất `/ask` đang được "bảo vệ" bằng một khóa ai cũng đoán được. Bất kỳ
> ai tìm ra URL rồi thử khóa `"changeme"` đều gọi được, tiêu ngân sách LLM của
> mình mà mình không hề hay biết, cho tới khi nhận hóa đơn cao bất thường vài
> ngày sau. Vì `agent_api_key` không có mặc định, `pydantic` ném
> `ValidationError` và app crash ngay lúc khởi động khi thiếu biến — lỗi hiện
> ra ngay trong log/dashboard lúc mình đang theo dõi quá trình deploy, nên
> mình sửa được ngay lập tức thay vì phát hiện ra qua hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ service khi gọi `/ask`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:58:10.685136+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và tính tổng theo trường.** Vì mỗi dòng là JSON có key cố định
>    (`user_id`, `cost_usd`...), công cụ như `jq` có thể trích đúng field và
>    cộng dồn để trả lời "user nào tiêu nhiều tiền nhất hôm nay" bằng một
>    lệnh. Với `print()` dạng câu tự do, phải viết regex đoán mò theo đúng câu
>    chữ, và vỡ ngay khi ai đó đổi cách viết câu log.
> 2. **Đặt cảnh báo tự động theo `level`.** Dashboard log (Datadog, CloudWatch,
>    Railway logs...) group các dòng theo `level="error"` trong 5 phút gần
>    nhất để tính tỷ lệ lỗi và tự bắn cảnh báo khi tỷ lệ tăng đột biến.
>    `print("đã trả lời xong")` không có field mức độ log chuẩn hóa nên không
>    nhóm/đếm được theo cách này.

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
| 1 stage (bản đầu) | 446 MB (content size) / 1.73 GB (disk usage, `docker images` bản Docker Desktop mới) |
| Multi-stage | 63.7 MB (content size) / 270 MB (disk usage) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật: `docker images agent:single` → 446MB; `docker images
> day12-agent:prod` → 63.7MB — multi-stage nhỏ hơn khoảng **7 lần**, và nằm
> gọn dưới ngưỡng 500MB của CP2 trong khi bản 1 stage gần chạm ngưỡng đó.
>
> Phần chênh lệch (~380MB) đến từ:
> 1. **Base image đầy đủ vs slim.** `python:3.11` (bản 1 stage dùng) mang theo
>    trình biên dịch, header phát triển, và nhiều thư viện hệ thống không cần
>    lúc chạy. `python:3.11-slim` (mình dùng cho cả 2 stage của bản
>    production) đã bỏ hết những thứ đó.
> 2. **Không tách builder/runtime.** Bản 1 stage `RUN pip install` ngay trên
>    image cuối, nên mọi thứ `pip` cần TRONG LÚC CÀI (cache tải về, có thể cả
>    công cụ biên dịch cho các gói có phần mở rộng C) đều ở lại trong image
>    cuối cùng. Bản multi-stage của mình cài ở stage `builder`, rồi chỉ
>    `COPY --from=builder /install /usr/local` đúng phần đã cài xong sang
>    stage `runtime` — bỏ lại toàn bộ rác build.
> 3. **`COPY . .` copy cả thư mục.** Bản 1 stage copy nguyên repo (kể cả
>    `tests/`, `.git` nếu không có `.dockerignore` tốt...) thay vì chỉ
>    `app/` và `utils/` như bản production của mình.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của mình tách 2 stage: `builder` chỉ `COPY requirements.txt` rồi
> `pip install --prefix=/install`, `runtime` mới `COPY --from=builder`,
> `useradd`, rồi cuối cùng mới `COPY app ./app` và `COPY utils ./utils`. Sửa
> một ký tự trong `app/main.py` rồi build lại: Docker so hash từng layer theo
> thứ tự — `requirements.txt` không đổi nên toàn bộ stage `builder` (kể cả
> `pip install`, bước tốn thời gian nhất) và các layer đầu của `runtime`
> (`COPY --from=builder`, `useradd`) đều lấy từ cache (`CACHED`). Chỉ từ layer
> `COPY app ./app` trở đi (`COPY utils`, `RUN chown`, `USER`, `CMD`) mới build
> lại, vì nội dung thư mục `app` đã đổi hash.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` (như bản Dockerfile gốc lúc
> đầu, chỉ 1 stage): sửa một ký tự bất kỳ trong code cũng làm hash của layer
> `COPY . .` đổi, kéo theo MỌI layer sau nó — kể cả `pip install` — mất cache
> và phải chạy lại từ đầu, dù không có thư viện nào thay đổi. Build đang từ
> vài giây (nhờ cache) tăng lên hàng chục giây tới cả phút vì phải tải và cài
> lại toàn bộ `requirements.txt` qua mạng mỗi lần sửa dù chỉ một dòng code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python của app có lỗ hổng (ví dụ một thư viện dính
> lỗi remote code execution, hoặc một endpoint không kiểm tra input kỹ) khiến
> kẻ tấn công chạy được lệnh shell bên trong container. (2) Nếu container
> đang chạy bằng root (UID 0), lệnh đó có toàn quyền BÊN TRONG container: đọc
> ghi mọi file, cài phần mềm, sửa cấu hình hệ thống trong container. (3) Nếu
> container có cấu hình không an toàn (mount thư mục host vào container,
> chạy `--privileged`, hoặc gặp đúng lỗ hổng escape của container
> runtime/kernel), quyền root bên trong container có thể leo thang thành
> quyền root trên chính máy host — toàn bộ server bị chiếm, không chỉ riêng
> container đó.
>
> Lệnh `USER appuser` trong Dockerfile của mình (`useradd --uid 10001
> appuser` rồi `USER appuser`) cắt đứt chuỗi này ngay ở bước (2): dù code có
> lỗ hổng và kẻ tấn công chạy được lệnh, lệnh đó chỉ chạy với quyền của một
> user thường (UID 10001) — không ghi được vào thư mục hệ thống, không cài
> được phần mềm ở nơi cần quyền root, và ngay cả khi có escape container thì
> thứ leo thang ra host cũng chỉ là quyền user thường, không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây liên tiếp**. Cách đạt được: gửi đúng 10
> request vào giây thứ 59 của phút hiện tại (bộ đếm của phút này còn nguyên
> hạn mức 10 → cả 10 đều được cho qua). Một giây sau, đồng hồ sang phút mới,
> bộ đếm reset về 0 — gửi tiếp 10 request vào giây 00–01 của phút kế tiếp
> (cũng đúng luật vì đang ở một "phút" khác, hạn mức lại đầy). Tổng cộng 20
> request lọt qua trong khoảng 2 giây thực tế, dù hạn mức danh nghĩa là
> 10/phút. Sliding window 60 giây (cách mình cài trong `rate_limiter.py`,
> dùng `zremrangebyscore(key, 0, now - 60)` rồi `zcard`) không có kẽ hở này vì
> nó luôn đếm đúng 60 giây gần nhất tính từ THỜI ĐIỂM request tới, không phụ
> thuộc ranh giới phút đồng hồ cố định nào cả.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau: rate limit giới hạn **số lượng** request trong một khoảng thời
> gian, không quan tâm mỗi request tốn bao nhiêu tiền. Cost guard giới hạn
> **tổng số tiền** đã chi trong tháng, không quan tâm số lượng request.
>
> - **Rate limit cho qua, cost guard chặn:** user gửi 5 request/phút (dưới
>   hạn mức `RATE_LIMIT_PER_MINUTE=10`), nhưng mỗi câu hỏi rất dài, tốn nhiều
>   token. Chỉ vài request như vậy đã vượt `MONTHLY_BUDGET_USD` → `guard.check()`
>   raise `402` dù chưa hề chạm ngưỡng rate limit.
> - **Cost guard cho qua, rate limit chặn:** user gửi 20 request/giây toàn câu
>   hỏi cực ngắn kiểu "hi", mỗi request tốn gần như không đáng kể (ngân sách
>   tháng còn dư rất nhiều) nhưng tần suất gọi vượt xa 10/phút →
>   `limiter.check()` raise `429` dù ngân sách gần như còn nguyên.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối. Cả 3 container gọi endpoint gộp đó
> đều thấy `store.ping()` trả `False` → endpoint trả `503`. (2) Vì đây cũng
> chính là endpoint mà orchestrator dùng làm **liveness probe**, orchestrator
> hiểu `503` ở đây là "process đã chết, cần restart" (đúng nghĩa liveness),
> chứ không phải "tạm thời chưa sẵn sàng, đừng gửi traffic vào" (nghĩa
> readiness). (3) Orchestrator ra lệnh restart cả 3 container gần như CÙNG
> LÚC, vì cả 3 phụ thuộc chung một Redis và cùng mất kết nối cùng lúc. (4)
> Trong lúc cả 3 đang restart, nếu Redis vẫn chưa hồi phục hết trong khoảng
> 30 giây đó, container mới khởi động lại cũng lập tức thấy Redis lỗi → lại
> bị đánh giá "chưa sống" → có thể bị restart tiếp, tạo vòng lặp restart liên
> tục. (5) Kết quả: khi Redis phục hồi, có thể không còn container nào đang ở
> trạng thái phục vụ được — một sự cố Redis 30 giây (vốn chỉ nên khiến load
> balancer tạm ngừng gửi traffic) biến thành downtime toàn hệ thống, dù code
> app không hề có lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với `store.py` hiện tại (lịch sử lưu trong Redis List qua `ConversationStore`),
> mình gọi `/ask` liên tiếp với cùng `X-User-Id` và thấy `history_length` tăng
> đều đặn (0 → 2 → 4 → ...) — đúng như test `test_lich_su_duoc_dung_lai_giua_cac_request`
> đã xác nhận — bất kể request rơi vào container nào trong 3 container, vì cả
> 3 cùng đọc/ghi chung một Redis.
>
> Nếu lịch sử được lưu trong một dict Python thường (biến toàn cục trong
> process) thay vì Redis: mỗi container có RAM riêng, dict đó không được chia
> sẻ giữa các container. `history_length` sẽ KHÔNG tăng đều nữa mà nhảy lung
> tung tùy load balancer route request vào container nào — ví dụ request 1
> vào container A (`history_length=0`), request 2 lại rơi vào container B
> (dict của B chưa từng thấy user này nên cũng trả `history_length=0`, đáng
> lẽ phải là 2), request 3 quay lại A thì thấy `history_length=2`... Agent
> trông như liên tục "mất trí nhớ" một cách ngẫu nhiên — đúng lý do CP4 bắt
> buộc chuyển state sang Redis.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Deploy lên Railway, `railway up` báo build thành công nhưng container không
> khởi động được.
>
> **Thông báo lỗi** (trong `railway logs` ngay sau khi container start):
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> Usage: uvicorn [OPTIONS] APP
> ```
>
> **Cách tìm nguyên nhân:** `docker build` chạy hoàn toàn ổn ở bước build (mọi
> layer build xong, image push thành công), nên lỗi không nằm ở Dockerfile.
> Nhìn vào thông báo, `uvicorn` nhận đúng chuỗi ký tự `"$PORT"` làm giá trị
> `--port` thay vì con số thật — nghĩa là biến môi trường `$PORT` không được
> **shell thay thế (expand)** trước khi tới `uvicorn`. Kiểm tra lại thấy
> `railway.toml` có khai `startCommand = "uvicorn app.main:app --host 0.0.0.0
> --port $PORT"` — dòng này **đè lên `CMD` trong Dockerfile**, và Railway chạy
> `startCommand` không thông qua shell, nên `$PORT` không hề được thay giá trị,
> khác với `CMD` gốc trong Dockerfile vốn đã bọc sẵn trong `sh -c "... --port
> ${PORT:-8000}"` (dùng cú pháp shell mới thay được).
>
> **Sửa:** xóa hẳn dòng `startCommand` khỏi `railway.toml`, để Railway tự dùng
> đúng `CMD` của Dockerfile (đã có `sh -c` bọc sẵn, xử lý `$PORT` đúng cách).
> Sau khi xóa và `railway up` lại, container khởi động thành công, healthcheck
> `/health` pass ngay lần đầu.
>
> (Trong lúc dựng lại service trên Railway còn gặp thêm 2 sự cố nữa: lỡ deploy
> code agent đè lên service Redis do CLI tự link nhầm sau `railway add
> --database redis`, và Docker Desktop ở máy ban đầu thiếu Windows Service
> `com.docker.service` do cài không bằng quyền Admin — cả hai đều đã xử lý
> xong trước khi có kết quả deploy thật trong `DEPLOYMENT.md`.)
