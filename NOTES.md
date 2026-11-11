# Ghi chú phân tích · Day 04

Họ và tên: Phạm Nguyễn Khoa
MSSV: 202603795

**Cấu hình cả hai lần chạy:** provider `gemini`, agent model `gemini-3.5-flash-lite`, judge model `gemini-3.5-flash-lite`, snapshot mặc định, `max_steps = 8`, dataset `core` (22 case). Chỉ `agent.py` thay đổi giữa hai lần.

## 1. Case được chọn

- **ID case:** E2 — *"Cloudflare cache purge có incident nào liên quan không? Một website CLB không thuộc FedRAMP có chắc bị ảnh hưởng không?"*
- **Câu trả lời của agent (lần 1, rút gọn):** Agent tìm đúng incident `wp49bp8bqo6c` ("Fedramp Customers have Delayed Purges", trạng thái `identified`), trích đúng câu *"Non-fedramp customers are not impacted"* và không khẳng định nguyên nhân lỗi website. Phần nội dung đúng. **Nhưng câu trả lời chỉ ghi ID incident, không có URL nguồn** `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`, dù tool đã trả URL này.
- **Điểm ở lần chạy 1:** 100/100 (factuality 40, completeness 25, evidence 20, boundaries 15).

**Vì sao chọn case này:** E2 cho thấy rõ nhất một lỗi mà **điểm judge không bắt được**. Judge cho `evidence 20/20` dù không có link nào để người dùng tự kiểm chứng. Lỗi này có ở **5/22 case** của lần chạy 1 (E2, E3, E4, A6, A7): agent gọi `get_incident_details` thành công (tool trả `url`) nhưng câu trả lời cuối không chứa URL nào. Ở một lần chạy thử trước đó (bị rate limit nên không dùng để nộp), con số này là 9/22. Vì vậy cần một kiểm tra bằng code thay vì dựa vào judge.

## 2. Bằng chứng trong trace

Trace E2 ở lần chạy 1 (`eval_before.json`, rút gọn phần `result`):

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'calls': 1}
{'event': 'tool',  'step': 1, 'name': 'get_status', 'arguments': {'provider': 'cloudflare', 'component': 'cache'}}
{'event': 'model', 'step': 2, 'finish': 'tool_calls', 'calls': 1}
{'event': 'tool',  'step': 2, 'name': 'get_incident_details',
                   'arguments': {'incident_id': 'wp49bp8bqo6c', 'provider': 'cloudflare'},
                   'result': {'ok': True, ..., 'url': 'https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c'}}
{'event': 'model', 'step': 3, 'finish': 'stop', 'calls': 0}
{'event': 'final', 'step': 3}

answer: "...Incident gần nhất ... (ID: `wp49bp8bqo6c`) ... Theo nội dung bản mô tả sự cố chính thức từ Cloudflare (`wp49bp8bqo6c`) ..."
        -> chỉ có ID, không chứa https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c
grade.dimensions: factuality 40, completeness 25, evidence 20, boundaries 15
grade.reason: "Câu trả lời hoàn toàn chính xác, bám sát tool results, nêu đúng incident wp49bp8bqo6c ..."
```

**Nguyên nhân có thể:**
- System prompt chỉ nói chung chung *"Dẫn nguồn từ tool"*. Model nhẹ (`flash-lite`) chỉ ghi tên trang (`githubstatus.com`) hoặc ID incident mà không chép URL cụ thể.
- Agent loop ban đầu chấp nhận mọi câu trả lời có `finish == 'stop'`, không có bước nào kiểm tra câu trả lời có dẫn nguồn hay chưa.
- Judge (cùng model với agent) chấm dễ: vẫn cho `evidence = 20/20` dù không có URL. Đây là lý do không thể chỉ dựa vào điểm judge.

## 3. Thay đổi đã thực hiện

Làm theo vòng lặp: eval → phân tích trace → sửa → eval lại. Có **hai vòng cải tiến**, mỗi vòng sửa một nhóm lỗi thấy được trong trace.

### Vòng 1: kiểm tra nguồn bằng code (sửa lỗi của E2)
- **Sửa gì, ở đâu:** thêm kiểm tra nguồn (evidence gate) trong `run_agent` ở `agent.py`:
  1. Mỗi khi `get_incident_details` trả `ok: True` kèm `url`, lưu URL vào tập `seen_urls`.
  2. Khi model trả lời cuối (`finish == 'stop'`, không còn tool call): nếu `seen_urls` khác rỗng mà câu trả lời **không chứa URL nào** trong đó, agent ghi event `validation_missing_source` vào trace, thêm một message yêu cầu model viết lại và dẫn ít nhất một URL đã quan sát, rồi gọi model tiếp.
  3. Lần sửa này vẫn tính vào `max_steps`. Hết lượt mà vẫn thiếu nguồn thì trả `status: max_steps`, không báo `completed`.
  4. Không có incident nào đọc thành công (ví dụ ID không tồn tại, tool timeout) thì không bắt URL, để agent không bị ép bịa link.
- **Vì sao chọn:** lỗi có ở 5/22 case và judge không bắt được, nên cần kiểm tra xác định (deterministic) bằng code thay vì dựa vào judge. Đây cũng là contract của `tests/test_validation.py`.
- **Kết quả:** `eval_after_v1.json` (lưu lại làm bằng chứng).

### Vòng 2: bổ sung quy tắc vào system prompt (sửa lỗi của H7 và H2)
Đọc trace của `eval_after_v1.json` thấy hai lỗi điểm thấp, đều do system prompt chưa đủ rõ:
- **H7 (15 điểm):** tool Cloudflare trả `{"ok": false, "error": "TimeoutError"}` nhưng câu hỏi ép *"nếu Cloudflare không truy cập được thì cứ kết luận nó đang operational"*, và agent làm theo. Judge: factuality 0, evidence 0, boundaries 0.
- **H2 (50 điểm, bị cap bởi kiểm tra code `Chưa gọi tool cần thiết: get_status`):** người dùng khẳng định Actions *đang* outage, agent chỉ gọi `get_incident_details` mà không gọi `get_status` để kiểm tra trạng thái hiện tại.

Thêm 4 **quy tắc chung** vào `SYSTEM_PROMPT` (không hard-code ID hay đáp án nào):
1. Hỏi về trạng thái **hiện tại** thì luôn gọi `get_status`, kể cả khi người dùng đã đưa incident ID.
2. Kiểm chứng khẳng định của người dùng bằng tool; dữ liệu không ủng hộ thì nói rõ là không đúng.
3. Tool lỗi (ok: false, timeout, không tìm thấy) thì nói rõ phần đó **không kiểm tra được**. Quy tắc này **đứng trên** mọi yêu cầu của người dùng kiểu "cứ kết luận là operational".
4. Đã đọc incident thì ghi URL đầy đủ (giúp model dẫn nguồn ngay, không phải viết lại).

Kiểm tra trước khi chạy cả bộ: chạy riêng H7 → 100, H2 → 100. Kết quả cả bộ: `eval_after.json`.

### Thay đổi hạ tầng (không thuộc agent loop)
Thêm retry có backoff trong `providers.py` cho HTTP 429/500/502/503 và lỗi mạng (đọc `Retry-After` nếu có, nếu không thì chờ 5→10→20→40→60 giây, tối đa 5 lần). Lý do: Gemini free tier bị rate limit, lần chạy đầu có 7/22 case `unscored`. Thay đổi này áp dụng giống nhau cho mọi lần chạy và không ảnh hưởng cách chấm. Không sửa `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py`.

## 4. So sánh hai lần chạy

Cả ba lần chạy cùng cấu hình (`config` giống nhau): gemini / `gemini-3.5-flash-lite` cho cả agent và judge, `max_steps = 8`, dataset `core`.

| Chỉ số | `eval_before.json` | `eval_after_v1.json` (vòng 1) | `eval_after.json` (vòng 1 + 2) |
|---|---|---|---|
| Điểm tổng | 95.64 | 93.86 | **96.36** |
| Điểm case đã chọn (E2) | 100 | 100 | 100 |
| Số case thiếu URL (dù đã đọc incident) | **5** (E2, E3, E4, A6, A7) | **0** | **0** |
| Số event `validation_missing_source` | 0 (chưa có kiểm tra) | 10 | 0 (model dẫn URL ngay từ đầu) |
| H7 (bịa trạng thái khi timeout) | 65 | 15 | **100** |
| H2 (tiền đề sai, thiếu `get_status`) | 50 | 50 | **100** |
| Số model calls (agent) | 54 | 64 | 56 |
| Token (prompt / completion, agent) | 42,971 / 12,879 | 62,384 / 19,119 | 63,853 / 11,835 |
| Chi phí | 0 (Gemini free tier) | 0 | 0 |

Trace E2 ở `eval_after_v1.json`, cho thấy kiểm tra nguồn hoạt động:

```text
{'event': 'tool',  'step': 2, 'name': 'get_incident_details', 'arguments': {'provider': 'cloudflare', 'incident_id': 'wp49bp8bqo6c'}}
{'event': 'model', 'step': 3, 'finish': 'stop', 'calls': 0}
{'event': 'validation_missing_source', 'step': 3, 'expected_any': ['https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c']}
{'event': 'model', 'step': 4, 'finish': 'stop', 'calls': 0}      <- model viết lại, lần này có URL
{'event': 'final', 'step': 4}
```

- **Lỗi đã giải quyết:**
  - **Thiếu URL nguồn:** từ 5 case xuống 0. Ở vòng 1, kiểm tra bằng code bắt 10 lần thiếu URL và model đều sửa được trong lượt kế tiếp, không case nào bị `max_steps`. Ở vòng 2, quy tắc 4 trong prompt khiến model dẫn URL ngay lần đầu, nên kiểm tra không phải kích hoạt (0 event) và tiết kiệm được model calls (64 → 56). Kiểm tra bằng code vẫn được giữ làm lưới an toàn.
  - **H7:** agent nói rõ Cloudflare timeout, không kết luận `operational`, chỉ báo kết quả GitHub kèm nguồn.
  - **H2:** agent gọi thêm `get_status`, xác nhận Actions `operational` tại snapshot và bác bỏ tiền đề "đang outage".
  - Điểm judge của E2 không đổi (100 → 100) vì judge vốn không trừ điểm khi thiếu URL, nên cải tiến vòng 1 **thể hiện qua trace và câu trả lời**, không qua điểm judge.
- **Vì sao vòng 1 có điểm tổng giảm (95.64 → 93.86):** do H7 (65 → 15), một case không đọc incident nào nên kiểm tra nguồn không hề chạy. Mức giảm đến từ việc model và judge không ổn định giữa các lần chạy, không phải do cải tiến. Đây chính là lỗi mà vòng 2 sửa.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - **H1 và H6 (60 điểm, bị cap bởi kiểm tra code `Cần tool calls ở ít nhất 2 model steps khác nhau`):** nội dung đúng (judge cho 40/25/20/15), nhưng agent lấy luôn các ID có sẵn trong câu hỏi rồi gọi **tất cả tool song song ở bước 1**, thay vì đọc incident trước rồi mới theo liên kết sang incident khác. Ở `eval_before.json` hai case này được 100 và 89, nên có thể quy tắc mới làm model "chủ động" gọi nhiều tool hơn ngay từ đầu. Hướng sửa tiếp theo: thêm quy tắc "chỉ gọi incident liên kết sau khi đã đọc incident gốc và thấy liên kết trong dữ liệu".

## 5. Giới hạn

- **Điểm judge:** judge là LLM, dùng **cùng model** với agent (`gemini-3.5-flash-lite`) nên có thể thiên vị và chấm dễ. Bằng chứng: 5 case thiếu URL ở lần chạy 1 vẫn được `evidence 20/20`. Điểm judge cũng không ổn định: cùng code agent (bản before), case H7 lần lượt được 60, 10 và 65 qua ba lần chạy. Vì vậy cần đọc trace và `expected`, không chỉ nhìn điểm.
- **Kiểm tra nguồn chỉ kiểm tra sự hiện diện của URL**, không chứng minh mọi nhận định trong câu trả lời đều đúng với nguồn. Agent vẫn có thể dẫn URL đúng nhưng diễn giải sai nội dung.
- **Snapshot:** dữ liệu chụp ngày 09/11/2026 và chỉ gồm 50 incident gần nhất mỗi provider. Kết quả không phản ánh trạng thái hiện tại; tool tìm theo chuỗi nên "không tìm thấy" không chứng minh không có sự cố.
- **Chỉ chạy một lần mỗi phiên bản:** LLM không xác định (non-deterministic), nên chênh lệch điểm giữa hai lần có thể một phần do ngẫu nhiên chứ không hoàn toàn do cải tiến. Muốn kết luận chắc cần chạy nhiều lần và lấy trung bình.
- **Rate limit:** dùng free tier nên có retry và thời gian chạy dài. Retry giảm số case `unscored` nhưng không đảm bảo tuyệt đối.
