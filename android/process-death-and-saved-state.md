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
Khi bạn rời ứng dụng sang dùng app khác, Android có thể âm thầm tắt tiến trình chạy ngầm để lấy lại RAM. Lúc đó dữ liệu trong ViewModel mất sạch, chỉ những giá trị bạn cất vào `SavedStateHandle` mới được giữ lại. Nhờ vậy, khi mở lại ứng dụng, màn hình vẫn hiện đúng trạng thái cũ dù tiến trình đã khởi động lại.

## Cần biết trước
- android-activity-lifecycle
- android-viewmodel
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2008, Android ra đời trên những chiếc điện thoại có dung lượng RAM rất ít. Để chạy đa nhiệm, hệ thống phải tắt tiến trình nền nhằm nhường bộ nhớ cho ứng dụng trước mặt.

Nhiều bạn mới đi làm tưởng ViewModel giữ được dữ liệu mãi mãi, nhưng thật ra nó chỉ sống qua lúc xoay màn hình. Khi tiến trình bị tắt ngầm, người dùng mở lại app sẽ thấy màn hình trắng trơn hoặc ứng dụng bị sập vì biến rỗng.

Google đưa ra `SavedStateHandle` để nối thẳng kho lưu trạng thái của hệ thống vào trong ViewModel. Bạn không còn phải tự viết code đóng gói dữ liệu rườm rà trong các hàm vòng đời của Activity.

## Nó thực sự là cái gì
`Process Death` không phải là lỗi tràn bộ nhớ (`OutOfMemory`) do code của bạn bị rò rỉ. Thật ra, nhân Linux chủ động tắt tiến trình chạy ngầm dựa vào điểm ưu tiên `oom_score_adj` khi máy thiếu RAM. Trước khi tắt, hệ thống chép dữ liệu trạng thái của bạn sang giữ ở tiến trình hệ thống `system_server`.

## Cơ chế
Khi bạn vừa ẩn ứng dụng xuống nền, hệ điều hành bảo Activity gom các trạng thái cần giữ vào một gói `Bundle`. Gói dữ liệu này được gửi qua kênh liên tiến trình `Binder` sang nằm nhờ ở tiến trình `system_server`. `SavedStateHandle` chỉ là một bảng khóa - giá trị trong ViewModel giúp bạn đọc và ghi thẳng vào gói `Bundle` đó. Lúc bạn mở lại app, Android tạo tiến trình mới rồi trả gói `Bundle` này về để `SavedStateHandle` nạp lại dữ liệu cũ.

## Đánh đổi
Dùng `SavedStateHandle` giúp bạn giữ nguyên màn hình đang dùng dở dù người dùng đã rời app hàng giờ liền. Khi tiến trình bị tắt, mọi tin nhắn đang xếp hàng trên [MessageQueue](concept:android-handler-looper-message-queue) của luồng chính cũng mất sạch.

Đổi lại, kênh truyền `Binder` giới hạn tổng dung lượng gói dữ liệu ở mức khoảng một megabyte cho cả tiến trình. Nếu bạn cố nhét danh sách dài hay ảnh bitmap vào đây, ứng dụng sẽ sập ngay lúc người dùng bấm nút Home.

## Nguyên lý gốc
Tiến trình có thể bị tắt bất cứ lúc nào. Vì vậy, trạng thái màn hình phải được cất ở nơi sống lâu hơn tiến trình đó.

## Key insight
> ViewModel mất sạch khi tiến trình bị tắt ngầm, chỉ dữ liệu gửi qua `SavedStateHandle` sang `system_server` mới còn sống.

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
Nếu bạn cất dữ liệu vượt ngưỡng một megabyte của bộ đệm `Binder`, giao dịch liên tiến trình sẽ bị tràn và ném lỗi ngay. Đừng bao giờ nhét cả đối tượng dữ liệu lớn vào `SavedStateHandle`, mà chỉ nên lưu mã định danh hoặc từ khóa đang tìm kiếm. Triệu chứng khi mắc bẫy là app sập với lỗi `TransactionTooLargeException`, mất sạch chữ vừa gõ dở, hoặc mở lại ra sai trang.

## Câu hỏi thực chiến
**Hỏi:** ViewModel có sống sót qua việc hệ điều hành tiêu diệt tiến trình (Process Death) không?
Không. Khi hệ thống tắt tiến trình, toàn bộ vùng nhớ RAM của app gồm cả ViewModel lẫn các biến singleton đều bị xóa sạch.

**Hỏi:** Làm thế nào để phân biệt xoay màn hình với việc bị tiêu diệt tiến trình?
Khi bạn xoay màn hình, đối tượng ViewModel cũ vẫn nằm nguyên trên RAM và chỉ nối lại vào Activity mới. Còn sau `Process Death`, hệ thống phải tạo một đối tượng ViewModel mới tinh rồi nạp lại dữ liệu từ `SavedStateHandle`.

**Hỏi:** Có nên lưu danh sách phân trang vào SavedStateHandle không?
Không nên. Bạn chỉ cần lưu mã ID hoặc số trang đang xem vào `SavedStateHandle` để lúc mở lại thì gọi lấy lại dữ liệu. Cách này giúp bạn không làm tràn bộ đệm 1MB của `Binder`.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được xoay màn hình (ViewModel còn sống) và Process Death (ViewModel bị xóa cùng tiến trình).
- [ ] Giải thích được cách SavedStateHandle gửi Bundle qua Binder sang tiến trình system_server để giữ dữ liệu.
- [ ] Nêu được giới hạn 1MB của bộ đệm Binder và lỗi TransactionTooLargeException khi lưu dữ liệu quá lớn.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

