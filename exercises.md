# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Khắc Giáp  Mã học viên: 2A202602950

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

- Tình huống: deploy lên Railway nhưng quên đặt `AGENT_API_KEY` trên dashboard.
- Nếu có mặc định `"changeme"`: app vẫn lên, `/health` 200, nhìn như mọi thứ ổn.
  Nhưng `"changeme"` nằm công khai trong repo → ai đọc code cũng gọi được `/ask`,
  đốt ngân sách LLM của mình mà mình không biết.
- Không có mặc định: app chết ngay lúc khởi động, log báo rõ lỗi (đo thật):
  ```
  ValidationError: 1 validation error for Settings
  agent_api_key
    Field required [type=missing, ...]
  ```
  → deploy fail, healthcheck không qua, mình phát hiện trong vài giây thay vì sau hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thật, lượt thứ 3 khi gọi `/ask` 3 lần liên tiếp với user `sv01`:

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T14:43:20.006239+00:00", "user_id": "sv01", "tokens_in": 93, "tokens_out": 44, "cost_usd": 4.035e-05}
```

Hai việc làm được mà `print` không làm được:

1. Lọc theo trường: trên log viewer của Railway/Datadog tìm đúng mọi request của
   `user_id = sv01`, hoặc mọi dòng `level = error`, không phải dò chữ bằng mắt hay regex.
2. Tính toán và cảnh báo: cộng `cost_usd` theo user theo ngày, đếm số request mỗi
   phút, đặt alert khi chi phí tăng bất thường. Chuỗi "đã trả lời xong" không có số
   nào để tính.

Nhờ log có số liệu mà mình thấy được `tokens_in` tăng 3 → 44 → 93 qua 3 lượt, vì
history được gửi kèm mỗi lần hỏi. Với `print` thì không biết được chuyện này.

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
| 1 stage (bản đầu) | 1.73 GB trên đĩa (447 MB nén) |
| Multi-stage | 310 MB trên đĩa (71.9 MB nén) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Docker Desktop của mình dùng containerd nên `docker images` hiện 2 cột: dung lượng
trên đĩa và dung lượng nén (lúc push/pull). Bản 1 stage lấy từ commit gốc `306b897`.

Xem `docker history` thì phần chênh gần như không nằm ở thư viện của app: layer
`pip install` ở bản 1 stage là 95.1 MB, layer `/opt/venv` ở bản multi-stage là 96 MB,
gần bằng nhau. Chênh lệch nằm ở base image: `python:3.11` nặng 1.61 GB, còn
`python:3.11-slim` chỉ 189 MB. Bản đầy đủ có 2 layer apt nặng 694 MB và 202 MB, chứa
gcc, các gói header `-dev`, build tools, git... Những thứ đó chỉ cần khi compile, app
chạy thì không dùng. `COPY . .` ở bản 1 stage chỉ 356 kB vì `.dockerignore` đã loại
`.git`, `.venv`, tests.

Multi-stage giúp ở chỗ việc cài đặt xong xuôi ở stage builder, stage runtime chỉ cần
copy `/opt/venv` + `app/` + `utils/` sang một base slim, không mang theo đồ nghề build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình thêm 1 dòng comment vào cuối `app/main.py` rồi build lại với
`--progress=plain`.

Với Dockerfile hiện tại, build lại mất 5.4 giây:
- Dùng lại cache (`CACHED`): `python -m venv`, `COPY requirements.txt`,
  `RUN pip install`, `useradd`, `WORKDIR /app`, `COPY --from=builder /opt/venv`.
- Chạy lại: `COPY app/` (vì file đổi) và `COPY utils/` (layer đứng sau layer đã đổi).

Thử một Dockerfile tạm đặt `COPY . .` trước `pip install` rồi làm y như vậy: build
lại mất 25.9 giây. `COPY . .` đổi vì `main.py` đổi, nên `RUN pip install` mất cache
và chạy lại hết (riêng bước này 15.6 giây), kéo theo `COPY /opt/venv` cũng chạy lại.
Tức là sửa 1 ký tự code mà phải cài lại toàn bộ thư viện, chậm gần 5 lần. Trên CI
hoặc khi requirements nhiều hơn thì còn chênh nữa.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

1. Code có lỗ hổng cho chạy lệnh tùy ý, ví dụ đưa input người dùng vào
   `subprocess`/`eval`, hoặc một thư viện dính lỗi RCE.
2. Kẻ tấn công chạy được lệnh trong container, với đúng quyền của process app. Nếu
   container chạy bằng root thì đó là root.
3. Root trong container là UID 0 thật trên kernel của host, vì Docker mặc định không
   đổi UID.
4. Có root thì sửa được code app, cài thêm tool bằng `apt`, đọc/ghi mọi thứ mount từ
   host vào. Gặp lỗ hổng container escape, hoặc lỡ mount `/var/run/docker.sock`, là
   thành root trên máy host.

`USER appuser` (UID 10001) cắt ở bước 2: shell của kẻ tấn công chỉ là user thường.
Không ghi được vào `/app` (thuộc root), không cài thêm được gì, và nếu có thoát ra
khỏi container thì cũng chỉ là một user không có đặc quyền trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây.

Cách làm: gửi 10 request lúc 10:00:59, dùng hết quota của phút 10:00. Sang 10:01:00
bộ đếm reset về 0, gửi tiếp 10 request. Vậy là 20 request trong khoảng 1–2 giây,
gấp đôi hạn mức.

Với sliding window thì ở 10:01:00 hệ thống vẫn đếm 60 giây gần nhất, vẫn thấy 10
request lúc 10:00:59, nên request thứ 11 bị 429. Đo thật trên bản deploy Railway:
gửi 15 request liên tiếp thì 10 cái 200, 5 cái sau 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit đếm số request trong 60 giây gần nhất, vượt thì 429. Cost guard đếm số
tiền đã tiêu trong tháng, vượt thì 402. Một cái chặn gọi quá nhanh, một cái chặn
tiêu quá nhiều.

- Rate limit cho qua, cost guard chặn: user gửi đều, chưa bao giờ quá 10 request/phút,
  nhưng mỗi request rất đắt: prompt dài, hội thoại dài. Log câu 2 cho thấy
  `tokens_in` tăng theo history. Gửi như vậy cả tháng thì vượt $10, bị 402 dù chưa
  lần nào dính 429.
- Ngược lại: một script spam 15 request rẻ trong 1 giây, mỗi cái khoảng $0.00002.
  Request thứ 11 đã bị 429, trong khi tổng chi phí chưa tới $0.001 so với ngân sách
  $10, cost guard không có lý do gì để chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối → cả 3 container cùng lúc trả 503 ở endpoint health.
2. Orchestrator thấy health check fail vài lần liên tiếp, tưởng container hỏng, nên
   restart cả 3.
3. Restart không sửa được Redis. Container mới lên vẫn 503, lại bị restart, rơi vào
   vòng crash loop, và orchestrator chờ lâu dần giữa các lần restart (backoff).
4. Trong lúc đó request đang xử lý dở bị cắt ngang, không còn instance nào phục vụ.
5. Redis sống lại sau 30 giây, nhưng container đang kẹt trong vòng restart/backoff,
   nên sự cố kéo dài lâu hơn nhiều so với 30 giây ban đầu.

Nếu tách ra: `/health` vẫn 200 vì process vẫn ổn, không ai restart gì cả. `/ready`
trả 503 nên load balancer tạm không gửi traffic vào. Redis về thì `/ready` 200 lại
và nhận traffic ngay, không container nào phải khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Mình chạy `docker compose up --scale agent=3`, cho mỗi container ra một cổng riêng
(18000, 18001, 18002, vì cổng 8000 máy đang bận). Gọi `/ask` 6 lần xoay vòng 3 cổng
với cùng một `X-User-Id`, nên mỗi request chắc chắn rơi vào một container khác nhau.
Để thấy trường hợp dict Python, mình chạy lại với `REDIS_URL=fake://`: mỗi container
giữ Redis giả trong RAM của riêng nó, giống hệt dùng dict.

| Lượt | Cổng | Lưu trong Redis | Lưu trong RAM từng container |
|------|------|-----|-----|
| 1 | 18000 | 0 | 0 |
| 2 | 18001 | 2 | 0 |
| 3 | 18002 | 4 | 0 |
| 4 | 18000 | 6 | 2 |
| 5 | 18001 | 8 | 2 |
| 6 | 18002 | 10 | 2 |

Với Redis, con số tăng đều 0 → 10 vì cả 3 container đọc chung một lịch sử. Lưu trong
RAM thì mỗi container chỉ nhớ những lượt rơi vào nó: user hỏi 6 câu mà agent chỉ nhớ
được 2 message. Mình thử thêm: gọi 2 lần vào cổng 18000 (0 → 2), `docker restart`
container đó rồi gọi lại thì về 0. Lưu trong RAM thì deploy lại hay restart là mất sạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thật đã gặp:
- Thông báo: "Deploying service ... There was an error deploying from source."
  Trước đó trang Settings báo "Could not load branches" / "Could not load public networking".
- Tìm nguyên nhân: xem tab Deployments → không có deployment nào; kiểm tra config service →
  service không có `source` (không gắn repo). Staged changes lúc bấm Deploy không được áp dụng
  nên service tạo ra trống.
- Sửa: Discard thay đổi đang treo, xóa service lỗi, tạo lại bằng + Create → GitHub Repo
  (service có source ngay), đặt Variables, Generate Domain → deploy thành công, `/health` 200.
