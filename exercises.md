# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đậu Quang Ý  Mã học viên: 2A202602661

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `agent_api_key = "changeme"`, khi deploy lên cloud (như Railway/Render) mà người phát triển quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động thành công và báo trạng thái Healthy. Khi đó, API công khai sẽ chấp nhận khóa `"changeme"`. Kẻ tấn công hoặc bot quét API tự động có thể thử các khóa mặc định phổ biến và gọi dịch vụ AI hoàn toàn miễn phí, làm cạn kiệt ngân sách hoặc lộ tài nguyên mà ta không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, khi không có giá trị mặc định, Pydantic sẽ raise `ValidationError` ngay lúc service khởi động (nguyên lý Fail-fast), deployment trên cloud sẽ lập tức báo lỗi (crash loop) và ngăn chặn việc public một dịch vụ không an toàn ra Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:50:00.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.00015}`

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn và lọc có cấu trúc (Structured Querying & Filtering)**: Các hệ thống gom log tập trung (Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động parse các trường JSON để lọc theo điều kiện cụ thể (ví dụ: truy vấn toàn bộ request của `user_id == 'sv-test'`, hoặc tìm các lượt gọi có `cost_usd > 0.01`), trong khi `print` dạng văn bản tự do đòi hỏi phải viết Regex phức tạp và rất dễ sai sót.
2. **Thống kê định lượng và thiết lập cảnh báo tự động (Metrics & Alerting)**: Có thể dễ dàng tổng hợp số liệu định lượng theo thời gian thực (ví dụ: tính tổng chi phí `sum(cost_usd)`, đo đạc số token vào/ra trung bình, hoặc kích hoạt cảnh báo tới Slack/PagerDuty khi `level == "error"` vượt ngưỡng), điều mà chuỗi plain text thông thường không thể tự động hóa một cách tin cậy.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800MB+) bao gồm:
1. **Sự khác biệt về base image**: Bản 1 stage dùng `python:3.11` đầy đủ dựa trên Debian bản chuẩn, chứa hàng loạt công cụ hệ thống, trình quản lý gói đầy đủ, các thư viện C/C++ build tools không cần thiết cho lúc chạy. Bản multi-stage dùng `python:3.11-slim`, chỉ giữ lại runtime tối thiểu của Debian.
2. **Loại bỏ công cụ build và cache biên dịch**: Ở stage `builder`, các công cụ như compiler (`gcc`, `g++`), header files (`python-dev`), pip build caches, wheel cache phục vụ quá trình cài đặt đều bị loại bỏ hoàn toàn khỏi image cuối. Stage `runtime` chỉ copy duy nhất thư mục kết quả `/install` sang `/usr/local`, giúp container cuối cùng không mang theo các gói build cồng kềnh, giảm attack surface và tối ưu kích thước image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Docker cache theo từng layer từ trên xuống dưới. Các layer cài đặt môi trường và thư viện (`FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...`) không bị thay đổi nên Docker tái sử dụng lại 100% từ cache (`CACHED`). Chỉ từ layer copy code ứng dụng (`COPY app ./app`) và các lệnh phía sau mới phải chạy lại. Nhờ đó thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ ký tự nào trong `app/main.py`, layer `COPY . .` sẽ bị thay đổi checksum, làm Docker vô hiệu hóa cache (cache invalidation) của chính nó và toàn bộ các layer phía sau. Hậu quả là lệnh `RUN pip install` buộc phải chạy lại từ đầu, tải và cài đặt lại toàn bộ thư viện mỗi lần sửa code, làm chậm quá trình CI/CD và tiêu tốn băng thông/thời gian build một cách lãng phí.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện khi chạy bằng root (UID 0):
  1. Kẻ tấn công khai thác lỗ hổng RCE (Remote Code Execution) hoặc Path Traversal trong code Python hoặc thư viện dependency.
  2. Kẻ tấn công thực thi mã shell bên trong container. Do tiến trình chạy bằng user `root` (UID 0 bên trong container, trùng mapping với root UID 0 trên Linux host nếu không dùng user namespace), kẻ tấn công có toàn quyền đọc, ghi, sửa file hệ thống trong container.
  3. Kẻ tấn công khai thác tiếp các lỗ hổng container escape (ví dụ: kernel exploit, Docker socket mounted `/var/run/docker.sock`, hoặc các privileged capabilities chưa drop) để thoát ra ngoài máy host.
  4. Khi thoát ra máy host, tiến trình vẫn giữ quyền root (UID 0) trên host, cho phép kẻ tấn công chiếm toàn quyền kiểm soát máy chủ vật lý/VM.
- Lệnh `USER appuser` cắt đứt chuỗi này: Lệnh này hạ quyền tiến trình xuống một user không đặc quyền (`appuser` với UID 10001, không có quyền `sudo`). Khi bị khai thác RCE, kẻ tấn công chỉ có quyền hạn chế của `appuser`, không thể sửa đổi file hệ thống, không thể tương tác với docker socket hoặc khai thác các syscall yêu cầu root privilege, do đó ngăn chặn hiệu quả nguy cơ container escape và leo thang đặc quyền lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
- Giải thích:
  Với cơ chế fixed-window (đếm theo phút đồng hồ và reset lúc giây 00):
  + Ở phút thứ nhất, người dùng gửi 10 request dồn vào giây cuối cùng (giây thứ 59, ví dụ 10:00:59). Hệ thống ghi nhận 10/10 request cho phút đó và hợp lệ.
  + Ngay khi đồng hồ điểm sang giây 00 của phút thứ hai (10:01:00), bộ đếm bị reset về 0. Người dùng lập tức gửi tiếp 10 request nữa trong giây này (10:01:00).
  + Tổng cộng: từ 10:00:59 đến 10:01:00 (chỉ trong 2 giây liên tiếp), người dùng đã gửi thành công 10 + 10 = 20 request mà không vi phạm quy tắc phút đồng hồ. Điều này tạo ra đột biến tải gấp đôi (burst traffic) làm sập hệ thống. Thuật toán sliding window khắc phục hoàn toàn hiện tượng này bằng cách luôn tính chính xác số lượng request trong khoảng thời gian trôi 60 giây bất kỳ tính từ thời điểm hiện tại `[now - 60s, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  + Rate limit kiểm soát **tần suất / số lượng request** trong một đơn vị thời gian ngắn (ví dụ: 10 request / 60 giây), nhằm bảo vệ hệ thống khỏi nghẽn mạng, tấn công DoS, và giữ độ ổn định cho server.
  + Cost guard kiểm soát **chi phí tài chính / mức tiêu thụ tài nguyên thực tế** trong chu kỳ dài (ví dụ: $10.0 / tháng), tính theo lượng token LLM hoặc tiền USD, nhằm bảo vệ ngân sách của chủ sở hữu hệ thống.
- Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
  Người dùng chỉ gửi 1 request duy nhất trong ngày (hoàn toàn thỏa mãn giới hạn 10 request/phút), nhưng request đó tải lên một tài liệu khổng lồ tiêu tốn 150.000 tokens LLM, có chi phí ước tính là $3.0. Nếu tài khoản của người dùng trong tháng đó đã tiêu $9.5/$10.0, Cost guard sẽ chặn ngay lập tức (lỗi 402 Payment Required) để bảo vệ ngân sách, mặc dù Rate limit không hề kích hoạt.
- Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
  Đầu tháng, người dùng chưa tiêu đồng nào (ngân sách còn nguyên $10.0). Người dùng viết script gửi 30 câu hỏi cực ngắn "hi" liên tục trong vòng 5 giây. Chi phí của 30 câu hỏi này chỉ tốn $0.003 (rất nhỏ so với ngân sách $10.0, Cost guard sẽ duyệt), nhưng việc bắn 30 request trong 5 giây đã vượt quá 10 req/phút của Rate limiter, do đó Rate limiter sẽ chặn từ request thứ 11 với mã lỗi 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra khi gộp kiểm tra Redis vào `/health`:
1. Giây 0: Redis gặp sự cố tạm thời hoặc khởi động lại (mất kết nối trong 30 giây).
2. Giây 1 - 10: Endpoint `/health` của cả 3 container gọi kiểm tra Redis và đều bị fail (trả về lỗi hoặc timeout).
3. Giây 10 - 20: Bộ giám sát của nền tảng (Docker daemon / Kubernetes kubelet / Cloud platform) dựa vào `/health` (liveness probe) thấy cả 3 container đều un-healthy. Hệ thống cho rằng tiến trình app đã bị treo/hỏng và tự động **kill (SIGKILL) cả 3 container** rồi khởi động lại chúng liên tục (crash loop / restart storm).
4. Giây 20 - 30: Cả 3 container vừa khởi động lên lại tiếp tục check `/health` -> Redis vẫn chưa sẵn sàng -> lại bị kill và restart lần nữa. Toàn bộ tài nguyên CPU/RAM bị đốt vào việc boot container vô ích.
5. Giây 30+: Khi Redis vừa hồi phục, cả cụm 3 container đang dở dang ở trạng thái restart/chết, không container nào có thể phục vụ request ngay. Nếu có request người dùng đến sẽ bị mất hoàn toàn (downtime).
*Ngược lại, nếu tách riêng: `/health` vẫn 200 (giữ container sống), chỉ `/ready` trả 503 (load balancer tạm dừng điều phối traffic). Khi Redis sống lại sau 30 giây, `/ready` lập tức xanh trở lại và cụm container tiếp tục phục vụ bình thường mà không cần restart.*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless): `history_length` tăng đều đặn theo mỗi lượt hội thoại: 0, 2, 4, 6... bất kể request được load balancer định tuyến ngẫu nhiên vào instance 1, 2 hay 3, vì cả 3 instance đều cùng truy vấn và cập nhật vào một kho Redis tập trung.
- Nếu lưu trong một dict Python trong RAM của từng tiến trình (Stateful):
  Vì mỗi container có không gian bộ nhớ (RAM) hoàn toàn độc lập, khi load balancer phân phối các request kế tiếp nhau theo cơ chế round-robin vào các instance khác nhau:
  + Lần 1 vào container A: `history_length` = 0. Container A lưu 2 message vào RAM của A.
  + Lần 2 vào container B: `history_length` = 0 (thay vì 2), vì RAM của B hoàn toàn trống rỗng! Agent trên container B trả lời như thể chưa từng nói chuyện với user.
  + Lần 3 vào container C: `history_length` lại là 0.
  + Lần 4 quay lại container A: `history_length` mới nhảy lên 2.
  Hậu quả là `history_length` nhảy loạn xạ (0, 0, 0, 2, 2...), ngữ cảnh trò chuyện bị đứt đoạn và agent bị "mất trí nhớ" ngẫu nhiên tùy thuộc vào việc request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Application failed to respond on PORT 8080 within timeout / Health check failed on port 8000: connection refused`.
- Cách tìm ra nguyên nhân: Kiểm tra log runtime trên Dashboard của nền tảng cloud (Render/Railway), nhận thấy nền tảng tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=8080`), trong khi file cấu hình hoặc lệnh khởi chạy ban đầu lại hardcode `--port 8000`. Khi container chạy, uvicorn lắng nghe trên cổng 8000, còn load balancer của Cloud lại gửi probe kiểm tra sức khỏe vào cổng 8080 mà nó đã cấp. Kết quả là healthcheck timeout và cloud hủy deployment.
- Cách sửa: Cập nhật lại lệnh CMD trong `Dockerfile` thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để uvicorn linh hoạt nhận cổng từ biến môi trường `$PORT` do platform chỉ định; đồng thời trong `app/config.py`, trường `port: int = 8000` của Pydantic `BaseSettings` sẽ tự động ghi đè bằng giá trị của biến môi trường `PORT` khi chạy trên cloud.
