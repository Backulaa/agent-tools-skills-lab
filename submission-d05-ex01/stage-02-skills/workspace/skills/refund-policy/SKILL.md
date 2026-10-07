---
name: refund-policy
description: Tra cứu và áp dụng chính sách hoàn tiền theo ngày mua từ tài liệu trong workspace. Dùng khi người dùng hỏi có được hoàn tiền, điều kiện hoàn tiền hoặc phí hoàn tiền; kể cả khi thông tin mua hàng chưa đầy đủ.
---

# Tra cứu chính sách hoàn tiền

1. Kiểm tra có đủ ngày mua, ngày yêu cầu hoàn và trạng thái kích hoạt.
   Nếu thiếu thông tin, hỏi lại phần còn thiếu rồi dừng; chưa kết luận
   đủ hay không đủ điều kiện và không tự giả định chưa kích hoạt.
   Nếu ngày không hợp lệ, mơ hồ hoặc ngày yêu cầu trước ngày mua,
   yêu cầu người dùng xác nhận lại.

2. Đọc reference bằng read_file tại:
   skills/refund-policy/references/answer-template.md

3. Dùng list_files liệt kê data/policies/.
   Lấy đường dẫn từ kết quả tool, không suy đoán hoặc cố định tên file.

4. Đọc các mục có type là file bằng read_file theo đường dẫn vừa tìm được.
   Xác định phạm vi hiệu lực từ nội dung tài liệu, không dựa vào tên
   hoặc thứ tự file trong kết quả liệt kê.

5. Chọn chính sách có phạm vi hiệu lực chứa ngày mua.
   Không dùng ngày yêu cầu hoàn hoặc ngày hiện tại để chọn phiên bản.
   Đọc kỹ điều kiện bao gồm hoặc không bao gồm ngày biên.
   Nếu không có chính sách phù hợp hoặc phạm vi chồng lấn không rõ
   cách ưu tiên, báo chưa đủ căn cứ và yêu cầu làm rõ.

6. Tính số ngày đã qua bằng chênh lệch ngày lịch giữa ngày yêu cầu hoàn
   và ngày mua. Không cộng thêm một ngày và không dùng ngày của máy.

7. Đối chiếu thời hạn và các điều kiện khác, gồm trạng thái kích hoạt,
   theo tài liệu đã chọn. Bằng đúng giới hạn ngày vẫn đáp ứng điều kiện
   thời gian. Chỉ kết luận đủ điều kiện khi đáp ứng mọi điều kiện.

8. Trả lời theo reference, nêu chính sách áp dụng, số ngày đã qua,
   kết luận, lý do, phí nếu đủ điều kiện và đường dẫn tài liệu căn cứ.
   Nếu phí là phần trăm nhưng thiếu giá trị đơn hàng, chỉ nêu tỷ lệ;
   không tự tạo giá trị đơn hoặc số tiền phí.

9. Nếu tool trả lỗi, báo lỗi và phần chưa đủ căn cứ.
   Không bịa nội dung chính sách hoặc tuyên bố đã đọc thành công.