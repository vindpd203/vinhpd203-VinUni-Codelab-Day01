# DANH SÁCH THAY ĐỔI SO VỚI CODE BAN ĐẦU

> Ngày hoàn thành: 2026-11-06
> Trạng thái: **9/9 test pass** (`pytest tests/ -v`)

---

## 1. `starter-code/template.py` — điền code vào các chỗ TODO

### 1.1. Phần imports & khởi tạo đầu file (thêm mới)

| Thay đổi | Lý do |
|----------|-------|
| Thêm `import sys` | Dùng cho fix Unicode bên dưới |
| Thêm `from dotenv import load_dotenv` + gọi `load_dotenv()` | Nạp file `.env` (chứa `GEMINI_API_KEY`) vào biến môi trường khi chạy. Không có dòng này thì Python không tự đọc `.env` dù đã `pip install python-dotenv` |
| Thêm `sys.stdout.reconfigure(encoding="utf-8")` (có guard `hasattr`) | Console Windows mặc định mã hóa cp1252, không in được dấu tiếng Việt → lỗi `'charmap' codec can't encode character 'ệ'` |

### 1.2. Hằng số cấu hình

| Thay đổi | Lý do |
|----------|-------|
| Thêm dòng giá `"gemini-3.5-flash-lite": {"input": 0.30, "output": 2.50}` vào `PRICING_1M_TOKENS` | Model dùng cho chatbot live (giá tham chiếu [OpenRouter](https://openrouter.ai/google/gemini-3.5-flash-lite)) |
| Thêm hằng `CHAT_MODEL = "gemini-3.5-flash-lite"` | `gemini-2.5-flash` (model mặc định của lab) trả lỗi **404 "no longer available to new users"** với tài khoản API mới tạo. Giữ nguyên `GEMINI_MODEL = "gemini-2.5-flash"` để không lệch spec lab (test `compare_models` hardcode giá theo 2.5-flash) |

### 1.3. Tám hàm được triển khai (thay cho `raise NotImplementedError`)

| # | Hàm | Nội dung triển khai |
|---|-----|---------------------|
| 1 | `call_openai` | `openai.OpenAI(...)` → `chat.completions.create(model, messages, temperature, top_p, max_tokens)` → đo `time.time()` → usage lấy từ `response.usage.prompt_tokens` / `completion_tokens`. SDK được import **bên trong hàm** (bắt buộc để mock `@patch("openai.OpenAI")` của test có tác dụng) |
| 2 | `call_gemini` | SDK mới `google-genai`: `genai.Client(api_key=...)` → `GenerateContentConfig(temperature, top_p, max_output_tokens)` → `client.models.generate_content(...)` → usage từ `usage_metadata.prompt_token_count` / `candidates_token_count` |
| 3 | `call_anthropic` | `anthropic.Anthropic(...)` → `messages.create(...)` → text từ `response.content[0].text`, usage từ `response.usage.input_tokens` / `output_tokens` |
| 4 | `compare_models` | Gọi lần lượt gpt-4o, gpt-4o-mini, gemini-2.5-flash cùng prompt; tính cost theo công thức `(input_tokens × rate_in + output_tokens × rate_out) / 1_000_000` từ `PRICING_1M_TOKENS`; trả dict với keys `gpt4o`, `gpt4o_mini`, `gemini_flash` |
| 5 | `streaming_chatbot` | Vòng lặp `input()` → thoát khi gõ `quit`/`exit`; stream bằng `client.models.generate_content_stream()` in từng chunk `flush=True`; lưu history dạng chuỗi `User:`/`Gemini:`, chỉ giữ **6 tin nhắn gần nhất** (`history[-6:]` = 3 lượt hội thoại); dùng `CHAT_MODEL` (model live còn khả dụng) |
| Bonus A | `retry_with_backoff` | Vòng lặp `max_retries + 1` lần: gọi `fn()`, nếu exception thì `sleep(base_delay * 2**attempt)` rồi thử lại; hết lượt thì `raise` exception cuối |
| Bonus B | `batch_compare` | List comprehension: `dict(compare_models(p), prompt=p)` cho từng prompt — thêm key `"prompt"` vào kết quả |
| Bonus C | `format_comparison_table` | Bảng Markdown 6 cột (`Prompt / Model / Response / Latency / Tokens / Cost`), truncate response 50 ký tự, tên hiển thị `GPT-4o`, `GPT-4o-Mini`, `Gemini-Flash` |

> Không hàm nào bị đổi signature — đúng yêu cầu "Do NOT change function signatures".

---

## 2. `solution-code/solution.py` — file mới (toàn bộ folder là mới)

Copy y hệt `template.py`. Đường dẫn này **do file test quy định** (`tests/test_template.py` dòng 23: `SOLUTION_PATH = LAB_DIR / "solution-code" / "solution.py"`) — test ưu tiên load file này nếu tồn tại, không có mới fallback về `template.py`.

---

## 3. `tests/test_template.py` — sửa 1 dòng (bug của test, không phải của bài làm)

```diff
-            def _get_mock():
+            def _get_mock(*args, **kwargs):
```

**Lý do:** mock `side_effect = _get_mock` nhưng hàm khai báo nhận 0 tham số, trong khi `compare_models(prompt)` phải truyền prompt vào → `TypeError` với **mọi** implementation đúng. Sửa để mock chấp nhận đối số — không thay đổi gì phần test đang verify (số kết quả, key `prompt`, nội dung bảng).

---

## 4. Các bước setup môi trường (không đổi code)

1. Tạo venv: `python -m venv .venv` + kích hoạt
2. Cài thư viện: `pip install -r requirements.txt` và `pip install python-dotenv`
3. Tạo `.env` từ mẫu `.env.example` với `GEMINI_API_KEY` (OpenAI/Anthropic chưa có key)

---

## 5. Kiểm chứng

| Kiểm tra | Kết quả |
|----------|---------|
| `pytest tests/ -v` | **9/9 pass** |
| Live `call_gemini` (key thật) | ✅ Trả lời + latency + token usage |
| Live streaming chatbot (Task 5) | ✅ Stream tiếng Việt đúng encoding, `quit` thoát sạch |
| `compare_models` live | ⏸ Chưa chạy được — thiếu `OPENAI_API_KEY` trong `.env` (test mock vẫn pass) |

## Lưu ý khi demo live

- `gemini-2.5-flash` / `2.5-flash-lite` bị Google ngừng cấp cho tài khoản API mới → mọi lời gọi live với Gemini dùng `gemini-3.5-flash-lite` trở lên (xem danh sách bằng `client.models.list()`).
- Muốn chạy đủ `compare_models`: thêm `OPENAI_API_KEY` (và `ANTHROPIC_API_KEY` nếu cần Task 3 live) vào `.env`.
