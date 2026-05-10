# UI/UX Design Guidelines: Daily Report App (Minimalism Edition)

## 1. Concept & Vibe
Ứng dụng báo cáo hàng ngày với phong cách **Minimalism (Tối giản)**. Tập trung tối đa vào nội dung và sự tinh tế, loại bỏ các chi tiết thừa để tạo không gian làm việc sạch sẽ, hiện đại và chuyên nghiệp.

**Đặc điểm phong cách Minimalism:**
- **Không gian trắng (White Space):** Sử dụng khoảng cách rộng rãi giữa các thành phần để giao diện "thở" được, giúp người dùng tập trung vào việc viết báo cáo.
- **Hình khối sạch sẽ:** Bo góc vừa phải (border-radius khoảng 8px - 12px), không dùng hiệu ứng 3D hay bóng đổ quá đậm.
- **Đường nét tinh tế:** Sử dụng các đường kẻ mảnh (border 1px) hoặc sự thay đổi sắc độ màu nền rất nhẹ để phân chia khu vực thay vì dùng bóng đổ.
- **Typography làm trọng tâm:** Phông chữ sạch, hiện đại là yếu tố trang trí chính.

## 2. Color Palette (Bảng màu)
Bảng màu giữ nguyên tinh thần ấm áp nhưng áp dụng theo cách tối giản:
- **#FFFBF1 (Trắng ấm):** Màu nền chính.
- **#FFF2D0 (Vàng kem nhẹ):** Màu nền phụ cho các khối nội dung, tạo sự phân tách tinh tế với nền chính.
- **#FFB2B2 (Hồng nhạt):** Màu nhấn cho các trạng thái bổ trợ hoặc tag.
- **#E36A6A (Đỏ san hô):** Màu nhấn cho các hành động quan trọng (nút bấm chính).

## 3. Typography (Phông chữ)
Yêu cầu: Hiện đại, cực kỳ sạch sẽ và hỗ trợ Tiếng Việt hoàn hảo.
- **Phông chữ duy nhất:** **`Be Vietnam Pro`** (Google Fonts). 
  - Đây là phông chữ Sans-serif hiện đại, được thiết kế tối ưu cho Tiếng Việt. 
  - Sử dụng các độ dày khác nhau (Light, Regular, SemiBold) để tạo phân cấp thông tin thay vì dùng nhiều kiểu font.
  - Tiêu đề dùng Bold, nội dung dùng Regular, các ghi chú nhỏ dùng Light.

## 4. Cấu trúc dữ liệu & Form nhập liệu (Core Features)
### Logic lưu trữ:
- **Lưu theo ngày:** Báo cáo gắn liền với ngày cụ thể. Giao diện có bộ chọn ngày (Date Picker) tối giản tích hợp ở Header.
- **Tự động tải:** Khi chuyển ngày, ứng dụng tự động tải dữ liệu cũ (nếu có) để xem/chỉnh sửa.

### Form nhập liệu (3 Phần chính):
Sử dụng các thẻ `textarea` với thiết kế phẳng, nền `#FFF2D0` nhạt, viền chỉ xuất hiện khi focus:
1. **Việc đã hoàn thành:** "Hôm nay bạn đã làm được những gì?"
2. **Kế hoạch tiếp theo:** "Dự định cho ngày mai là gì?"
3. **Khó khăn (Blockers):** "Có điều gì đang cản trở bạn không?"

### Tính năng bổ trợ trên Form:
- **Mini Toolbar (Thanh công cụ gọn nhẹ):** Xuất hiện phía trên mỗi ô textarea khi hover. Chứa 2 nút:
  - **Tăng thụt lề (Indent):** Thêm khoảng trắng vào đầu dòng.
  - **Giảm thụt lề (Outdent):** Xóa khoảng trắng ở đầu dòng.
  - Hỗ trợ gõ danh sách đa cấp bằng văn bản thuần (plain text).
- **Nút "Sao chép báo cáo" (Copy):** Thiết kế dạng Outline (viền mảnh `#E36A6A`). Tự động gộp nội dung 3 phần kèm định dạng list đa cấp để dán vào group chat.
- **Nút "Lưu báo cáo" (Save):** Nút đặc màu `#E36A6A`, thiết kế phẳng, không bóng đổ dày.

## 5. Responsive Strategy
- **Desktop:** Bố cục 2 cột rõ ràng, sidebar thanh mảnh bên trái.
- **Mobile:** Chuyển sang bố cục 1 cột, tối ưu hóa diện tích hiển thị của các ô nhập liệu.

## 6. Animations & Micro-interactions
Chuyển động trong phong cách Minimalism cần sự **tinh tế và nhanh gọn**:
- **Timing:** Nhanh hơn (`150ms - 200ms`).
- **Easing:** `ease-in-out` hoặc `linear`.
- **Hover:** Thay đổi nhẹ về màu nền hoặc độ đậm của viền. Nút bấm chính có thể đậm màu hơn một chút khi hover.
- **Transition:** Các hiệu ứng chuyển trang hoặc hiện thông báo dùng `fade-in` nhẹ nhàng.