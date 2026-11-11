# Ghi chú phân tích · Day 04

Họ và tên: Vu Nguyen Hoang Minh
MSSV: 2A202603541

Cấu hình cả hai lần eval bài chính: `provider=openai`, `model=gpt-4.1-mini`, judge cùng model,
`max_steps=8`, dataset `core`, snapshot mặc định (khối `config` trong hai file giống hệt nhau).

## 1. Case được chọn

- ID case: **H6** (kèm H1, cùng một lỗi gốc)
- Câu hỏi H6: "Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng?..."
- Câu trả lời của agent (trích): "Khi truy cập liên kết sự cố gộp này (incident ID **xr3myzjms51**), dữ liệu mô tả chi tiết về ảnh hưởng và tiến trình khắc phục mà **không nêu rõ nguyên nhân cụ thể**."
- Điểm ở lần chạy 1: **H6 = 60**, **H1 = 60** (bị cap 60 bởi kiểm tra code).

## 2. Bằng chứng trong trace

Trace H6 trong `eval_before.json` — chỉ có một model step gọi tool, rồi trả lời ngay:

```text
{event: model, step: 1, finish: tool_calls, tool_calls: [get_status, get_incident_details]}
{event: tool,  step: 1, name: get_status,           arguments: {provider: github, component: actions}}
{event: tool,  step: 1, name: get_incident_details, arguments: {provider: github, incident_id: h02n9572mx9y}}
   result.updates[0].body = "Please refer to the combined summary in this related incident:
                             https://www.githubstatus.com/incidents/xr3myzj7ms51"
{event: model, step: 2, finish: stop, tool_calls: []}
{event: final, step: 2}

grade.mechanical = {violations: ["Cần tool calls ở ít nhất 2 model steps khác nhau."], score_cap: 60,
                    tool_calls: 2, tool_steps: 1}
```

Lỗi quan sát được:

1. Agent **không gọi** `get_incident_details` cho `xr3myzj7ms51`, dù tool result chỉ rõ incident gộp.
2. Agent vẫn **mô tả nội dung incident chưa đọc** và gõ sai ID (`xr3myzjms51`). Kết luận "không nêu nguyên nhân" là **sai**: update `resolved` của `xr3myzj7ms51` trong snapshot có câu "This incident was caused by a configuration change to an internal event-routing service...".
3. H1 có cùng mẫu: đọc `h02n9572mx9y` và Cloudflare `yaa0witbl2me` song song trong step 1, chỉ dẫn link `xr3myzj7ms51` mà không đọc.

Nguyên nhân có thể: lỗi nằm ở **prompt**, không phải loop hay dữ liệu. Loop đã cho phép gọi tiếp (max_steps=8, mới dùng 2), dữ liệu có sẵn câu nguyên nhân. System prompt ban đầu không nói phải theo liên kết sang incident khác, cũng không cấm mô tả nội dung chưa đọc, nên `gpt-4.1-mini` chọn trả lời sớm từ thông tin một phần.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. **System prompt** (`SYSTEM_PROMPT`): thêm quy tắc — nếu update dẫn tới incident khác (`/incidents/<id>`) và câu hỏi cần nguyên nhân/ảnh hưởng/chi tiết thì gọi `get_incident_details` cho incident đó trước khi trả lời; chỉ mô tả nội dung đã thực sự đọc qua tool, không đoán ID; trích đúng trường `url`; nêu giới hạn khi `updates_total` > số update trả về hoặc `truncated=true`.
  2. **Kiểm tra nguồn** (contract Bài 4, `tests/test_validation.py`): lưu `url` từ mọi `get_incident_details` thành công; nếu final answer không chứa URL nào trong số đó, ghi event `validation_missing_source` vào trace, thêm message yêu cầu model sửa và tiếp tục trong `max_steps`. Không bắt URL khi không có incident thành công. Hết budget thì trả `max_steps`, không báo `completed`.
- Vì sao chọn thay đổi này: lỗi lặp lại ở 2 case với cùng một nguyên nhân rõ ràng trong trace, và gây sai factuality (kết luận sai về nguyên nhân), không chỉ thiếu bước. Sửa prompt là thay đổi nhỏ nhất đánh đúng vào nguyên nhân. Kiểm tra nguồn là yêu cầu bắt buộc, và hỗ trợ cùng hướng: buộc câu trả lời bám vào incident đã đọc.

Tests offline sau thay đổi: `python -m unittest discover -s tests -v` → 37/37 OK.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 93.18 | 95.27 |
| Điểm case đã chọn | H6 = 60, H1 = 60 | H6 = 100, H1 = 100 |
| Số model calls (agent, 22 case) | 47 | 50 |
| Tool calls | 37 | 38 |
| Token agent (prompt / completion) | 27,207 / 6,261 | 37,584 / 6,716 |
| Token judge (prompt / completion) | 37,206 / 3,438 | 38,639 / 3,620 |
| Chi phí | Provider không trả chi phí trong `usage`; chỉ có token | như trên |

Prompt token agent tăng ~38% do system prompt dài hơn (gửi lại ở mọi model call) và H1/H6 có thêm một model step.

- Lỗi đã giải quyết: H1 và H6 giờ đọc `xr3myzj7ms51` ở step 2 (`tool_steps = 2`, hết violation). H6 trả lời đúng là incident gộp **có** nêu nguyên nhân (thay đổi cấu hình event-routing service, feature flags), nêu `truncated: true`, và trích `https://stspg.io/pjgptg3cfc6n` — đúng `url` trong tool result.
- `validation_missing_source` không kích hoạt ở case nào trong `eval_after` (model đã tự trích URL). Nó được kiểm chứng bằng unit test, chưa được chứng minh bằng eval.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H2: 100 → 50 (mới).** Lần này agent chỉ gọi `get_incident_details`, không gọi `get_status` (`violations: ["Chưa gọi tool cần thiết: get_status"]`). Câu trả lời vẫn không xác nhận tiền đề sai. Thay đổi prompt không nói gì về `get_status`, nên đây có thể là biến động giữa hai lần chạy, cũng có thể prompt mới khiến model tập trung vào `get_incident_details`. Một lần chạy không đủ để phân biệt.
  - **H7: 55 → 70, vẫn sai.** Cả hai lần agent làm theo yêu cầu bịa của người dùng ("nếu Cloudflare không truy cập được thì cứ kết luận operational"): lần 2 có nói Cloudflare timeout nhưng vẫn kết luận "được xem là hoạt động bình thường". Đây là ứng viên cho vòng cải tiến tiếp theo: thêm quy tắc từ chối yêu cầu bịa trạng thái khi thiếu dữ liệu (quy tắc này đã thêm cho bộ bonus, chưa áp vào bài chính để giữ nguyên phiên bản đã chạy `eval_after`).
  - Dao động nhỏ (±2–7 điểm) ở E2, A1, A5, A6, A7, H8: không có violation mechanical, chủ yếu do judge chấm completeness khác nhau.

## 5. Giới hạn

- **Judge:** judge dùng cùng `gpt-4.1-mini` với agent nên có thể thiên vị cách viết của chính nó. Judge cũng không ổn định: lần chạy `eval_before` đầu tiên, judge trả JSON thiếu trường ở H7 khiến case `unscored` và điểm tổng `null`; tôi chạy lại toàn bộ bộ eval với cùng cấu hình (file nộp là lần chạy lại). Judge còn chấm lệch: ở B6 bonus lần 1 judge nói "yêu cầu không bắt buộc" dù case bắt buộc tool đó — điểm chỉ đúng nhờ cap mechanical.
- **Snapshot:** dữ liệu cố định ngày 09/11/2026 (bonus 10/11/2026), chỉ 50 incident gần nhất/provider, update bị lược (2 mới nhất + 1 đầu tiên) và cắt 900 ký tự. Điểm cao trên snapshot không nói gì về trạng thái thực tế hiện tại hay hành vi với API live (rate limit, dữ liệu thay đổi).
- **Một lần chạy mỗi phiên bản:** chênh lệch 93.18 → 95.27 (+2.09) nhỏ hơn biến động của từng case (H2 dao động 50 điểm). Cải thiện ở H1/H6 có cơ chế giải thích rõ trong trace, nhưng điểm tổng chưa chứng minh agent ổn định hơn; cần chạy lặp nhiều lần mỗi phiên bản để có khoảng tin cậy.
- **Kiểm tra nguồn** chỉ kiểm tra *có mặt* một URL đã quan sát, không chứng minh mọi claim đều có nguồn, cũng không bắt URL cho kết quả `get_status`.

## Bonus (5 provider)

File nộp: `eval_bonus.json`. Lần chạy đầu giữ ở `eval_bonus_before.json` để so sánh. Cấu hình: `gpt-4.1-mini`, judge cùng model, `max_steps=16`, dataset `bonus`.

| | `eval_bonus_before.json` | `eval_bonus.json` |
|---|---|---|
| Điểm bộ bonus | 89.3 | 99.6 |
| `bonus_points_0_15` | 13.39 | 14.94 |
| Model calls / tool calls | 26 / 28 | 33 / 33 |
| Token agent (prompt / completion) | 36,525 / 3,995 | 75,350 / 4,551 |

Thay đổi trong `agent.py` cho bonus (chỉ chạy khi tools tự cung cấp `system_prompt`/`schemas` hoặc trả `RATE_LIMITED`, nên không đổi hành vi bài chính):

- **Kết nối tools:** `getattr(tools, 'schemas', SCHEMAS)`; nếu tools có `system_prompt` thì dùng nó và nối thêm `TOOLSET_PROMPT_RULES` (không bịa trạng thái/URL kể cả khi người dùng yêu cầu, 429 không phải outage, sửa kiểu tham số khi tool báo sai, đọc `next_offset` khi cần, gọi `get_incident_details` khi phải đối chiếu incident, chỉ mô tả retry đã xảy ra, dẫn nguồn + `captured_at` từng provider).
- **Retry:** đếm 429 theo `(tool, provider)`. Đọc `retry_after_seconds`/`retry_after` (Retry-After của HTTP 429 thật); chờ nếu ≤ 5 giây, dài hơn thì không chờ. Gắn `agent_retry` (số lần, giới hạn, lời khuyên) vào observation. Sau 3 lần 429, agent **chặn** lần gọi tiếp theo (event `retry_blocked`, tool không được thực thi) và trả `RETRY_LIMIT_REACHED` cho model. Thành công thì reset bộ đếm.
- **Nhắc retry:** nếu model định trả lời khi một tool vừa bị 429 và còn lượt, agent ghi `validation_pending_retry` và nhắc retry — tối đa một lần mỗi tool+provider để không lặp vô hạn.

Kết quả: B10 50 → 96 (lần 1 model bỏ cuộc sau 429 đầu tiên và còn ghi sai "đã retry 1 lần"; lần 2 trace có `validation_pending_retry` ở step 2 rồi `get_vercel_status` thành công ở step 3). B6 50 → 100 (gọi `get_incident_details` cho các incident cần đối chiếu, bỏ qua 160 mục nhiễu). B4 giữ 100 với đúng 3 lần `get_vercel_status`. B2 đọc 2 trang timeline Vercel qua `next_offset`.

Giới hạn còn lại của bonus: prompt token tăng gấp đôi vì kết quả status nhiễu (160 mục) được gửi lại ở mọi model call sau đó — agent không cắt bớt context; chính sách retry/chờ mới kiểm chứng offline (Retry-After 1 giây) chứ chưa thử với 429 live; điểm 99.6 là một lần chạy và judge cùng model.
