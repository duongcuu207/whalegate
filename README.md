# Whalegate - Zcode Endpoint 
 
Gateway biến session `chat.deepseek.com` của bạn thành endpoint API chuẩn, hỗ trợ **3 format**:
 
| Format | Endpoint | Dùng cho |
|---|---|---|
| OpenAI Chat Completions | `POST /v1/chat/completions` | OpenAI SDK, Cline, Continue, LiteLLM, dsh |
| Anthropic Messages | `POST /v1/messages` | Claude Code, Anthropic SDK, Cursor |
| OpenAI Responses | `POST /v1/responses` | OpenAI SDK mới (`client.responses`) |
| (phụ) Token estimate | `POST /v1/messages/count_tokens` | Claude Code gọi để đo prompt |
 
Cả ba chạy trên **cùng một lõi**: cùng pool account, failover, stateful conversation, tool calling, vision, DeepThink.
 
> **Cảnh báo:** tự động hoá web app có thể vi phạm ToS của DeepSeek và làm khoá tài khoản. Dùng cho học tập/nghiên cứu trên **tài khoản của chính bạn**, đừng bán hoặc chia sẻ quyền truy cập. Không có bảo hành (MIT).
 
---
 
## 1. Yêu cầu
 
- **Windows / Linux / macOS**
- Muốn build từ source: **Rust 1.75+** (`cargo`)
- Chrome/Chromium (để đăng nhập và lấy session)
- Một tài khoản `chat.deepseek.com` đã đăng nhập
 
---
 
## 2. Build từ source
 
```bash
cd whalegate-main
cargo build --release
```
 
Binary ra `target/release/whalegate.exe` (Windows) hoặc `target/release/whalegate`.
 
Chạy:
 
```bash
./target/release/whalegate.exe serve
```
 
Mặc định nghe `127.0.0.1:8790`. Đổi cổng:
 
```bash
DS_PORT=9000 ./target/release/whalegate.exe serve
```
 
Mở [http://127.0.0.1:8790/](http://127.0.0.1:8790/) để vào dashboard.
 
### Kiểm tra build
 
```bash
cargo test          # 81 test
cargo clippy        # không có cảnh báo ở src/api.rs, src/settings.rs, src/chat.rs
```
 
---
 
## 3. Cài bằng installer (khuyến nghị cho dùng lâu dài)
 
**Linux / macOS**
 
```sh
curl -fsSL https://raw.githubusercontent.com/RuriMeiko/whalegate/main/install.sh | sh
```
 
**Windows (PowerShell)**
 
```powershell
irm https://raw.githubusercontent.com/RuriMeiko/whalegate/main/install.ps1 | iex
```
 
Installer tải binary dựng sẵn, đặt vào `~/.whalegate/bin`, tạo shortcut Startup (Windows) hoặc systemd unit (Linux) để **tự chạy lại sau reboot**, và cấu hình `dsh` nếu có.
 
> Chạy trần bằng shell (như `./whalegate.exe serve`) thì tiến trình **chết là mất**, không ai bật lại. Đây là điểm khác biệt quan trọng — xem mục 8.
 
---
 
## 4. Thêm tài khoản
 
Trên dashboard, nút **+ Thêm**:
 
1. **Đăng nhập qua Chrome** — mở tab Chrome riêng, đăng nhập DeepSeek, gateway tự bắt token + cookie.
2. **Quét tab Chrome** — nếu bạn đã đăng nhập sẵn trong Chrome, quét mọi tab `chat.deepseek.com` và thêm hết.
3. **Dán token tay** — mở tab đã đăng nhập → F12 → Console → `copy(localStorage.userToken)` → dán vào ô.
 
Thêm nhiều account để có pool: gateway chia request theo kiểu ít bận nhất, có cooldown khi lỗi.
 
Nút **Verify** không chỉ kiểm tra token — nó gửi một lượt chat thật tối thiểu, vì account bị throttle vẫn qua được bước auth. Kết quả: `alive` / `throttled` / `auth_dead` / `unknown`.
 
### Token hết hạn
 
Khi DeepSeek trả `INVALID_TOKEN`, account bị đánh dấu **"invalid token — cần login lại"** và bị cách ly ~30 phút. Xử lý: dashboard → **Đăng nhập qua Chrome** (hoặc **Quét tab Chrome**), rồi bấm **Reset** trên account đó.
 
---
 
## 5. Gọi API
 
Tất cả ví dụ giả định cổng `8790` và **chưa đặt API key**.
 
### 5.1 OpenAI Chat Completions
 
```bash
curl http://127.0.0.1:8790/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-web",
    "messages": [{"role": "user", "content": "Giải thích borrow checker trong 3 câu."}],
    "stream": true
  }'
```
 
Python:
 
```python
from openai import OpenAI
 
client = OpenAI(base_url="http://127.0.0.1:8790/v1", api_key="anything")
 
resp = client.chat.completions.create(
    model="deepseek-web",
    reasoning_effort="max",     # none | auto | max
    messages=[{"role": "user", "content": "Chứng minh √2 vô tỉ."}],
)
print(resp.choices[0].message.reasoning_content)   # DeepThink
print(resp.choices[0].message.content)
```
 
Trường mở rộng (truyền qua `extra_body`):
 
| Trường | Kiểu | Mặc định | Ý nghĩa |
|---|---|---|---|
| `reasoning_effort` | string | `DS_DEFAULT_EFFORT` | `none` = Nhanh, `auto` = nghĩ lượt người dùng, `max` = luôn nghĩ |
| `thinking` | bool | `false` | Cách cũ bật DeepThink |
| `search` | bool | `false` | Bật tìm kiếm web của DeepSeek |
| `new_thread` | bool | `false` | Bắt đầu hội thoại web mới thay vì nối tiếp |
 
### 5.2 Anthropic Messages
 
```bash
curl http://127.0.0.1:8790/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-web",
    "max_tokens": 1024,
    "system": "Trả lời ngắn gọn.",
    "messages": [{"role": "user", "content": "Xin chào"}]
  }'
```
 
Python:
 
```python
from anthropic import Anthropic
 
client = Anthropic(base_url="http://127.0.0.1:8790", api_key="anything")
msg = client.messages.create(
    model="deepseek-web",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Xin chào"}],
)
print(msg.content[0].text)
```
 
Đã hỗ trợ: `system` (string hoặc block), `tools[].input_schema`, `tool_use`/`tool_result` trong lịch sử, ảnh (`source.base64` hoặc `source.url`), `thinking: {"type":"enabled"}` (bật DeepThink và trả block `thinking`), streaming (`message_start` → `content_block_*` → `message_delta` → `message_stop`).
 
Bị bỏ qua (web app không có tương đương): `max_tokens`, `temperature`, `top_p`, `top_k`, `metadata`.
 
**Claude Code** — trỏ về gateway:
 
```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:8790
export ANTHROPIC_API_KEY=anything
# nếu đặt DS_API_KEY thì dùng đúng key đó
claude
```
 
### 5.3 OpenAI Responses
 
```bash
curl http://127.0.0.1:8790/v1/responses \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek-web", "input": "Kể ba số nguyên tố.", "stream": true}'
```
 
Python:
 
```python
from openai import OpenAI
client = OpenAI(base_url="http://127.0.0.1:8790/v1", api_key="anything")
r = client.responses.create(model="deepseek-web", input="Kể ba số nguyên tố.")
print(r.output_text)
```
 
Hỗ trợ `instructions`, `input` (string hoặc item array gồm `message`, `function_call`, `function_call_output`), `tools` dạng phẳng, streaming đủ event (`response.created` → `response.output_item.*` → `response.output_text.delta` → `response.completed`).
 
Hai giới hạn **cố ý**:
- `previous_response_id` trả `400` — gateway không lưu response, hãy gửi đủ `input` mỗi lượt.
- Tool dựng sẵn (`web_search`, `file_search`…) bị bỏ vì web app không có tương đương.
 
### 5.4 Endpoint khác
 
```bash
curl http://127.0.0.1:8790/v1/models          # danh sách model + context_length
curl -X POST http://127.0.0.1:8790/v1/messages/count_tokens \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hello"}]}'
```
 
---
 
## 6. Cấu hình trên web
 
Dashboard → **cài đặt**. Nhóm **API** gồm:
 
| Mục | Ý nghĩa |
|---|---|
| **Chat Completions** | Bật/tắt `POST /v1/chat/completions` |
| **Anthropic Messages** | Bật/tắt `POST /v1/messages` (kèm `count_tokens`) |
| **OpenAI Responses** | Bật/tắt `POST /v1/responses` |
| **API key** | Để trống = ai gọi cũng được. Đặt = `/v1/*` yêu cầu `Authorization: Bearer <key>` **hoặc** `x-api-key: <key>` |
| **Tên model** | Tên hiện trong `/v1/models` và trong câu trả lời. Trống = `deepseek-web` |
 
Format bị tắt trả `404`, giống route không tồn tại. Mọi thay đổi có hiệu lực **ngay**, không cần restart.
 
Mỗi endpoint đang bật có nút copy URL đầy đủ.
 
Nhóm khác:
 
| Nhóm | Mục đáng chú ý |
|---|---|
| Hội thoại | **Stateful** — nối tiếp phiên web, chỉ gửi phần mới (nhanh và rẻ hơn nhiều cho agent loop) |
| Tốc độ | **Mức suy nghĩ mặc định** (`none`/`auto`/`max`), **Trần prompt** (vượt thì cắt tool_result cũ rồi bỏ lượt cũ) |
| Pool | **Đồng thời mỗi tài khoản** (mặc định 2 — cao hơn dễ bị giới hạn) |
| Mạng | **Múi giờ** (giây lệch UTC, +7 = 25200) |
 
Mục đánh dấu **cần restart** sẽ hiện hộp thoại hỏi khởi động lại sau khi lưu (chờ hết request rồi mới restart).
 
### Biến môi trường
 
Thứ tự ưu tiên: **giá trị lưu trên dashboard → biến môi trường → mặc định**.
 
| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `DS_PORT` | `8790` | Cổng (luôn bind `127.0.0.1`) |
| `DS_API_KEY` | – | Khoá `/v1/*` |
| `DS_ACCOUNTS_PATH` | `~/.whalegate/accounts.json` | File account. Settings/usage/samples nằm cạnh đó |
| `DS_DEFAULT_EFFORT` | `none` | Mức suy nghĩ mặc định |
| `DS_CONTEXT_TOKENS` | `500000` | `context_length` báo cho client |
| `DS_PROMPT_MAX_CHARS` | `1200000` | Trần prompt |
| `DS_MAX_CONC_PER_ACC` | `2` | Request đồng thời mỗi account |
| `DS_STATEFUL` | `1` | Nối tiếp phiên web |
| `DS_QUEUE_WAIT_SECS` | `120` | Chờ account rảnh trước khi trả `429` |
 
Danh sách đầy đủ: xem `README.md` mục **Configuration**.
 
---
 
## 7. Lỗi thường gặp
 
**`{"error":"no account configured"}` / `503`**
Chưa thêm account, hoặc account bị tắt trong pool. Dashboard → **+ Thêm**.
 
**`{"error":{"message":"thiếu hoặc sai API key ..."}}` / `401`**
Có đặt API key. Gửi kèm `-H "x-api-key: <key>"` hoặc `-H "Authorization: Bearer <key>"`.
 
**`404` ở `/v1/messages` hoặc `/v1/responses`**
Format đó đang tắt trong **Cài đặt → API**.
 
**`429 pool exhausted`**
Mọi account đang bận/cooldown. Chờ vài giây (response có `Retry-After`), hoặc thêm account.
 
**Request treo rất lâu rồi mới lỗi**
Account bị cooldown nhưng vẫn trong pool → gateway xếp hàng chờ. Kiểm tra dashboard: nếu thấy `invalid token — cần login lại` thì phải đăng nhập lại (mục 4).
 
**Account báo `invalid token`, log có `AUTH_INVALID: INVALID_TOKEN`**
Token DeepSeek hết hạn. Đăng nhập lại qua Chrome trên dashboard, rồi **Reset**.
 
**`chỉ cho phép từ localhost` / `403`**
`/` và `/api/*` chỉ nhận loopback. Đừng proxy dashboard ra ngoài. `/v1/*` thì mở cho mọi nguồn — nên đặt API key trước khi expose.
 
**`không mở được cổng 8790` (os error 10048)**
Đã có tiến trình khác giữ cổng. Tìm và tắt:
 
```bash
netstat -ano | grep ":8790"        # lấy PID
taskkill //F //PID <pid>           # Windows
```
 
hoặc đổi cổng bằng `DS_PORT`.
 
**Server tự tắt, log có `[restart] đang khởi động lại…`**
Ai đó gọi `/api/restart` hoặc `/api/update`. Trên Windows, gateway `exit(0)` và trông cậy vào script/supervisor bên ngoài để bật lại. Chạy trần bằng shell thì nó **thoát luôn** — xem mục 8.
 
---
 
## 8. Chạy nền lâu dài
 
Chạy `./whalegate serve` trong terminal chỉ sống đến khi terminal đóng. Ba cách bền:
 
### Windows — Task Scheduler
 
```powershell
$exe = "D:\GSVN\whalegate-main\target\release\whalegate.exe"
schtasks /Create /TN "whalegate" /SC ONLOGON /RL LIMITED /TR "`"$exe`" serve"
schtasks /Run /TN "whalegate"
```
 
Xoá: `schtasks /Delete /TN "whalegate" /F`
 
### Windows — supervisor bằng PowerShell (đơn giản nhất)
 
```powershell
while ($true) {
  & "D:\GSVN\whalegate-main\target\release\whalegate.exe" serve
  Start-Sleep -Seconds 2
}
```
 
Vòng này tự bật lại sau mỗi lần tự-restart/self-update.
 
### Linux — systemd
 
```bash
sudo cp deploy/whalegate.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now whalegate
```
 
Hoặc dùng `install.sh` — nó làm sẵn bước này.
 
---
 
## 9. Cấu trúc project
 
```text
src/
├── main.rs            CLI: serve | whoami | chat | chatraw | setup-dsh | version
├── chat.rs            lõi chat (run_chat + start_core) — pool, failover, stateful, tool-call
├── api.rs             /v1/messages (Anthropic) + /v1/responses (OpenAI) trên cùng lõi
├── server.rs          router, quản lý account, dashboard API, parser DSML
├── client.rs          HTTP + SSE client tới chat.deepseek.com
├── settings.rs        cấu hình dashboard (dashboard → env → default)
├── usage.rs           đếm token, giá API, latency
├── pow.rs             solver DeepSeekHashV1
├── chrome_capture.rs  CDP login, bắt tab, refresh cookie
├── stream_guard.rs    chặn markup tool-call rò ra client
└── ui.html            dashboard (nhúng vào binary)
```
 
File dữ liệu (không nằm trong zip): `~/.whalegate/accounts.json`, `settings.json`, `usage.json`, `samples.jsonl`.
 
---
 
## 10. Ghi chú về bản này
 
- Thêm `src/api.rs` (779 dòng) với Anthropic Messages + OpenAI Responses + `count_tokens`.
- `chat.rs` tách `start_core()`, để mọi format dùng chung một đường xử lý.
- `settings.rs` thêm kind `Text` (cho API key / tên model) và nhóm `API`.
- `ui.html` viết đầy đủ panel Cài đặt (`loadCfg`/`closeSettings` trước đây chưa tồn tại nên modal hỏng).
- API key nhận cả hai kiểu header; file `settings.json` được đặt quyền `0600` trên Unix.
- `previous_response_id` và built-in tools của Responses bị từ chối/bỏ có chủ đích — ghi rõ trong README.
 
