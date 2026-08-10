# Phiếu Phản Ánh — K4 Ngày 12

**Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hải Yến  Mã học viên: 2A202601604

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi mình deploy lên Railway lần đầu (xem `DEPLOYMENT.md`), mình phải set
> `API_TOKEN` trong dashboard trước khi service chạy được — nếu quên set,
> container crash ngay lúc khởi động và Railway báo "deploy failed" rõ ràng
> trong log. Mình biết ngay là thiếu biến môi trường và sửa trước khi public
> URL từng được công bố cho ai.
>
> Nếu `api_token` có mặc định `"changeme"`, container vẫn khởi động bình
> thường, healthz/readyz vẫn trả 200, mình sẽ tưởng deploy thành công. Nhưng
> `/chat` lúc đó chấp nhận bất kỳ ai gửi `Authorization: Bearer changeme` —
> vì đây là giá trị mặc định trong code công khai trên GitHub, ai đọc source
> cũng biết. Người lạ có thể gọi LLM miễn phí bằng ngân sách của mình, và
> mình chỉ phát hiện ra khi nhận hóa đơn hoặc khi `cost_guard` báo 402 hàng
> loạt — tức là đã mất tiền rồi mới biết có lỗ hổng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy được sau khi chạy `uvicorn app.main:app` cục bộ và gọi
> `/chat`:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:37:49.362742+00:00", "client_id": "demo", "prompt_tokens": 2, "completion_tokens": 34, "usd_cost": 2.07e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và tổng hợp theo trường.** Vì log là JSON có cấu trúc, mình có thể
>    chạy truy vấn kiểu `sum(usd_cost) group by client_id` hoặc lọc
>    `event = "chat_completed" AND client_id = "demo"` trên Cloud Logging /
>    Datadog để biết chính xác client nào tốn tiền nhất trong ngày. Với
>    `print(...)`, đó chỉ là một chuỗi văn bản — muốn lấy `client_id` hay
>    `usd_cost` ra phải viết regex đoán mò và dễ vỡ khi format đổi.
> 2. **Đặt cảnh báo (alert) tự động.** Trường `severity` viết hoa đúng chuẩn
>    mà Google Cloud Logging/Datadog hiểu, nên có thể set alert kiểu "báo tôi
>    nếu có dòng `severity=ERROR` trong 5 phút" hoặc "báo nếu tổng
>    `usd_cost` vượt ngưỡng". `print` không có trường mức độ nghiêm trọng để
>    hệ thống phân loại, nên không thể alert theo mức độ được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1700 MB (1.7 GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Đo thật bằng `docker images`: bản 1-stage build từ `python:3.11` (bản đầy
> đủ, có gcc, các thư viện dev, header C...) dùng để vừa cài dependency vừa
> chạy app trong cùng một layer, và `COPY . .` mang theo toàn bộ thư mục dự
> án (kể cả file không cần lúc runtime). Bản multi-stage tách stage
> `builder` (cũng `python:3.11-slim`) để chạy `pip install`, rồi chỉ
> `COPY --from=builder /install /usr/local` — tức là chỉ lấy các gói Python
> đã cài, không mang theo compiler/toolchain dùng để build chúng — sang
> image chạy cuối cùng dựa trên `python:3.11-slim`.
>
> Phần chênh lệch ~1.4 GB chủ yếu là: (1) base image đầy đủ so với `-slim`
> (khác biệt hàng trăm MB do các gói hệ thống không cần cho runtime), và
> (2) build tool-chain (gcc, make, header files) mà `pip install` một số gói
> cần lúc *cài đặt* nhưng không cần lúc *chạy* — multi-stage vứt bỏ chúng
> hoàn toàn ở stage cuối, image chạy production chỉ còn Python interpreter +
> package đã cài + source code.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile hiện tại: `COPY requirements.txt .` → `RUN pip install` (stage
> `builder`) → `COPY --from=builder ...` → `COPY app/` → `COPY utils/` →
> `USER appuser` → `HEALTHCHECK`/`CMD`. Khi sửa một ký tự trong
> `app/main.py` rồi build lại:
>
> - **Dùng lại từ cache:** toàn bộ stage `builder` — `COPY requirements.txt .`
>   và `RUN pip install ...` — vì Docker so nội dung file `requirements.txt`
>   không đổi, layer đó khớp cache, bỏ qua luôn (không tải lại package nào).
>   `COPY --from=builder /install /usr/local` ở stage runtime cũng cache vì
>   builder stage không đổi.
> - **Phải chạy lại:** `COPY app/ app/` (vì nội dung thư mục `app/` đã đổi)
>   và mọi layer *sau* nó trong cùng stage — `COPY utils/`, `RUN useradd`,
>   v.v. — vì Docker cache theo thứ tự tuyến tính: một layer đổi thì mọi
>   layer đứng sau nó trong cùng stage bị invalidate, dù nội dung của chúng
>   không đổi.
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install` thì mất hết lợi ích
> cache: bất kỳ thay đổi nào trong source code (kể cả sửa một dấu chấm ở
> `main.py`) đều làm layer `COPY . .` bị invalidate, kéo theo `RUN pip
> install` — vốn đứng ngay sau — cũng phải chạy lại từ đầu, tải và cài lại
> toàn bộ dependency dù `requirements.txt` không hề đổi. Với build hiện tại
> mất khoảng 30–40 giây tải package chỉ để sửa một dòng code, thay vì build
> gần như tức thì.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của mình có một lỗ hổng — ví dụ một
> dependency (`fastapi`, `pydantic`, ...) có lỗ hổng deserialization, hoặc
> một endpoint xử lý input không an toàn — cho phép kẻ tấn công thực thi
> lệnh tùy ý trong tiến trình Python (RCE). (2) Nếu process đó chạy bằng
> `root` bên trong container (hành vi mặc định khi Dockerfile không có
> `USER`), lệnh tùy ý đó cũng chạy với quyền root *trong container*. (3) Từ
> đó, kẻ tấn công có thể khai thác thêm một lỗ hổng escape-container (kernel
> exploit, container misconfig, mount `/var/run/docker.sock`, hay đơn giản
> là container chạy `--privileged`) để leo thang từ "root trong container"
> thành "root trên máy host" — lúc đó họ kiểm soát toàn bộ host, không chỉ
> service của mình.
>
> Lệnh `USER` trong Dockerfile của mình (`RUN useradd --create-home --uid
> 10001 appuser` rồi `USER appuser`) cắt đứt chuỗi này ở bước (2): dù bước
> (1) vẫn xảy ra (RCE trong code Python), tiến trình bị chiếm quyền chỉ có
> quyền của `appuser` — một user thường, uid 10001, không có quyền ghi vào
> hệ thống file ngoài phạm vi được cấp, không có quyền root trong container.
> Muốn escape ra host lúc này khó hơn nhiều bậc vì kẻ tấn công phải tìm thêm
> một lỗ hổng leo thang đặc quyền *bên trong* container trước, thay vì đã có
> sẵn root ngay từ bước đầu.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là yêu cầu bắt buộc của chuẩn HTTP (RFC 7235):
> khi server trả 401, nó phải nói cho client biết *cách* xác thực — ở đây là
> "dùng scheme Bearer". Thiếu header này, client (và cả thư viện HTTP chuẩn
> như `requests`, trình duyệt, hay công cụ như Postman) không biết phải gửi
> credential theo định dạng nào, và một số middleware/proxy dựa vào header
> này để tự động thử lại request với token đính kèm đúng chỗ.
>
> Trả **cùng một** thông báo `"invalid or missing bearer token"` cho cả ba
> trường hợp (thiếu header, sai scheme, sai token) là để không tặng thông
> tin cho kẻ đang dò token. Nếu server trả lỗi khác nhau kiểu "thiếu header"
> vs "sai token", kẻ tấn công dò token bằng cách thử hàng loạt giá trị có
> thể phân biệt được lúc nào họ đã "đến đúng cửa" (header đúng định dạng,
> chỉ sai giá trị token) và tập trung brute-force vào đó, thay vì phải đoán
> mù cả cấu trúc request lẫn giá trị token. Thông báo mơ hồ, dùng chung buộc
> kẻ tấn công phải đoán mọi thứ cùng lúc, không có tín hiệu để thu hẹp phạm
> vi dò.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> `refill_per_second = 10/60 ≈ 0.1667`. Giả sử client vừa tiêu hết xô
> (`tokens=0`) rồi im lặng đúng 10 phút (600 giây): phần nạp thêm trong lúc
> im lặng là `600 × 0.1667 ≈ 100` token. Với dòng `min(capacity, tokens)`
> đang có trong code, xô bị chặn ở `capacity=10` dù tính ra 100 — nên client
> gửi liên tiếp được đúng **10 request** thành công, request thứ 11 nhận
> 429.
>
> Nếu bỏ `min(capacity, ...)`, `available()` trả thẳng ~100 (không bị chặn
> trần), nên client gửi liên tiếp được khoảng **100 request** trước khi hết
> token và bị 429 — gấp 10 lần `capacity` đã khai báo. Lý do: xô lẽ ra chỉ
> nên "chứa" tối đa `capacity` token bất kể im lặng bao lâu (đúng ý nghĩa từ
> "sức chứa"); thiếu `min()`, thời gian im lặng biến thành một khoản tín
> dụng tích lũy không giới hạn, và im lặng càng lâu thì burst cho phép sau
> đó càng lớn — hoàn toàn phá vỡ mục đích chặn burst của rate limiter.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **Hạn mức $30/tháng:** nếu sự cố xảy ra ngay đầu chu kỳ (ví dụ ngày 1 lúc
> 2h sáng), client có thể đốt toàn bộ **$30** trong một lần liên tục trước
> khi `guard.check()` chặn — vì hệ thống chỉ nhìn tổng chi tiêu của cả
> tháng, không phân biệt "1 sự cố dồn dập" với "dùng đều cả tháng". Sau khi
> chạm trần, service bị khóa (402) cho *client đó* tới tận đầu tháng sau —
> tức có thể phải chờ gần 30 ngày mới tự hồi phục, trừ khi có người can
> thiệp thủ công reset hạn mức.
>
> **Hạn mức $1/ngày** (cách `CostGuard` trong code đang dùng, khóa theo
> `spend:{client_id}:{YYYY-MM-DD}`): thiệt hại tối đa của cùng sự cố chỉ là
> **$1** — đúng 1/30 so với cách trên — vì key chi tiêu được tính riêng theo
> từng ngày UTC. Ngay khi đồng hồ sang ngày UTC mới (chậm nhất là tới nửa
> đêm UTC cùng ngày, tức tối đa khoảng 22 giờ sau nếu sự cố bắt đầu lúc 2h
> sáng UTC), key ngày cũ hết hiệu lực, `spent()` cho ngày mới trả về `0.0`,
> và client tự động được cấp lại ngân sách mà không cần ai can thiệp. Hạn
> mức theo ngày vừa giới hạn thiệt hại tối đa nhỏ hơn nhiều, vừa tự hồi phục
> nhanh hơn nhiều so với hạn mức theo tháng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu `/healthz` (liveness) cũng gọi `store.ping()`:
>
> 1. Redis mất kết nối. Cả 3 container đều dùng chung một Redis, nên cả 3
>    đều bị ảnh hưởng gần như đồng thời.
> 2. Lần liveness probe kế tiếp của orchestrator (mỗi container tự có chu kỳ
>    probe riêng, thường mỗi vài giây tới vài chục giây) gọi `/healthz`,
>    endpoint này gọi `store.ping()`, Redis không phản hồi → trả 503.
> 3. Orchestrator hiểu 503 ở endpoint **liveness** là "process đã chết,
>    không tự hồi phục được" (khác readiness, vốn có nghĩa là "tạm thời
>    đừng gửi traffic tới") → nó **kill và restart** container đó, dù tiến
>    trình Python vẫn đang chạy hoàn toàn bình thường, chỉ là Redis đang bận.
> 4. Vì cả 3 container probe gần như cùng lúc và cùng phụ thuộc một Redis,
>    cả 3 đều bị đánh rớt liveness và bị restart gần như đồng thời → **toàn
>    bộ cụm mất khả năng phục vụ** trong lúc restart, dù chỉ có 1 dependency
>    (Redis) gặp sự cố tạm thời.
> 5. Trong lúc 3 container đang khởi động lại (tốn vài giây tới vài chục
>    giây mỗi container), Redis đã kết nối lại sau đúng 30 giây như đề bài —
>    nhưng cụm vẫn chưa phục vụ được vì đang trong pha restart, có thể
>    healthz tiếp tục fail ở vài lần probe kế nếu Redis chưa ổn định ngay,
>    gây ra một vòng crash-loop ngắn.
> 6. Kết quả: một sự cố Redis 30 giây — vốn `/readyz` xử lý gọn bằng cách
>    chỉ tạm ngừng nhận traffic mới mà không giết container — biến thành
>    một đợt outage toàn cụm kéo dài hơn 30 giây, hoàn toàn do gộp nhầm
>    liveness với readiness. Đây chính là lý do `/healthz` phải "nhẹ" (không
>    đụng Redis) còn `/readyz` mới được phép kiểm tra dependency.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu tiên lên Railway bị đánh dấu **FAILED — Network >
> Healthcheck**. Deploy Logs cho thấy uvicorn tự thoát ngay khi khởi động
> với lỗi:
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
> và Network Logs báo `/healthz` không bao giờ trả lời: `1/1 replicas never
> became healthy!`. Nghĩa là container không hề chết vì thiếu biến môi
> trường hay sai code — nó nhận đúng chữ `"$PORT"` (chuỗi, có cả ký tự `$`)
> làm giá trị cổng thay vì một con số, nên uvicorn crash trước khi kịp mở
> cổng.
>
> Mình đi tìm nguyên nhân theo hướng loại trừ: đầu tiên nghi ngờ thiếu biến
> môi trường, vào tab **Variables** kiểm tra thì thấy `API_TOKEN` chưa được
> set — nhưng đó là nghi vấn phụ, không giải thích được tại sao `$PORT` lại
> không expand. Tiếp theo kiểm tra **Settings → Deploy → Custom Start
> Command** — ô này trống, chỉ có placeholder xám, nên không phải nguyên
> nhân. Cuối cùng phát hiện file `railway.toml` ngay trong repo có dòng:
> ```toml
> startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"
> ```
> Dòng này ghi đè `CMD` trong Dockerfile. Vấn đề là Railway chạy
> `startCommand` **không qua shell**, nên `$PORT` không được hệ điều hành
> expand thành số trước khi truyền cho uvicorn — trong khi `CMD` gốc ở
> Dockerfile có bọc `sh -c "..."` nên mới expand đúng. `startCommand` trong
> `railway.toml` chạy thẳng như một argv list, không đi qua shell, nên biến
> môi trường bị truyền nguyên văn dưới dạng chuỗi ký tự.
>
> Cách sửa: đổi `startCommand` để tự bọc qua shell:
> ```toml
> startCommand = "sh -c \"uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}\""
> ```
> Commit, push, Railway tự deploy lại → `/healthz` trả 200, healthcheck
> pass. Xác nhận lại toàn bộ bằng các lệnh curl trong `DEPLOYMENT.md`:
> `/healthz` và `/readyz` đều 200, `/chat` không token trả 401 kèm
> `WWW-Authenticate: Bearer`, có token trả 200, và test rate limit 15 lần
> liên tiếp cho ra đúng mẫu `200×10, 429...` khớp với `BUCKET_CAPACITY=10`.
