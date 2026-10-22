---
id: android-handler-looper-message-queue
title: Handler, Looper & MessageQueue
domain: android
tags: [android, looper, handler, message-queue, main-thread, epoll]
collections: [interview-fundamental]
related: [thread, android-activity-lifecycle, kotlin-coroutines, android-process-death-and-saved-state]
created: 2026-09-15
diagrams: [Cơ chế|handler-looper-message-queue-loop.png|Handler gửi tin vào MessageQueue có rào cản đồng bộ, Looper liên tục lấy tin hoặc ngủ đông bằng epoll_wait nhường CPU]
---

## Bản 30 giây
Một luồng hệ điều hành thông thường sẽ lập tức kết thúc và giải phóng tài nguyên sau khi chạy hết lệnh. Android giữ cho luồng giao diện sống sót liên tục nhờ vòng lặp `Looper` và hàng đợi `MessageQueue` xử lý tin nhắn. `Handler` đóng vai trò sứ giả để các luồng phụ gửi công việc về luồng chính một cách an toàn.

## Cần biết trước
- thread
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Hệ thống hiển thị của Android không an toàn đa luồng (*Not Thread-Safe*). Nhiều luồng cùng lúc thao tác lên cây giao diện sẽ gây xé hình và xung đột trạng thái nghiêm trọng.

Để ngăn ngừa thảm họa này, Google áp dụng nguyên tắc truy cập đơn luồng cho toàn bộ hệ thống View. Mọi lệnh cập nhật màn hình bắt buộc phải chuyển giao về luồng chính duy nhất xử lý tuần tự.

Bộ ba `Handler`, `Looper` và `MessageQueue` ra đời từ phiên bản Android đầu tiên để giải quyết bài toán giao tiếp giữa các luồng. Chúng thiết lập một đường ray truyền tin nhắn bền bỉ giữa các tiến trình.

## Nó thực sự là cái gì
`Looper` KHÔNG phải là một vòng lặp `while(true)` chạy rỗng liên tục gây đốt cháy năng lượng CPU. Thực chất, đây là một cơ chế ngủ đông dựa trên lời gọi hệ thống `epoll_wait` của nhân Linux.

Khi hàng đợi hết việc, luồng chính tự động nhường CPU và chỉ thức dậy khi có tệp mô tả sự kiện phát tín hiệu.

## Cơ chế
Mỗi `Thread` chỉ có thể gắn với duy nhất một đối tượng `Looper` thông qua biến cục bộ luồng (*ThreadLocal*). `MessageQueue` lưu trữ các thông điệp và gói lệnh theo thứ tự mốc thời gian thực thi.

Vòng lặp `Looper.loop()` liên tục lấy thông điệp từ hàng đợi và gọi hàm xử lý của `Handler` đích. Rào cản đồng bộ (*Sync Barrier*) được dùng để ưu tiên các thông điệp vẽ giao diện trước các tác vụ thông thường.

## Đánh đổi
Cơ chế hàng đợi tuần tự loại bỏ hoàn toàn các lỗi xung đột khóa (*Race Condition*) trên cây giao diện. Đổi lại, nếu một tác vụ trong hàng đợi xử lý quá lâu, toàn bộ các tác vụ phía sau sẽ bị phong tỏa.

Nếu luồng chính bị phong tỏa khiến sự kiện tương tác chờ quá năm giây, hệ thống sẽ báo lỗi không phản hồi.

## Nguyên lý gốc
Nguyên lý vòng lặp sự kiện: giữ tiến trình sống sót bằng cách tuần tự hóa mọi kích thích từ môi trường. Sự phân chia giữa luồng tính toán nặng và luồng điều phối giao diện đảm bảo độ mượt mà của hệ thống tương tác.

## Key insight
> Looper không làm nóng máy nhờ lệnh chờ epoll, và Handler chỉ là cổng gửi nhận chứ không sở hữu luồng.

## Khuôn mẫu chuẩn
```kotlin
class WorkerThread : Thread() {
    lateinit var handler: Handler
    override fun run() {
        Looper.prepare()
        handler = Handler(Looper.myLooper()!!) { msg ->
            // Xử lý thông điệp trên luồng nền
            true
        }
        Looper.loop()
    }
}
```

## Bẫy & ranh giới
Tạo `Handler` nặc danh không tĩnh trong Activity sẽ ngầm giữ tham chiếu đến Context, gây [rò rỉ bộ nhớ](concept:android-memory-leak) nghiêm trọng. Khi hệ thống gặp biến cố [tiêu diệt tiến trình](concept:android-process-death-and-saved-state), các tin nhắn trong hàng đợi cũng bị hủy bỏ. Chống chỉ định đẩy các tác vụ giải mã file hoặc gọi cơ sở dữ liệu vào hàng đợi luồng chính. Triệu chứng lỗi gồm rò rỉ bộ nhớ qua chuỗi tin nhắn chờ, giật khung hình và [lỗi ứng dụng không phản hồi (*ANR*)](concept:android-main-thread-and-anr).

## Câu hỏi thực chiến
**Hỏi:** Tại sao vòng lặp vô tận Looper.loop() trên Main Thread không làm treo hay nóng máy điện thoại?
Không nóng máy vì nó sử dụng cơ chế `epoll_wait` của Linux. Khi không có tin nhắn trong hàng đợi, luồng chính đi vào trạng thái ngủ đông và nhường CPU cho tiến trình khác.

**Hỏi:** Handler tạo trên luồng phụ có thể cập nhật giao diện trực tiếp được không?
Không được. Handler gửi công việc vào hàng đợi của Looper gắn liền với nó; muốn sửa giao diện, bạn phải truyền `Looper.getMainLooper()` vào hàm khởi tạo.

**Hỏi:** Rào cản đồng bộ (Sync Barrier) trong MessageQueue có tác dụng gì?
Nó tạm chặn các tin nhắn thông thường để ưu tiên tín hiệu vẽ màn hình. Nhờ đó, khung hình 60 hay 120 FPS luôn được xử lý đúng hạn.

## Ý bắt buộc khi giải thích lại
- [ ] Looper.loop() chạy dựa trên lệnh ngủ epoll_wait của nhân Linux.
- [ ] Handler gắn liền với Looper thông qua ThreadLocal để định tuyến tin nhắn.
- [ ] Bẫy Handler nặc danh giữ tham chiếu Activity gây rò rỉ bộ nhớ.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

