# Ghi chú phân tích · Day 04

Họ và tên: Hồ Xuân Vũ
MSSV: 2A202603406
Lớp: L3A

## 1. Case được chọn

- ID: A5
- Câu hỏi: Tìm danh sách đầy đủ incident GitHub Actions trong năm 2025 và nói rõ độ đầy đủ.
- Kết quả baseline trong `eval_before.json`: **50/100**. Đây là case trong bộ đầy đủ 22 case.
- Tóm tắt lỗi: Agent nhận thấy tool chỉ tìm trong 50 incident gần nhất nhưng vẫn kết luận không có incident nào trong năm 2025. Nguồn không đủ dữ liệu để kết luận như vậy.

## 2. Bằng chứng trong trace

Trace A5 trong `eval_before.json`:

1. Agent gọi `get_status(provider="github", component="Actions")`.
2. Tool trả `matching_incidents_in_last_50: 23` và ghi chú: chỉ tìm trong 50 incident gần nhất; không tìm thấy không chứng minh là không có incident.
3. Dù đã có cảnh báo giới hạn đó, câu trả lời vẫn kết luận không có incident nào được ghi nhận trong năm 2025 và trình bày danh sách gần đây như thể đã đáp ứng yêu cầu đầy đủ.

Judge chấm 50 điểm và nêu rõ kết luận về năm 2025 không được hỗ trợ bởi dữ liệu. Đây là lỗi diễn giải giới hạn bằng chứng, không phải lỗi tool.

## 3. Thay đổi đã thực hiện

- `agent.py`: hoàn thiện vòng lặp model–tool: giữ nguyên assistant message, xử lý mọi tool call, trả kết quả với đúng `tool_call_id`, giới hạn số lượt, chuyển arguments JSON lỗi thành observation, ghi trace, và không coi phản hồi bị cắt là hoàn tất.
- `agent.py`: nếu đọc chi tiết incident thành công, lưu URL nguồn; nếu câu trả lời thiếu URL đã quan sát thì ghi `validation_missing_source` và yêu cầu model sửa trong giới hạn lượt.
- `agent.py`: cập nhật system prompt theo evidence của A5: khi tool chỉ xem 50 incident gần nhất thì không được tuyên bố danh sách lịch sử đầy đủ hoặc kết luận không có incident thuộc khoảng thời gian ngoài dữ liệu đã xem. Prompt cũng yêu cầu phân biệt thời điểm snapshot và phạm vi ảnh hưởng của incident.
- `agent.py`: lỗi provider được trả ra kết quả lỗi, đồng thời che key trước khi đưa chi tiết lỗi vào output.
- `providers.py`: xác thực OpenAI-compatible API bằng Bearer token lấy từ biến môi trường; key không được ghi trực tiếp trong source.

## 4. So sánh eval

Đã chạy đủ cả 22 case trước và sau cải tiến. Hai report cùng cấu hình: `openai`, agent/judge model `gpt-4o-mini`, `max_steps=8`, dataset `core`.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Case đã chấm | 22/22 | 22/22 |
| Điểm tổng | 81.14/100 | 80.59/100 |
| Case A5 | 50/100 | 85/100 |
| Agent model calls | 51 | 49 |
| Token agent | 46,308 | 46,024 |
| Token judge | 39,400 | 38,774 |
| Tổng token | 85,708 | 84,798 |
| Chi phí | Không có trong report | Không có trong report |

A5 tăng 35 điểm; câu trả lời sau cải tiến nói rõ không thể tạo danh sách đầy đủ và không thể kết luận không có incident năm 2025 từ 50 incident gần nhất. Tuy vậy, điểm trung bình toàn bộ giảm 0.55 điểm (81.14 xuống 80.59). Các điểm judge có thể dao động; không thể khẳng định giảm nhẹ này là do thay đổi prompt. Cần xem từng case thay vì chỉ dựa vào điểm tổng.

### Các bước đã chạy trong PowerShell

1. Chạy `python -m unittest discover -s tests -v` để kiểm tra agent offline.
2. Cấu hình `PROVIDER=openai`, `MODEL=gpt-4o-mini` và API key trong biến môi trường PowerShell. Key không được ghi vào source hoặc report.
3. Chạy riêng A5 bằng `python evals/run_evals.py --case A5 --output eval_a5_after.json` để xem đúng lỗi đã chọn.
4. Chạy toàn bộ trước cải tiến bằng `python evals/run_evals.py --output eval_before.json`.
5. Cập nhật system prompt để tôn trọng giới hạn 50 incident gần nhất, chạy lại unit test, sau đó chạy toàn bộ bằng `python evals/run_evals.py --output eval_after.json`.
6. Kiểm tra cả hai report có 22/22 case được chấm, điểm khác `null`, và cấu hình giống nhau. Các file hiện tại thỏa các điều kiện này.

Không sửa tay score, trace, expected hay dataset. `eval_e2.json` là một lần chạy E2 độc lập (1/1, 80/100), không thay thế eval toàn bộ; judge có thể đưa điểm khác nhau cho cùng case giữa các lần chạy.

## 5. Giới hạn

- Điểm LLM judge chỉ là ước lượng; phải đối chiếu `expected`, câu trả lời và tool trace.
- Snapshot là dữ liệu tại một thời điểm; tìm kiếm giới hạn trong 50 incident không đại diện cho toàn bộ lịch sử hay status hiện tại.
- Một lần chạy cho mỗi phiên bản không chứng minh độ ổn định; kết quả có thể đổi giữa các lần chạy/model.
- A5 cải thiện trong lần chạy này nhưng điểm tổng giảm nhẹ; chỉ một lần chạy mỗi phiên bản không chứng minh xu hướng ổn định hoặc quan hệ nhân quả.
