# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Mẫu điền (mỗi bạn 1 case, lấy từ trial đang fail trong report)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11`, run ID `E-seeded_regression-20261119-0cb4` trong `reports/eval_seeded_regression.json` |
| Triệu chứng | Graders fail: `RefundAmountCorrect` (ledger 20000 != label 0), `DbFactKept` (phát hiện 1 event model_override), `pass_rate` (status resolved != denied). Reply nói: *"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."* trong khi đơn O-202 chỉ trễ 2 ngày và không đủ điều kiện bồi thường (label refund_vnd = 0). |
| Bước sai đầu tiên | Bước 3: kind `model_override`, payload `{"delay_days": 5}`. |
| Vì sao là bước *đầu tiên* | Bước 1 (`handoff`) và Bước 2 (`investigator`) hoàn toàn đúng: investigator tra cứu DB chính xác trả về `order.delay_days = 2`. Bước 3 là thời điểm đầu tiên Agent tự ý lấy thông tin khách khai khống trong ticket ("trễ 5 ngày") để ghi đè dữ liệu DB (`model_override`). Các bước 4, 5, 6 bị sai dây chuyền trực tiếp từ quyết định sai này. |
| Failure mode | `FM-03: ticket_trust — tin ticket hơn DB` |
| Đối chứng | Cùng `C-002@11` trên shape `fixed`: Bước 2 trả về `delay_days = 2`. Agent không kích hoạt override; bước 3 tính quote trả về `amount_vnd: 0, reason: "no_entitlement"`; bước 4 reply từ chối bồi thường lịch sự và ledger ghi nhận 0đ (khớp 100% với nhãn). |
| Regression test | `test_override_event_breaks_db_fact` trong `tests/test_lab9_graders.py` và metric `db_fact_kept` (grader `DbFactKept`). Đã kiểm chứng: Fail trên `seeded_regression` (db_fact_kept tụt còn 0.9286) và Pass 100% trên `fixed` (db_fact_kept = 1.0). |
| Chưa chắc chắn | Trace chỉ ghi nhận hành vi override sau khi LLM parse ticket, không cho biết liệu LLM hiểu nhầm ngữ nghĩa tiếng Việt hay do instruction prompt ưu tiên thông tin khách hàng quá mức. |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
