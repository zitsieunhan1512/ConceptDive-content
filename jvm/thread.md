---
id: thread
title: Thread
domain: jvm
tags: [jvm, thread, concurrency, operating-system, memory]
collections: []
related: [kotlin-coroutines, kotlin-coroutine-state-machine]
created: 2026-09-09
sources:
  - Thread API reference — Java SE | https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html
  - Java launcher options (-Xss) — Oracle | https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
  - Thread (computing) — Wikipedia | https://en.wikipedia.org/wiki/Thread_(computing)
diagrams: [Cơ chế|thread-context-switch.png|Java Thread ánh xạ 1:1 sang Linux pthread, bộ lập lịch cấp time slice và tổn thất khi hoán đổi ngữ cảnh]
---

## Bản 30 giây
Khi ứng dụng vừa tải ảnh vừa vẽ màn hình, máy tính phải chia việc cho các luồng (`Thread`) chạy cùng lúc. Mỗi `Thread` là một đường chạy riêng do nhân hệ điều hành trực tiếp cấp phát và chia lượt dùng CPU. Đổi lại, mỗi luồng giữ một ngăn xếp riêng và làm CPU tốn công cất rồi nạp lại trạng thái mỗi lần đổi lượt.

## Cần biết trước

## Vấn đề buộc nó ra đời
Thập niên 1960, hệ điều hành chia sẻ thời gian (time-sharing) ra đời để nhiều người cùng dùng chung một máy tính lớn. Mỗi tiến trình (process) lúc đó có vùng nhớ tách biệt hẳn nhau. Muốn truyền dữ liệu qua lại giữa hai tiến trình thì rất chậm và rườm rà.

Năm 1995, Java 1.0 đưa luồng vào ngôn ngữ qua lớp `Thread`. Thay vì phải tạo tiến trình mới nặng nề, bạn cho nhiều luồng chạy chung trong một tiến trình và đọc cùng vùng nhớ Heap.

Thế nhưng nếu bạn cứ tạo luồng mới thoải mái cho mỗi việc nhỏ, hệ điều hành sẽ nhanh chóng hết chỗ quản lý. Tạo hàng trăm luồng chỉ làm CPU tốn thời gian chuyển qua lại giữa các luồng thay vì chạy code của bạn.

## Nó thực sự là cái gì
`Thread` trong JVM không phải là luồng nhẹ do máy ảo tự giả lập. Trên cả máy ảo HotSpot lẫn Android ART, mỗi đối tượng `Thread` bạn chạy đều gắn 1:1 với một luồng nhân `pthread` thật của Linux.

## Cơ chế
Khi bạn gọi `start()`, máy ảo gọi xuống thư viện C để nhờ nhân Linux tạo một luồng `pthread` thật. Nhân giữ trước vùng địa chỉ ảo (mặc định 1MB trên HotSpot 64-bit) làm ngăn xếp gọi hàm (`call stack`). Bộ lập lịch của hệ điều hành chia cho mỗi luồng từng khoảng thời gian ngắn (`time slice`) để vào chạy trên nhân CPU. Khi đổi luồng, CPU cất thanh ghi cũ, nạp ngữ cảnh mới và chậm đi vì `cache` mất dữ liệu quen.

## Đánh đổi
Dùng nhiều `Thread` giúp bạn tận dụng hết các nhân vật lý của chip CPU để tính toán song song thật sự. Đổi lại, mỗi luồng chiếm trước một vùng địa chỉ ảo và tốn thêm bộ nhớ quản lý trong nhân hệ điều hành. Nếu bạn mở số luồng nhiều hơn số nhân CPU, máy sẽ tốn rất nhiều công chuyển ngữ cảnh (`context switch`) qua lại.

## Nguyên lý gốc
Một nhân CPU chia thời gian thành các lát cực nhỏ rồi luân phiên cho từng luồng vào chạy (Time-sharing).

## Key insight
> Tạo thêm luồng không làm CPU tính nhanh hơn: vượt quá số nhân vật lý, máy chỉ tốn công hoán đổi ngữ cảnh.

## Ví von
Một đầu bếp phụ trách nấu nhiều món cùng lúc trên các bếp khác nhau. Mỗi lần đổi món, đầu bếp phải đặt dao xuống, cầm đũa lên và nhớ lại mình đang nấu dở gia vị nào.

## Khuôn mẫu chuẩn
```kotlin
val thread = Thread {
    println("Running on: ${Thread.currentThread().name}")
}.apply {
    name = "Worker-Thread"
    isDaemon = false
    start()
}
thread.join()
```

## Bẫy & ranh giới
Bẫy nguy hiểm nhất là bạn gọi `Thread().start()` ngay trong vòng lặp hoặc mỗi khi bấm nút gọi mạng. Mỗi luồng giữ trước 1MB địa chỉ ảo (HotSpot 64-bit), nên tạo vài nghìn luồng sẽ chạm trần Linux và sập `OutOfMemoryError`. Bẫy thứ hai là mở quá nhiều luồng cùng tính toán nặng, làm CPU mất phần lớn thời gian chỉ để đổi lượt chạy. Đừng tự tạo `Thread` lẻ tẻ; hãy dùng sẵn `ThreadPool` có giới hạn hoặc chuyển sang [Coroutines](concept:kotlin-coroutines).

## Câu hỏi thực chiến
**Hỏi:** Tạo một Thread mới trong Java có tốn ngay 1MB RAM vật lý không?
Không. Máy ảo chỉ xin Linux giữ chỗ trên địa chỉ ảo, còn RAM thật cấp dần khi hàm gọi sâu xuống. Vì vậy tạo quá nhiều luồng thường sập do chạm trần số luồng Linux hoặc cạn địa chỉ ảo trước RAM.

**Hỏi:** Đã có Coroutines rồi thì khi nào vẫn bắt buộc phải tự quản lý Thread?
Khi bạn gọi xuống thư viện C/C++ qua JNI hoặc chạy các hàm chặn cứng luồng không có chỗ ngắt. Hàm C không hiểu cơ chế tạm dừng của [Coroutines](concept:kotlin-coroutines). Bạn phải nhốt chúng vào luồng riêng để khỏi kẹt nhóm luồng chung.

**Hỏi:** `Thread.sleep()` khác gì `Thread.yield()` và coroutine `delay()`?
Khác nhau ở chỗ có trả luồng cho việc khác dùng hay không. Gọi `sleep()` là ôm luôn luồng đó đứng chờ, còn `yield()` là nhường lượt chạy hiện tại cho luồng khác. Riêng `delay()` thì trả hẳn luồng về cho nhóm luồng chung để hàm khác nhảy vào chạy trong lúc chờ.

**Hỏi:** Đặt số lượng luồng trong ThreadPool bằng bao nhiêu là tối ưu?
Tùy vào việc của bạn đang bắt CPU tính toán liên tục hay chỉ ngồi chờ mạng và ổ đĩa. Nếu tính toán nặng, số luồng chỉ nên đặt bằng số nhân CPU vật lý để khỏi tốn công chuyển ngữ cảnh. Nếu chủ yếu chờ đọc ghi I/O, bạn đặt nhiều luồng hơn được vì đa số thời gian các luồng chỉ nằm đợi.

**Hỏi:** Vì sao dừng Thread bằng `stop()` bị cấm mà phải dùng `interrupt()`?
Vì `stop()` giết luồng ngay lập tức và buông hết các khóa (`lock`) mà luồng đó đang giữ. Đối tượng đang sửa dở bên trong khóa sẽ bị bỏ lại ở trạng thái sai lệch mà các luồng khác không hề biết. Còn `interrupt()` chỉ bật một lá cờ báo hiệu để luồng đang chạy tự kiểm tra, đóng file gọn gàng rồi mới thoát.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được mỗi Thread trên JVM và ART gắn 1:1 với một luồng pthread thật của nhân Linux.
- [ ] Nêu được máy ảo HotSpot mặc định giữ trước 1MB địa chỉ ảo trên hệ 64-bit cho call stack, chưa tốn ngay RAM vật lý.
- [ ] Trình bày được chi phí khi đổi luồng: cất và nạp lại thanh ghi, kèm việc bộ nhớ đệm CPU bị mất dữ liệu đang dùng.
- [ ] Chỉ ra được bẫy tạo Thread trong vòng lặp làm chạm trần số luồng của hệ điều hành và ném OutOfMemoryError.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

