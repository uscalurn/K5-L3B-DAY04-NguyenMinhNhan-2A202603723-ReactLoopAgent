# Ghi chú phân tích · Day 04

- Họ và tên: Nguyễn Minh Nhân
- MSSV: 2A202603723

## 1. Case được chọn

- **ID:** H6
- **Câu hỏi:** Kiểm tra GitHub Actions, xem incident `h02n9572mx9y` rồi theo link incident liên quan để tìm nguyên nhân.
- **Lần 1 (60/100):** Agent nói Actions đang hoạt động bình thường, incident đã resolved. Nhưng agent chưa mở incident liên quan mà đã kết luận chưa đủ thông tin về nguyên nhân.

## 2. Bằng chứng trong trace

```text
Before:
step 1: get_status(github, actions)
step 1: get_incident_details(github, h02n9572mx9y)
         -> có link tới incident xr3myzj7ms51
step 2: model dừng, không mở link đó
=> H6: 60 điểm, tool chỉ được gọi ở 1 model step

After:
step 1: get_status + get_incident_details(h02n9572mx9y)
step 2: get_incident_details(xr3myzj7ms51)
         -> tìm được nguyên nhân là thay đổi cấu hình event-routing
step 3: model trả lời
=> H6: 100 điểm
```

Em nghĩ lỗi là model thấy link trong kết quả tool nhưng không gọi tiếp để đọc, nên trả lời hơi sớm.

## 3. Thay đổi đã thực hiện

- Em sửa `SYSTEM_PROMPT` và `run_agent()` trong `agent.py` để agent chú ý đọc incident liên quan khi câu hỏi cần, kiểm tra URL nguồn trước khi trả lời.
- Thêm phần nhận diện link incident và kiểm tra còn thiếu bằng chứng hay không. Vẫn giữ `max_steps`, xử lý lỗi tool/JSON và retry 429 có giới hạn.
- Em chọn sửa chỗ này vì có link thôi chưa đủ, cần thật sự gọi tool đọc nội dung. Trong lần chạy sau, model tự gọi tiếp ở step 2, không phải do validator ép gọi.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng (22 case) | 91.77 | 98.45 |
| Điểm H6 | 60 | 100 |
| Số model calls (cả bộ) | 50 | 52 |
| Tổng token (cả bộ) | 39,074 | 71,179 |
| Chi phí | File không ghi | File không ghi |

Hai lần đều chạy `gpt-4.1-mini` cho agent và judge, `max_steps=8`, dataset `core`; timestamp snapshot quan sát được trong H6 giống nhau.

- **Đã cải thiện:** H6 đã đọc thêm incident gộp và nêu được nguyên nhân, có dẫn nguồn.
- **Vẫn còn lỗi:** E1 từ 98 xuống 85 vì agent tìm `github-actions` không ra incident, rồi lại nói không có sự cố gần đây. Cách tìm chuỗi này chưa tốt.

## 5. Giới hạn

- Điểm do LLM judge chấm nên có thể chưa chính xác hoàn toàn.
- Dữ liệu là snapshot, tool chỉ tìm trong 50 incident gần nhất; không tìm thấy chưa chắc là không có. Một số update còn bị cắt ngắn.
- Em mới chạy mỗi phiên bản một lần, nên chưa biết kết quả có ổn định không. Agent cũng có lúc gọi tool chưa đúng do nhận diện câu hỏi bằng từ khóa.