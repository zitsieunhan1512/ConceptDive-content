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
Một luồng bình thường sẽ tự kết thúc và biến mất ngay sau khi chạy xong dòng lệnh cuối cùng trong hàm `run()`. Android giữ cho luồng chính sống suốt lúc mở ứng dụng nhờ vòng lặp `Looper` liên tục rút việc từ hàng đợi `MessageQueue`. Còn `Handler` là đầu mối để các luồng phụ gửi công việc về hàng đợi của luồng chính một cách an toàn.

## Cần biết trước
- thread
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Hệ thống View của Android không an toàn khi truy cập đa luồng (*Not Thread-Safe*). Nếu hai luồng cùng lúc sửa một nút bấm, giao diện sẽ hiển thị sai lệch hoặc sập ứng dụng ngay.

Để tránh việc phải đặt khóa đồng bộ ở mọi View, Google quy định chỉ một luồng duy nhất được phép sửa giao diện. Mọi lệnh vẽ màn hình và xử lý chạm tay đều phải đưa về luồng chính để chạy lần lượt.

Bộ ba `Handler`, `Looper` và `MessageQueue` có mặt từ bản Android đầu tiên để chuyển việc giữa các luồng. Nhờ đó, luồng phụ tải dữ liệu xong có thể gửi kết quả về luồng chính để hiển thị.

## Nó thực sự là cái gì
`Looper` không phải là một vòng lặp `while(true)` chạy rỗng liên tục làm tốn pin và nóng máy. Thực ra, bên dưới nó dùng lệnh `epoll_wait` của nhân Linux để cho luồng tạm ngủ khi rảnh.

Khi hàng đợi `MessageQueue` không có việc nào cần chạy, luồng chính sẽ ngủ yên và nhường CPU cho việc khác. Ngay khi bạn chạm vào màn hình hoặc có tin nhắn mới gửi tới, nhân Linux mới đánh thức luồng dậy làm việc.

## Cơ chế
Mỗi `Thread` chỉ gắn được với tối đa một `Looper`, được cất riêng trong vùng nhớ `ThreadLocal` của luồng đó. Hàng đợi `MessageQueue` xếp các gói tin nhắn theo thứ tự thời điểm cần chạy từ sớm tới muộn.

Vòng lặp `Looper.loop()` lần lượt lấy từng tin nhắn ra và giao cho đúng `Handler` đã gửi tin đó xử lý. Khi vẽ khung hình mới, hệ thống cắm chốt chặn (*Sync Barrier*) vào hàng đợi để ưu tiên chạy lệnh vẽ trước.

## Đánh đổi
Chạy từng việc một trên một hàng đợi giúp cây giao diện không bao giờ bị hai luồng tranh chấp dữ liệu (*Race Condition*). Đổi lại, nếu một công việc ở đầu hàng chạy quá lâu, mọi việc xếp phía sau đều phải đứng chờ.

Nếu giữ luồng chính quá 5 giây làm kẹt sự kiện chạm, hệ thống sẽ báo lỗi ứng dụng không phản hồi.

## Nguyên lý gốc
Giữ luồng sống lâu bằng một vòng lặp rút việc từ hàng đợi, và tách riêng luồng tính toán nặng khỏi luồng vẽ giao diện.

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
Tạo `Handler` ẩn danh trong `Activity` rồi gửi tin hẹn giờ sẽ giữ chặt `Activity` và gây [rò rỉ bộ nhớ](concept:android-memory-leak). Khi ứng dụng dưới nền gặp cảnh [tiêu diệt tiến trình](concept:android-process-death-and-saved-state), mọi tin nhắn đang chờ trong `MessageQueue` cũng mất sạch. Tuyệt đối không đẩy việc đọc file lớn hay truy vấn cơ sở dữ liệu vào `Handler` của luồng chính. Triệu chứng dính bẫy là rò rỉ bộ nhớ, giật khung hình và [lỗi ứng dụng không phản hồi (*ANR*)](concept:android-main-thread-and-anr).

## Câu hỏi thực chiến
**Hỏi:** Tại sao vòng lặp vô tận Looper.loop() trên Main Thread không làm treo hay nóng máy điện thoại?
Vì bên dưới nó gọi hàm `epoll_wait` của nhân Linux chứ không chạy vòng lặp rỗng. Khi `MessageQueue` hết tin nhắn cần chạy ngay, luồng chính tự ngủ và nhường hoàn toàn CPU cho các luồng khác.

**Hỏi:** Handler tạo trên luồng phụ có thể cập nhật giao diện trực tiếp được không?
Không được, vì `Handler` gửi việc vào hàng đợi của `Looper` nào mà nó được gắn lúc khởi tạo. Muốn gửi việc từ luồng phụ về sửa giao diện, bạn phải truyền `Looper.getMainLooper()` vào lúc tạo `Handler`.

**Hỏi:** Rào cản đồng bộ (Sync Barrier) trong MessageQueue có tác dụng gì?
Khi đến nhịp vẽ màn hình, hệ thống thả chốt `Sync Barrier` vào hàng đợi để tạm gác các tin nhắn thường sang một bên. Nhờ đó, tin nhắn vẽ khung hình được chạy vượt lên trước, giúp màn hình giữ được tốc độ 60 hoặc 120 hình mỗi giây.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao Looper.loop() không làm tốn CPU nhờ cơ chế ngủ epoll_wait của nhân Linux.
- [ ] Trình bày được cách Handler gắn với Looper qua ThreadLocal để gửi việc về đúng hàng đợi.
- [ ] Chỉ ra được bẫy Handler ẩn danh giữ tham chiếu Activity qua tin nhắn hẹn giờ gây rò rỉ bộ nhớ.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

