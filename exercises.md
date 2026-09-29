# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Gia Thành  Mã học viên: 2A202602626

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên khai báo `AGENT_API_KEY` thì `Settings` sẽ báo lỗi validation và container không khởi động. Nhờ vậy tôi thấy lỗi ngay trong log deploy và bổ sung biến trong dashboard trước khi service có URL công khai. Nếu dùng mặc định `"changeme"`, service vẫn chạy và bất kỳ ai biết khóa mặc định đều có thể gọi `/ask`, làm lộ quyền sử dụng và tiêu ngân sách trước khi tôi phát hiện cấu hình sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON của sự kiện hoàn tất request có dạng: `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T04:57:17+00:00","user_id":"sv-test","tokens_in":94,"tokens_out":47,"cost_usd":0.0000423}`. Từ log có cấu trúc này tôi có thể (1) lọc/đếm tất cả request của một `user_id` hoặc các event `ask_completed` trong dashboard log; (2) tính tổng chi phí hay phát cảnh báo khi `cost_usd` hoặc số token tăng bất thường. Chuỗi `print("đã trả lời xong")` không có trường dữ liệu nhất quán để máy lọc, tổng hợp hay cảnh báo.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Kết quả build thực tế cho thấy image một stage là 1.73 GB, còn image multi-stage là 271 MB, giảm khoảng 1.46 GB. Image một stage dùng base image Python đầy đủ và chứa toàn bộ build context cùng các thành phần không cần cho runtime. Bản multi-stage dùng `python:3.11-slim` ở runtime, chỉ chép package đã cài cùng `app/` và `utils/`; vì vậy các công cụ build, cache/tệp tạm và phần không cần để chạy production không đi vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa một ký tự trong `app/main.py`, các layer từ `FROM`, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` vẫn dùng cache vì `requirements.txt` không đổi. Runtime cũng dùng lại các layer tạo user và copy package từ builder; chỉ `COPY app ./app` (và các layer sau nó như `USER`, metadata, `HEALTHCHECK`, `CMD`) được tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source làm invalid layer `COPY . .`, khiến `pip install` chạy lại dù dependency không đổi; build sẽ chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng như thực thi lệnh do xử lý input không an toàn có thể cho kẻ tấn công chạy lệnh trong container. Nếu process là root, lệnh đó có toàn quyền root trong container: đọc/ghi file, cài công cụ và tìm credential hoặc socket Docker bị mount nhầm. Từ một cấu hình host/container yếu, các quyền này có thể bị dùng để thoát container hoặc điều khiển tài nguyên host. `USER appuser` làm process ứng dụng và lệnh bị khai thác chỉ có quyền của user thường; nó chặn bước leo quyền root trong container, giảm đáng kể hậu quả dù không thay thế việc vá lỗ hổng và cấu hình host an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request. Người dùng gửi 10 request ngay trước mốc đổi phút, ví dụ 10:00:59, rồi gửi tiếp 10 request ngay sau mốc 10:01:00. Bộ đếm theo phút lịch reset ở giây 00 nên chấp nhận cả hai nhóm, dù chỉ cách nhau khoảng 2 giây. Sliding window đếm 60 giây gần nhất nên sau 10 request đầu, 10 request kế tiếp sẽ bị chặn cho đến khi request cũ ra khỏi cửa sổ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong cửa sổ 60 giây để chống burst và bảo vệ năng lực xử lý; cost guard theo dõi tổng chi phí của từng user trong tháng để không vượt ngân sách USD. Ví dụ user chỉ gửi 1 request/phút nên qua rate limit, nhưng đã dùng gần hết 10 USD và request kế tiếp làm tổng chi tiêu vượt ngân sách: cost guard trả 402. Chiều ngược lại, user mới chưa tốn tiền nên còn ngân sách và cost guard cho qua, nhưng đã gửi 10 request trong 60 giây: rate limiter trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> (1) Redis mất kết nối, endpoint gộp trả trạng thái không khỏe cho cả ba container. (2) Orchestrator hiểu nhầm process của cả ba container đều chết và lần lượt restart hoặc thay thế chúng. (3) Trong thời gian Redis vẫn lỗi 30 giây, container mới cũng fail health check, nên vòng restart tiếp tục. (4) Khi Redis trở lại, cụm vừa phải khởi động lại vừa có thể mất request đang xử lý; đây là sự cố dây chuyền không cần thiết. Tách `/health` chỉ kiểm tra process còn sống, còn `/ready` báo dependency chưa sẵn sàng để load balancer ngừng gửi traffic mà không restart cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi lần request vào bất kỳ replica nào đều đọc/ghi cùng key `history:<user_id>`, nên `history_length` tăng liên tục theo toàn bộ cuộc hội thoại (đến giới hạn lưu 20 message), không phụ thuộc container nhận request. Nếu dùng dict Python trong RAM, mỗi replica có một dict riêng: request bị phân phối sang A, B, C sẽ làm `history_length` nhảy về các giá trị riêng của từng replica, có lúc 0/2 rồi lại tăng; khi replica restart thì lịch sử trong nó mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi đưa service lên Railway, tôi kiểm tra `/ready` để xác nhận dependency thay vì chỉ thấy deploy “Online”. Nếu `REDIS_URL` chưa được gắn đúng vào service agent, endpoint này sẽ trả `503` với `{"status":"not ready","redis":false}`. Tôi kiểm tra tab Variables và log của service để đối chiếu biến môi trường, sau đó tạo/gắn Redis service của Railway vào agent và redeploy. Kết quả kiểm tra sau cùng là `/ready` trả `200` với `{"status":"ready","redis":true}`; `/health` vẫn độc lập với Redis và trả `200`.
