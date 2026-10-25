# Trợ Lý Ông Bà — Design

## Mục đích
App Android (Google Play) giúp người lớn tuổi tự làm những việc cơ bản trên điện thoại mà hiện họ phải nhờ con cháu: gọi điện, nhắn tin, gọi Zalo cho người thân, nhớ tài khoản/mật khẩu (VNeID, ngân hàng, Zalo…), và xem hướng dẫn từng bước.

Người dùng có 2 vai trò trên **cùng một máy**:
- **Ông bà** (người dùng chính): chỉ bấm nút to, đọc chữ to, nghe đọc to.
- **Con cháu** (người cài đặt): thêm người thân, nhập tài khoản, chỉnh cỡ chữ. Vào "Chế độ cài đặt" bằng mã PIN 4 số để ông bà không lỡ tay xoá/sửa.

## Mục tiêu (v1)
1. **Màn hình chính**: 4 ô thật to: Người thân · Sổ tài khoản · Hướng dẫn · Gọi khẩn cấp.
2. **Người thân**: danh sách có ảnh + tên to (chọn từ danh bạ máy). Bấm vào một người → 3 nút: **Gọi điện** (mở trình quay số với số có sẵn), **Nhắn tin** (mở app SMS với số có sẵn), **Gọi Zalo** (mở Zalo tới đúng người đó).
3. **Gọi khẩn cấp**: một người thân được chọn làm liên hệ khẩn cấp, bấm là gọi ngay.
4. **Sổ tài khoản**: lưu tên app, tên đăng nhập, mật khẩu, ghi chú. Mở sổ cần vân tay/khuôn mặt/khoá màn hình của máy. Chữ hiện to, có nút "Sao chép" và nút "Mở app" (VD mở VNeID).
5. **Hướng dẫn**: các bài hướng dẫn từng bước (Cách đăng nhập VNeID, Quên mật khẩu VNeID, Cách gọi Zalo, Cách nhắn tin…), mỗi bước có chữ to, ảnh minh hoạ tuỳ chọn, và nút **Đọc to** (giọng tiếng Việt). Nội dung lấy từ Supabase để cập nhật không cần ra bản app mới; dùng được khi không có mạng.
6. **Cài đặt** (sau PIN): cỡ chữ (Lớn / Rất lớn / Cực lớn), đổi PIN.

## Không làm trong v1
- Không đăng nhập, không tài khoản người dùng.
- Không đồng bộ nhiều máy, không cho con cháu điều khiển từ xa.
- Không thay launcher của máy, không tự đăng nhập hộ vào app khác.
- iOS: chưa phát hành (Expo vẫn giữ khả năng làm sau).

## Không được xảy ra (anti-goals)
- **Mật khẩu không bao giờ rời khỏi máy**: không gửi lên Supabase, không log, không analytics.
- Không có bước nào bắt ông bà gõ chữ hay tìm trong menu nhỏ.
- Không có màn hình "lỗi" kiểu kỹ thuật: lỗi nào cũng là một câu tiếng Việt dễ hiểu kèm một nút làm tiếp.
- Không xin quyền bị Play hạn chế (SEND_SMS, READ_CALL_LOG).

## Ràng buộc
- Expo SDK (bản ổn định mới nhất lúc scaffold), React Native, TypeScript strict, expo-router.
- Supabase chỉ đọc công khai (anon key + RLS chỉ SELECT) cho bảng hướng dẫn.
- Chữ tối thiểu 22pt, nút tối thiểu 72dp cao, độ tương phản AA trở lên.
- Toàn bộ chữ trên app là tiếng Việt.

## Quyết định của Claude (người dùng chưa duyệt từng mục)
- Gọi điện bằng `tel:` (mở trình quay số, ông bà bấm nút xanh) thay vì gọi thẳng, để khỏi phải xin quyền CALL_PHONE. Riêng nút Gọi khẩn cấp cũng dùng `tel:` ở v1.
- Gọi Zalo bằng link `https://zalo.me/<số điện thoại>`, mở thẳng cuộc trò chuyện; ông bà bấm biểu tượng gọi trong Zalo. Bài hướng dẫn "Gọi Zalo" có ảnh chỉ nút đó.
- Mở app khác (VNeID, Zalo) bằng `expo-intent-launcher.openApplication(package)`; nếu máy chưa cài thì mở trang app trên CH Play.
- Lưu mật khẩu bằng `expo-secure-store` (Android Keystore), mỗi tài khoản một khoá riêng.
- Dữ liệu không nhạy cảm (người thân, cài đặt, bộ nhớ đệm hướng dẫn) lưu bằng AsyncStorage.
