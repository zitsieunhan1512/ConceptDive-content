---
id: android-process-death-and-saved-state
title: Process Death & SavedStateHandle
domain: android
tags: [android, process-death, savedstate, low-memory-killer, viewmodel]
collections: [interview-fundamental]
related: [android-viewmodel, android-activity-lifecycle, android-memory-leak, android-handler-looper-message-queue]
created: 2026-09-15
diagrams: [Cơ chế|savedstate-process-death-binder.png|Activity ghi Bundle dưới 1MB qua Binder sang system_server trước khi LMKD diệt tiến trình, nạp lại vào SavedStateHandle khi mở lại]
---

## Bản 30 giây
Khi người dùng chuyển sang ứng dụng khác, hệ điều hành Android có thể âm thầm giải phóng bộ nhớ bằng cách hủy tiến trình. Dữ liệu lưu trên ViewModel sẽ biến mất hoàn toàn, chỉ những giá trị được đóng gói trong `SavedStateHandle` mới được khôi phục. Cơ chế này giúp tái tạo nguyên vẹn trải nghiệm của người dùng mà không phụ thuộc vào vòng đời tiến trình.

## Cần biết trước
- android-activity-lifecycle
- android-viewmodel
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2008, Android được thiết kế để chạy đa nhiệm trên các thiết bị di động có dung lượng RAM vô cùng eo hẹp. Hệ thống bắt buộc phải chủ động tiêu diệt các tiến trình chạy ngầm để nhường tài nguyên cho ứng dụng phía trước.

Lập trình viên hay ngỡ ViewModel giữ dữ liệu vĩnh viễn, nhưng nó chỉ sống sót qua việc xoay màn hình. Khi tiến trình bị hủy, người dùng quay lại sẽ gặp màn hình trắng hoặc bị sập ứng dụng do biến rỗng.

Google giới thiệu `SavedStateHandle` để kết nối trực tiếp kho lưu trữ trạng thái của hệ thống vào ViewModel. Lập trình viên không còn phải gọi thủ công các hàm tuần tự hóa phức tạp trong Activity.

## Nó thực sự là cái gì
Tiêu diệt tiến trình (*Process Death*) KHÔNG phải là lỗi tràn bộ nhớ (*OutOfMemory*) do rò rỉ mã nguồn gây ra. Thực chất, đây là hành động chủ động của trình dọn dẹp bộ nhớ nhân Linux dựa trên điểm ưu tiên (*oom_score_adj*).

Hệ thống lưu trữ dữ liệu trạng thái vào bộ nhớ của tiến trình máy chủ hệ thống (*system_server*) trước khi giải phóng.

## Cơ chế
Trước khi thu hồi tiến trình, hệ điều hành yêu cầu Activity ghi các trạng thái quan trọng vào một gói dữ liệu (*Bundle*). Gói dữ liệu này được gửi qua kênh liên tiến trình (*Binder*) đến tiến trình quản lý hệ điều hành.

`SavedStateHandle` là một lớp bao bọc dạng khóa giá trị giúp ViewModel đọc và ghi trực tiếp vào gói dữ liệu này. Khi người dùng mở lại app, tiến trình mới được tạo và hệ thống tự động truyền lại gói dữ liệu để khôi phục.

## Đánh đổi
Khôi phục trạng thái qua `SavedStateHandle` đảm bảo trải nghiệm người dùng liền mạch sau nhiều giờ rời khỏi ứng dụng. Khi tiến trình bị thu hồi, toàn bộ tin nhắn đang chờ trên [MessageQueue](concept:android-handler-looper-message-queue) cũng tan biến hoàn toàn.

Đổi lại, gói dữ liệu Binder bị giới hạn dung lượng nghiêm ngặt ở mức khoảng một megabyte cho mỗi tiến trình. Nếu bạn cố tình nhét danh sách lớn hoặc ảnh bitmap vào đây, ứng dụng sẽ bị sập ngay lập tức.

## Nguyên lý gốc
Nguyên lý tồn lưu trạng thái: tiến trình tính toán là tạm thời, nhưng trạng thái người dùng phải bền vững. Sự phân tách giữa tiến trình thực thi và kho lưu trữ trạng thái giúp hệ thống linh hoạt điều phối tài nguyên.

## Key insight
> ViewModel không bảo vệ dữ liệu trước việc hủy tiến trình, chỉ có SavedStateHandle mới vượt qua được ranh giới này.

## Khuôn mẫu chuẩn
```kotlin
class ProductDetailViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val productId: StateFlow<String> = savedStateHandle
        .getStateFlow("product_id", initialValue = "")

    fun updateProductId(id: String) {
        savedStateHandle["product_id"] = id
    }
}
```

## Bẫy & ranh giới
Lưu dữ liệu vượt quá giới hạn một megabyte của bộ đệm Binder sẽ kích hoạt ngoại lệ tràn giao dịch liên tiến trình. Chống chỉ định lưu toàn bộ đối tượng nghiệp vụ phức tạp vào `SavedStateHandle` thay vì chỉ lưu mã định danh. Triệu chứng gồm crash app với `TransactionTooLargeException`, mất trắng dữ liệu nhập dở hoặc tải lại trang sai lệch.

## Câu hỏi thực chiến
**Hỏi:** ViewModel có sống sót qua việc hệ điều hành tiêu diệt tiến trình (Process Death) không?
Không. Khi tiến trình bị hệ thống tiêu diệt, toàn bộ bộ nhớ RAM bao gồm ViewModel và singleton đều bị xóa sạch hoàn toàn.

**Hỏi:** Làm thế nào để phân biệt xoay màn hình với việc bị tiêu diệt tiến trình?
Dựa vào cờ trạng thái khôi phục. Khi xoay màn hình, instance của ViewModel được giữ nguyên, còn khi tiến trình tái sinh, ViewModel được tạo mới cùng `SavedStateHandle`.

**Hỏi:** Có nên lưu danh sách phân trang vào SavedStateHandle không?
Không nên. Bạn chỉ nên lưu ID hoặc số trang hiện tại để tải lại, tránh gây quá tải bộ đệm Binder 1MB.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt xoay màn hình (ViewModel sống) và Process Death (ViewModel chết).
- [ ] SavedStateHandle lưu dữ liệu qua Binder sang system_server để sống sót.
- [ ] Giới hạn bộ đệm Binder 1MB và bẫy TransactionTooLargeException.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

