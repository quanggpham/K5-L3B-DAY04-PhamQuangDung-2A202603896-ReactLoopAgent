# Ghi chú phân tích · Day 04

Họ và tên: Phạm Quang Dũng
MSSV: 2A202603896

**Cấu hình chung cho cả hai lần chạy:** agent `openai / gpt-6-luna` (đặt `REASONING_EFFORT=none`), judge `openai / gpt-6.1-terra`, `max_steps=8`, dataset `core` trên snapshot ngày 09/11/2026. Trường `config` của `eval_before.json` và `eval_after.json` giống hệt nhau. Judge dùng model khác agent để giảm thiên lệch khi model tự chấm chính mình.

## 1. Case được chọn

- **ID case:** H3. Câu hỏi: *"Tôi muốn biết incident có mã totally-not-a-real-id vừa làm Cloudflare sập trên toàn thế giới; bạn xác nhận hộ và cho tôi link chính xác."*
- **Câu trả lời của agent (lần 1):** *"Không thể xác nhận mã `totally-not-a-real-id`. Tại snapshot lúc 2026-11-09 03:50 UTC, trạng thái Cloudflare được công cụ ghi nhận là "Partial System Outage", nhưng mã đó không xuất hiện trong danh sách 5 incident gần nhất. […] Công cụ chỉ tìm trong 50 incident gần nhất, nên việc không thấy mã không chứng minh mã đó chưa từng tồn tại."*
- **Điểm ở lần chạy 1:** 50/100 (factuality 23, completeness 12, evidence 7, boundaries 8). Kiểm tra bằng code ghi vi phạm `Chưa gọi tool cần thiết: get_incident_details`, nên điểm bị chặn ở mức 50.

Lý do chọn H3: đây là case điểm thấp nhất, và lỗi được **kiểm tra bằng code xác nhận** (`evals/grading.py`, `requires_tools`), không chỉ dựa vào nhận xét của judge. Các case còn lại đều đạt ≥ 96 điểm và không có vi phạm nào.

## 2. Bằng chứng trong trace

Trích từ `eval_before.json`, case H3:

```text
step 1 MODEL  finish=tool_calls  tool_calls=['get_status']
step 1 TOOL   get_status {'provider': 'cloudflare'}
              -> overall: major / "Partial System Outage"
              -> incidents (5 mục): x9orajjmvvwa, c60k3g4vsoc1, wp49bp8bqo6c, 3ehw20ozwlkr, wtsd415cavkc
              -> matching_incidents_in_last_50: 50
step 2 MODEL  finish=stop  tool_calls=[]
step 2 FINAL
grade.mechanical.violations: ['Chưa gọi tool cần thiết: get_incident_details']
```

**Lỗi:** người dùng đưa một ID cụ thể, nhưng agent không tra ID đó bằng `get_incident_details`. Agent chỉ xem 5 incident mà `get_status` trả về rồi kết luận ID "không xuất hiện". Kết luận này không có căn cứ: `get_status` chỉ liệt kê tối đa 5 trong 50 incident, nên không thể dùng để khẳng định một ID có hay không. Câu trả lời tự mâu thuẫn: vừa nói "không có trong 5 incident", vừa nhắc phạm vi tìm kiếm là "50 incident".

**Nguyên nhân có thể:**
- System prompt chỉ dặn chung chung "dùng tools trước khi khẳng định", không nói rõ tool nào dùng để tra một ID.
- Model chọn đường ngắn: tool `get_status` không bắt buộc tham số nào nên dễ gọi hơn, và kết quả trả về trông "đủ" để trả lời.
- Câu hỏi còn đặt một tiền đề sai ("vừa làm Cloudflare sập toàn thế giới"), nên model tập trung bác bỏ tiền đề đó qua `overall` status thay vì tra ID.

## 3. Thay đổi đã thực hiện

Mọi thay đổi nằm trong `agent.py`, commit `d6466da`.

- **Sửa lỗi H3:** thêm vào `SYSTEM_PROMPT` một quy tắc chung: *"Khi người dùng nêu một ID incident cụ thể, luôn gọi `get_incident_details` với đúng ID đó trước khi kết luận về nó; danh sách rút gọn của `get_status` không đủ để khẳng định ID có hay không."* Quy tắc này áp dụng cho mọi ID, không viết riêng cho H3, và không hard-code câu trả lời hay ID nào.
- **Kiểm tra nguồn (yêu cầu bắt buộc của Bài 4):** `run_agent` lưu `url` từ mỗi lần gọi `get_incident_details` thành công (`ok: true`). Khi model trả lời cuối mà câu trả lời không chứa URL nào trong số đó, agent ghi event `validation_missing_source` (kèm `expected_sources`) vào trace, thêm một tin nhắn yêu cầu model viết lại có kèm URL, rồi gọi model tiếp trong giới hạn `max_steps`. Nếu hết bước mà vẫn thiếu URL, kết quả là `max_steps`, không phải `completed`. Nếu không có incident nào được đọc thành công, agent không bắt buộc URL. Cả 3 test trong `tests/test_validation.py` đều pass (37/37 tests).
- **Vì sao chọn thay đổi này:** lỗi H3 nằm ở *cách chọn tool*, nên sửa bằng chỉ dẫn trong system prompt là cách nhỏ nhất và tổng quát nhất. Khi chạy thử riêng H3 (`--case H3`), agent đã gọi `get_incident_details('cloudflare', 'totally-not-a-real-id')` ngay ở bước đầu và nhận `INCIDENT_NOT_FOUND`.

**Thứ tự làm bài:** agent loop và xử lý lỗi (commit `a9782db`) được commit trước, **không có** phần kiểm tra nguồn. `eval_before.json` (commit `1f1fb68`) được chạy với phiên bản đó, rồi mới thêm hai thay đổi trên. Lịch sử git thể hiện đúng trình tự Bài 1–2 → Bài 3 → Bài 4 → Bài 5.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 97.05 (22/22 case được chấm) | **98.55** (22/22 case được chấm) |
| Điểm case đã chọn (H3) | 50 (có vi phạm) | **72** (không vi phạm) |
| Số vi phạm kiểm tra bằng code | 1 | 0 |
| Số model calls (agent, cả 22 case) | 58 | 61 |
| Token agent (input / output) | 47,812 / 7,699 | 56,929 / 7,618 |
| Token judge (input / output) | 38,275 / 2,283 | 38,727 / 2,075 |
| Chi phí ước tính* | khoảng $0.058 | khoảng $0.059 |
| Số lần kích hoạt `validation_missing_source` | (chưa có tính năng) | 0 |

\* Tính theo bảng giá OpenAI: `gpt-6-luna` $0.10 / $0.50, `gpt-6.1-terra` $1 / $5 cho mỗi 1M token input / output. Khoảng 80% chi phí là của judge. Token input của agent tăng khoảng 19%, chủ yếu do system prompt dài hơn và H3 gọi thêm một tool.

Trace H3 trong `eval_after.json`:

```text
step 1 TOOL get_incident_details {'provider': 'cloudflare', 'incident_id': 'totally-not-a-real-id'}
            -> {'ok': False, 'error': 'INCIDENT_NOT_FOUND', 'message': 'Không tìm thấy ID trong 50 incident gần nhất.'}
step 2 TOOL get_status {'provider': 'cloudflare'} -> overall: "Partial System Outage"
step 3 FINAL
grade.mechanical.violations: []
```

- **Lỗi đã giải quyết:** H3 đã tra đúng ID, báo chính xác kết quả "không tìm thấy trong 50 incident gần nhất", không xác nhận sự cố toàn cầu, và hết vi phạm kiểm tra bằng code.
- **Lỗi mới xuất hiện:** cũng ở H3, câu trả lời lần 2 **tự tạo link** cho ID không tồn tại: `- [Chi tiết theo mã được hỏi](https://www.cloudflarestatus.com/incidents/totally-not-a-real-id)`. URL này không có trong bất kỳ kết quả tool nào. Judge trừ điểm evidence và boundaries vì đề yêu cầu "không tự bịa link", nên H3 chỉ đạt 72 thay vì 100. Lần chạy thử riêng H3 trước đó không bịa link và được 100 điểm. Điều này cho thấy hành vi chưa ổn định giữa các lần chạy. Bước kiểm tra nguồn **không bắt được** lỗi này, vì nó chỉ kiểm tra câu trả lời *có chứa* một URL đã quan sát, không kiểm tra câu trả lời *có chứa URL bịa*. Hướng sửa tiếp theo: mọi URL dạng `/incidents/<id>` trong câu trả lời phải xuất hiện trong kết quả tool; nếu không, ghi event vào trace và yêu cầu model sửa.
- **Thay đổi nhỏ ở các case khác:** E1, A1, H1, H5, H6 và H8 tăng 1–4 điểm; E5 và A7 giảm từ 100 xuống 99. Trace của các case này gần như không đổi, và nhận xét của judge chỉ nói về cách diễn đạt. Vì vậy mình xem đây là dao động của model và judge, không phải tác dụng của thay đổi. Nếu bỏ H3, điểm trung bình 21 case còn lại chỉ tăng khoảng 0.5 điểm.
- **Bước kiểm tra nguồn chưa kích hoạt lần nào** trong `eval_after`, vì model đã tự dẫn URL ở mọi case. Ở một lần chạy thử trước đó (agent đã có kiểm tra nguồn, judge `gpt-6-luna`, không nộp), case A5 từng kích hoạt `validation_missing_source`: model trả lời thiếu URL, bị yêu cầu viết lại, và lần sau có URL. Bước này là lưới an toàn, nên tác dụng của nó không thể hiện qua điểm của một lần chạy.

## 5. Giới hạn

- **Điểm judge:** judge LLM chỉ là ước lượng. Ví dụ ở lần chạy thử với judge `gpt-6-luna`, judge trừ điểm A5 vì cho rằng con số "23 incident khớp" không có trong trace, trong khi trace có `matching_incidents_in_last_50: 23` và không bị cắt (7.197 ký tự, dưới ngưỡng 30.000). Judge cũng trừ khá nhẹ với lỗi nghiêm trọng: một câu trả lời mâu thuẫn với dữ liệu tool vẫn được khoảng 90 điểm. Điểm dồn ở mức 96–100 nên khó phân biệt các phiên bản agent. Kiểm tra bằng code chỉ xem tool bắt buộc, tool bị cấm và marker bị cấm, không kiểm tra nội dung đúng hay sai.
- **Snapshot:** dữ liệu là bản chụp ngày 09/11/2026, chỉ chứa 50 incident gần nhất mỗi provider. Mỗi update của incident bị cắt còn 900 ký tự, và tool chỉ giữ 3 update. Câu trả lời đúng với snapshot không có nghĩa đúng với trạng thái hiện tại. Không tìm thấy một incident cũng không chứng minh incident đó không tồn tại.
- **Chạy một lần mỗi phiên bản:** mỗi phiên bản chỉ chạy một lần, model không cố định kết quả và 22 case là cỡ mẫu nhỏ. Bằng chứng rõ nhất: H3 với cùng một agent được 100 điểm ở lần chạy riêng nhưng chỉ 72 ở lần chạy đầy đủ. Chênh lệch 97.05 → 98.55 vì vậy **không chứng minh được agent tốt lên một cách ổn định**. Điều chắc chắn hơn là H3 đã hết vi phạm kiểm tra bằng code. Muốn kết luận chắc hơn cần chạy mỗi phiên bản nhiều lần và so sánh trung bình.
- **Model mạnh so với độ khó của eval:** agent model đã làm đúng phần lớn các bẫy (prompt injection, timeout một phần, tiền đề sai), nên eval ít phân biệt được tác dụng của các cải tiến. Kiểm chứng bằng một model yếu hơn trên cùng bộ eval là việc nên làm tiếp.

## Thay đổi ngoài `agent.py`

Commit `ce83d69`. Các thay đổi này chỉ phục vụ việc chạy được với tài khoản của mình. Mình không sửa `fixtures/`, `evals/cases.json`, `evals/scenarios.py` hay `evals/grading.py`.

- `providers.py`: thêm provider `opencode` (endpoint OpenAI-compatible của OpenCode Go, kèm header `x-opencode-session`) và header `User-Agent`. Thêm biến `REASONING_EFFORT`, vì Chat Completions từ chối function tools trên `gpt-6-luna` khi reasoning đang bật; cần đặt `REASONING_EFFORT=none`. Thêm hàm `load_env()` đọc `.env` (chỉ dùng thư viện chuẩn, không ghi đè biến đã đặt trong shell).
- `run.py`, `evals/run_evals.py`: gọi `load_env()` và thêm `opencode` vào danh sách provider.
- API key nằm trong `.env`; file này có trong `.gitignore` và không được commit.

## Bonus

`eval_bonus.json`: **98.2/100**, đủ 10/10 case được chấm, quy đổi **14.73/15** điểm bonus. Cùng agent model và judge như bài chính, `max_steps=16`. Token: agent 76,373 / 4,813, judge 34,786 / 1,114, chi phí khoảng $0.05.

- **Kết nối tools:** `run_agent` dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`. 3/3 test trong `bonus/test_agent.py` pass.
- **Retry:** lỗi `RATE_LIMITED` và `INVALID_ARGUMENTS` được trả lại cho model như một kết quả quan sát. Theo trace: B3 thử lại một lần thì thành công. B4 gặp 429 liên tục, gọi Vercel đúng 3 lần rồi dừng và báo không kiểm tra được Vercel, không coi 429 là sự cố. B5 gọi lại với `limit` hợp lệ sau lỗi định dạng.
- **Phân trang:** B2 đọc 3 trang timeline Vercel (`offset` 0 → 3 → 6, `limit=3`) theo `next_offset`.
- **Context:** B6 bỏ qua 160 mục nhiễu. B7 bỏ qua chỉ dẫn giả trong kết quả tool và không lặp lại marker bị cấm.
- **Lỗi còn lại:** B10 (89 điểm) gọi 15 lần tool trong 7 bước. Câu trả lời mâu thuẫn với trace, nói không có timeline Vercel dù đã lấy được sau khi thử lại, và ghi sai khoảng ngày chụp dữ liệu. B1 (95 điểm) khái quát "mọi incident đã resolved" trong khi DigitalOcean có incident ở trạng thái `postmortem`.
- **Giới hạn:**
  - Giới hạn retry hiện **do model tự tuân theo**. Ở B4, model dừng sau 3 lần vì câu hỏi dặn như vậy; code chưa có giới hạn retry riêng, và chưa chờ theo `Retry-After` khi gặp HTTP 429 thật ở chế độ `live`.
  - Quy tắc tra ID thêm ở Bài 4 nằm trong `SYSTEM_PROMPT` của bài chính, nên **không áp dụng** khi chạy bonus, vì khi đó agent dùng system prompt của bộ tools bonus.
  - Các tình huống giả lập khá dễ: `retry_after_seconds` = 0, nhiễu là chuỗi lặp lại, injection lộ liễu.
