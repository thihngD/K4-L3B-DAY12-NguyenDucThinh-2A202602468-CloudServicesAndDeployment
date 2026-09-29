# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder in nghiêng dưới mỗi câu hỏi bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Thịnh  Mã học viên: 2A202602468

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử lúc mình deploy lên Railway mà quên set biến `AGENT_API_KEY` trong
> dashboard. Nếu code có giá trị mặc định kiểu `"changeme"`, app vẫn khởi
> động bình thường, `/health` vẫn xanh, nhìn vào tưởng mọi thứ ổn. Nhưng thực
> ra lúc đó ai cũng gọi được `/ask` bằng đúng cái khóa "changeme" ghi sẵn
> trong code (mà code thì public trên GitHub), thế là người lạ dùng free mock
> LLM của mình thoải mái mà mình không biết, chỉ tới lúc xem log hoặc hết
> ngân sách mới phát hiện. Còn vì mình để `agent_api_key: str` không có mặc
> định, lúc quên set biến thì Railway build xong, container khởi động là
> crash ngay với lỗi ValidationError, log ghi rõ thiếu field nào. Mình thấy
> ngay app "chết" trên dashboard, vào set lại biến là xong — phát hiện lỗi
> trong vài giây thay vì để lọt ra ngoài mà không hay biết.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Đây là dòng log thật lấy từ `docker compose logs agent` lúc mình gọi `/ask`:
>
> ```
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:07:34.228194+00:00", "user_id": "sv01", "tokens_in": 1, "tokens_out": 40, "cost_usd": 2.415e-05}
> ```
>
> Hai việc mình làm được với dòng này mà `print("đã trả lời xong")` chịu thua:
>
> 1. **Cộng dồn chi phí theo user**: vì có key `user_id` và `cost_usd` sẵn ở
>    dạng số, mình có thể viết một câu lệnh kiểu `grep ask_completed log.txt |
>    jq '.cost_usd' | awk '{s+=$1} END {print s}'` để cộng ra tổng tiền đã
>    tiêu trong ngày, hoặc lọc riêng theo từng `user_id`. Print thường thì chữ
>    dính vào nhau, muốn lấy số ra phải tự viết regex đoán mò, dễ sai.
> 2. **Cắm vào hệ thống cảnh báo**: vì mỗi dòng là một JSON object độc lập,
>    Datadog/Grafana hay kể cả một script Python đơn giản đọc theo dòng, parse
>    `json.loads()` là ra ngay dict, từ đó dựng biểu đồ "số token dùng theo
>    giờ" hoặc bắn cảnh báo khi `cost_usd` một request vượt ngưỡng. Print chữ
>    thường không đảm bảo format ổn định, máy đọc dễ vỡ khi mình đổi câu chữ.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình build thật cả 2 bản bằng `docker images`, số đo ở trên là số thật.
> Chênh nhau khoảng 1.46GB. Lý do: bản 1 stage dùng `FROM python:3.11` — đây
> là bản đầy đủ, có kèm compiler (gcc), header file để build các gói Python
> cần biên dịch, với rất nhiều thư viện hệ thống không dùng lúc chạy app,
> chỉ cần lúc `pip install`. Bản multi-stage tách làm 2 tầng: tầng `builder`
> dùng `python:3.11-slim` để cài đặt (`pip install --user`), sau đó tầng
> `runtime` (cũng slim) chỉ COPY đúng thư mục `~/.local` chứa các gói đã cài
> xong sang, không mang theo compiler hay cache của pip. Nói đơn giản: bản 1
> stage giữ luôn "xưởng lắp ráp" (biên dịch) trong thành phẩm cuối cùng, còn
> multi-stage chỉ lấy "sản phẩm" ra khỏi xưởng rồi vứt xưởng đi, nên nhẹ hơn
> nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình đổi số version trong `app/main.py` (1.0.0 → 1.0.1) rồi build lại. Với
> Dockerfile hiện tại (COPY requirements.txt + pip install nằm ở stage
> `builder`, code copy sau ở stage `runtime`), kết quả build ghi rõ: tất cả
> layer của `builder` (WORKDIR, tạo user, COPY requirements.txt, RUN pip
> install) đều là `CACHED`, chỉ có layer cuối `COPY --chown=app:app . .` là
> chạy lại (mất 0 giây vì code nhẹ). Tức là sửa code không đụng gì tới việc
> cài thư viện.
>
> Sau đó mình thử dựng một bản Dockerfile "sai thứ tự" — `COPY . .` đặt
> trước `RUN pip install`, rồi lại đổi version một lần nữa và build lại bản
> đó. Lần này log build cho thấy `COPY . .` chạy lại (vì code đổi), và vì
> layer `pip install` đứng NGAY SAU nó nên Docker coi input của layer đó đã
> đổi theo, thế là toàn bộ `pip install` chạy lại từ đầu — tải lại và cài lại
> gần 30 gói, mất khoảng 21 giây thay vì 0 giây (cache hit). Vậy nên đặt
> `COPY requirements.txt` + `pip install` trước khi `COPY . .` là quan trọng:
> chỉ cần sửa 1 dòng code, đặt sai thứ tự là mỗi lần build phải cài lại toàn
> bộ thư viện, rất phí thời gian nếu lặp lại nhiều lần trong ngày.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện mình hình dung: (1) code Python của mình có lỗ hổng, ví dụ
> một thư viện third-party bị dính lỗi cho phép chạy lệnh tùy ý (remote code
> execution) — chuyện này xảy ra thật ngoài đời, không phải giả định xa vời.
> (2) Kẻ tấn công khai thác lỗ hổng đó, chạy được lệnh shell bên trong
> container. (3) Nếu process đang chạy bằng root, lệnh shell đó cũng chạy
> với quyền root — họ có thể đọc/ghi bất cứ file nào trong container, cài
> thêm phần mềm độc hại, hoặc quan trọng nhất là tìm cách "thoát" ra khỏi
> container (container escape) qua lỗ hổng của chính Docker/kernel. (4) Nếu
> thoát được, vì họ đang có quyền root bên trong, họ có thể trở thành root
> luôn trên máy host thật — lúc đó không chỉ app của mình bị chiếm mà cả máy
> chủ (và các container khác chạy trên đó) đều gặp nguy hiểm.
>
> Lệnh `USER app` cắt đứt chuỗi này ngay ở bước (3): dù kẻ tấn công có chạy
> được lệnh shell trong container, họ chỉ có quyền của user thường (`app`),
> không đọc/ghi được file hệ thống, không cài được package, và các lỗ hổng
> container escape thường yêu cầu quyền root bên trong mới khai thác được.
> Nói cách khác, `USER` không ngăn được lỗ hổng xảy ra, nhưng giới hạn thiệt
> hại nếu nó xảy ra — kẻ tấn công bị nhốt trong một "phòng" ít quyền hơn rất
> nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách làm: canh đúng lúc 10:00:59 (giây thứ
> 59 của phút hiện tại) gửi liền 10 request — vẫn hợp lệ vì bộ đếm của phút
> 10:00 chưa đầy. Chờ đúng 2 giây sau, tới 10:01:01 (phút mới đã bắt đầu,
> bộ đếm reset về 0) gửi tiếp 10 request nữa — cũng hợp lệ vì đang tính cho
> phút 10:01. Vậy trong đúng 2 giây đồng hồ thực tế, người dùng gửi được 20
> request mà hệ thống đếm-theo-phút vẫn thấy "đúng luật" ở cả hai phía. Đây
> chính là lỗ hổng mà sliding window (đếm 60 giây gần nhất tính từ thời điểm
> hiện tại, không quan tâm mốc phút tròn) tránh được — với sliding window,
> 20 request đó đều nằm trong cùng một cửa sổ 60 giây nên request thứ 11 sẽ
> bị chặn ngay.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số lượng request** trong một khoảng thời gian, không quan
> tâm mỗi request "nặng" hay "nhẹ". Cost guard đếm **số tiền** đã tiêu trong
> tháng, không quan tâm request đó có tần suất ra sao.
>
> Tình huống rate limit cho qua nhưng cost guard phải chặn: user chỉ gửi 2
> request trong phút này (còn xa hạn mức 10/phút), nhưng mỗi câu hỏi rất dài
> (gần 2000 ký tự, kịch trần) khiến token dùng nhiều, tốn gần hết ngân sách
> tháng chỉ với vài request. Rate limit thấy "mới 2 request, cho qua", nhưng
> cost guard thấy "tiền sắp hết" nên chặn ở request tiếp theo dù tần suất
> chưa cao.
>
> Tình huống ngược lại — cost guard cho qua nhưng rate limit phải chặn: user
> còn dư rất nhiều ngân sách (mới dùng 0.5$/10$ tháng), nhưng lại viết một
> vòng lặp gửi 50 request/giây với câu hỏi ngắn xíu (tốn ít tiền mỗi lần).
> Cost guard thấy "còn tiền, cho qua", nhưng rate limit đếm số request trong
> 60 giây gần nhất đã vượt 10 nên chặn ngay từ request thứ 11, bất kể tiền
> còn nhiều hay ít.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện mình hình dung:
>
> 1. Redis rớt kết nối. Cả 3 container `agent` cùng lúc gọi tới Redis đều
>    thất bại.
> 2. Vì `/health` (lúc này đã gộp chung với check Redis) trả về 503 cho cả 3
>    container, orchestrator (Docker/Railway/K8s) đọc `/health` làm liveness
>    probe, thấy 503 liên tục thì nghĩ là "process đã chết", không phải "đang
>    chờ dependency" — nó sẽ **restart cả 3 container** cùng lúc.
> 3. Container restart xong, khởi động lại app, nhưng Redis vẫn chưa sống lại
>    (mới rớt 30 giây, có thể vẫn đang trong lúc mất kết nối), nên `/health`
>    lại tiếp tục 503 → orchestrator lại restart tiếp → vòng lặp restart liên
>    tục (crash loop) trong suốt 30 giây đó.
> 4. Kết quả tệ nhất: đúng lúc cần cụm ổn định nhất để chờ Redis hồi phục thì
>    cả 3 container lại liên tục khởi động lại, tốn thời gian warm-up, có thể
>    làm rớt các request đang xử lý dở, và khi Redis sống lại chưa chắc cả 3
>    container đã "healthy" đúng lúc.
>
> Trong khi đó nếu tách riêng: `/health` không đụng Redis nên vẫn báo "sống"
> suốt 30 giây, orchestrator không restart container (đúng bản chất — process
> vẫn sống, chỉ có dependency ngoài đang lỗi). Chỉ có `/ready` báo 503, khiến
> load balancer tạm ngừng đẩy traffic mới vào, nhưng container vẫn giữ
> nguyên, sẵn sàng nhận traffic lại ngay khi `/ready` xanh trở lại mà không
> cần khởi động lại từ đầu.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy thật: bật thêm Nginx làm load balancer (theo cấu hình có sẵn ở
> `nginx/nginx.conf`), scale `agent` lên 3, rồi gọi `/ask` 5 lần liên tiếp
> với cùng `X-User-Id: sv01` qua cổng Nginx. Xem log từng container thì thấy
> request bị chia ra: agent-1 xử lý 2 lần, agent-2 xử lý 1 lần, agent-3 xử lý
> 2 lần — đúng như dự đoán, mỗi request có thể rơi vào container khác nhau.
> Nhưng `history_length` trong response vẫn tăng đều đặn: 0 → 2 → 4 → 6 → 8
> (mỗi lượt hỏi ghi 2 dòng: câu hỏi + câu trả lời), không hề bị reset hay
> nhảy lung tung, vì lịch sử nằm ở Redis — nơi cả 3 container cùng đọc/ghi
> chung.
>
> Nếu lịch sử lưu trong một dict Python bình thường (trong RAM của từng
> container) thì mỗi container sẽ có bộ nhớ riêng, không container nào thấy
> dữ liệu của container khác. Lúc đó `history_length` sẽ không tăng đều mà
> nhảy loạn: ví dụ agent-1 xử lý request 1 thấy history_length=0, xong tới
> request 2 rơi vào agent-2 (dict rỗng, chưa có gì) thì lại thấy
> history_length=0 (không phải 2), rồi request 3 quay lại agent-1 thì lại
> thấy 2 (chỉ tính phần agent-1 tự nhớ). Người dùng sẽ cảm giác agent "mất
> trí nhớ" liên tục dù họ vẫn đang hỏi cùng một cuộc trò chuyện.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi mình gặp: sau khi deploy lên Railway và tạo domain public, gọi
> `curl <url>/health` thì bị "Failed to connect... Could not connect to
> server" (kết nối bị từ chối/không phản hồi), dù logs cho thấy app đã
> khởi động thành công ("Application startup complete").
>
> Cách tìm nguyên nhân: mình xem log của service bằng `railway logs`, thấy
> dòng "Uvicorn running on http://0.0.0.0:8080" — tức là app đang lắng nghe
> ở cổng **8080**. Nhưng lúc tạo domain public, mình gõ lệnh
> `railway domain --port 8000` — nghĩa là domain lại đang trỏ traffic vào
> cổng **8000**. Hai cổng lệch nhau nên request từ ngoài vào domain không
> bao giờ chạm được tới app đang lắng nghe ở cổng khác. Nguyên nhân gốc:
> Railway tự động gán một giá trị `$PORT` ngẫu nhiên (8080) cho container
> lúc mình chưa set biến này tường minh, còn domain thì mình lại tạo cố định
> theo cổng 8000 mà Dockerfile khai báo mặc định.
>
> Cách sửa: set tường minh biến môi trường `PORT=8000` cho service trên
> Railway (khớp với domain đã tạo và với `EXPOSE 8000` trong Dockerfile), rồi
> trigger deploy lại. Sau khi redeploy, log cho thấy app chuyển sang lắng
> nghe đúng cổng 8000, gọi `/health` từ domain public trả về 200 bình
> thường.
