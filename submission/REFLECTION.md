Họ và tên: Nguyễn Thành Đạt
Mã học viên: 2A202600203

Trong các anti-pattern được đề cập, đội ngũ của chúng tôi dễ mắc phải rủi ro **"Small-file problem" (Vấn đề file nhỏ)** nhất.

Nguyên nhân là do hệ thống hiện tại chủ yếu tiếp nhận dữ liệu streaming liên tục từ các microservices và hệ thống log (ví dụ: user click-stream, system events). Việc ghi dữ liệu theo từng batch nhỏ giọt với tần suất cao vào Data Lake mà không có cơ chế quản lý sẽ nhanh chóng sinh ra hàng trăm ngàn file cực nhỏ. 

Điều này gây áp lực lớn lên hệ thống quản lý metadata, làm chậm đáng kể các quá trình tính toán analytics ở hạ tầng. Thay vì đọc dữ liệu một cách tối ưu, engine truy vấn lại tốn phần lớn thời gian rảnh rỗi chỉ để mở và đóng vô số file nhỏ. 

Để khắc phục rủi ro này, chúng tôi cần tích hợp bài học từ Lab vào kiến trúc thực tế: thiết lập các job tự động chạy lệnh `OPTIMIZE` kết hợp với `Z-ORDER` (theo các cột thường xuyên được filter như `user_id` hay `date`) định kỳ hàng ngày. Việc này giúp liên tục gộp các file nhỏ thành file có kích thước chuẩn, đảm bảo tốc độ truy vấn luôn được duy trì ổn định.
