# Judge audit

Chạy: `python -m lab9.judge --csv data/eval/judge_tone_public.csv --audit 5`
và `python -m lab9.judge --csv data/eval/judge_helpful_public.csv --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | Có (κ=0.757 ≥ 0.6, P=0.857 ≥ 0.8, R=0.800 ≥ 0.7) | Dùng |
| helpful | 100 | 0.375 | 1.000 | 0.300 | Không (R=0.300 < 0.7, κ=0.375 < 0.6) | Loại |

## ≥5 case audit thủ công

| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú / Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | H-008 | PASS | FAIL | Người đúng | Reply: *"Em đã hoàn tiền cho anh rồi nha, anh check giúp em."* — Xưng hô suồng sã ("em", "anh", "nha"), không đạt chuẩn giao tiếp CS doanh nghiệp. Judge bị nhầm là thân thiện. **Sửa rubric:** Bổ sung điều khoản cấm đại từ thân mật suồng sã, chuẩn hóa xưng hô sang "LogiX - anh/chị". |
| 2 | H-017 | PASS | FAIL | Người đúng | Reply: *"Dạ, bạn ơi LogiX đã hoàn 25.000đ cho đơn O-606 rồi nha."* — Xưng "bạn ơi" với khách hàng là thiếu trang trọng. **Sửa rubric:** Bắt buộc đại từ nhân xưng là "anh/chị", cấm dùng "bạn ơi", "nha". |
| 3 | H-029 | PASS | FAIL | Người đúng | Reply: *"Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm."* — Cụm "không cần nhắn thêm" mang sắc thái mệnh lệnh, xua đuổi khách hàng dù có mở đầu bằng "Dạ, vui lòng". **Sửa rubric:** Thêm tiêu chí cấm các câu lệnh cụt lủn hoặc ngăn cản khách hàng phản hồi. |
| 4 | H-042 | FAIL | PASS | Người đúng | Reply: *"Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ."* — Judge phạt vì cho rằng "thiếu lời xin lỗi", nhưng đây là thông báo thành công ngắn gọn, rõ ràng, lịch sự. **Sửa rubric:** Làm rõ thông báo hoàn tiền thành công không bắt buộc phải có câu xin lỗi nếu đã cung cấp đủ thông tin và hạn thời gian. |
| 5 | H-051 | FAIL | PASS | Người đúng | Reply: *"Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ."* — Judge phạt vì dùng từ tiếng Anh "refund", trong khi chính sách CS của LogiX cho phép từ này. **Sửa rubric:** Chấp nhận một số thuật ngữ thông dụng như "refund", không coi là vi phạm tone nếu câu văn vẫn lịch sự. |
| 6 | H-070 | PASS | FAIL | Người đúng | Reply: *"Dạ, LogiX đã hoàn 90.000đ cho đơn O-909 ạ 😂😂"* — Dùng emoji cười cợt trong ngữ cảnh giải quyết khiếu nại hoàn tiền. **Sửa rubric:** Cấm sử dụng emoji không phù hợp với ngữ cảnh khiếu nại/dịch vụ khách hàng. |
| 7 | H-081 | FAIL | PASS | Judge đúng | Reply: *"Dạ, đơn O-107 chưa đủ điều kiện hoàn theo chính sách hiện hành ạ."* — Người gán nhãn đọc nhầm ticket thành PASS. Reply đúng chính sách từ chối. **Sửa rubric/nhãn:** Cập nhật lại nhãn người chuẩn xác trong tập đối chuẩn. |

## Kết luận
- **Tiêu chí bị loại và lý do:**
  - Tiêu chí `helpful` bị **loại bỏ hoàn toàn khỏi CI Release Gate**.
  - **Dẫn ma trận nhầm lẫn:** `{'tp': 9, 'fp': 0, 'fn': 21, 'tn': 70}`. Trong 30 ca lỗi thực tế của con người (`tp + fn = 30`), judge chỉ phát hiện được 9 ca và bỏ sót tới 21 ca (`fn = 21`), dẫn đến Recall chỉ đạt 30% (`R = 0.3`).
  - **Nguyên nhân cốt lõi:** Judge bị ảo giác bởi sự lịch sự và dông dài (*verbosity bias / sycophancy*). Khi agent trả lời dài dòng, liên tục xin lỗi nhưng đưa ra số tiền hoàn hoàn toàn sai lệch so với sổ cái DB (ví dụ: HL-015, HL-016 báo hoàn 55.000đ trong khi ledger ghi 35.000đ - 40.000đ), judge vẫn chấm là PASS.
  - Theo công thức Rogan-Gladen: Tỉ lệ lỗi quan sát chỉ là 9% (`observed_fail_rate = 0.09`), nhưng sau khi hiệu chỉnh theo sai số của judge thì tỉ lệ lỗi thực tế lên tới 30% (`corrected_fail_rate = 0.3`). Nếu dùng judge này làm gate, 70% các phản hồi sai lệch tài chính sẽ lọt lưới vào production!
- **Rủi ro còn lại:**
  - Judge `tone_ok` dù đạt ngưỡng sàn trên tập dữ liệu này nhưng dữ liệu kiểm thử hiện tại mới chỉ là tập replay đã công khai (`judge_tone_public.csv`).
  - Để triển khai sản xuất an toàn, cần thu thập thêm tập nhãn người độc lập (*held-out real human dataset*) từ các phiên tương tác thực tế của người dùng để tránh overfitting vào tập đánh giá có sẵn.
