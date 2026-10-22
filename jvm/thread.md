---
id: thread
title: Thread
domain: jvm
tags: [jvm, thread, concurrency, operating-system, memory]
collections: []
related: [kotlin-coroutines, kotlin-coroutine-state-machine]
created: 2026-09-09
diagrams: [Cơ chế|thread-context-switch.png|Java Thread ánh xạ 1:1 sang Linux pthread, bộ lập lịch cấp time slice và tổn thất khi hoán đổi ngữ cảnh]
---

## Bản 30 giây
Máy tính cần thực thi nhiều tác vụ đồng thời mà không làm gián đoạn nhau. Thread là đơn vị thực thi nhỏ nhất do nhân hệ điều hành trực tiếp lập lịch và cấp phát tài nguyên. Mỗi luồng đòi hỏi một vùng ngăn xếp riêng và tốn chi phí hoán đổi bộ nhớ mỗi khi CPU chuyển đổi qua lại.

## Cần biết trước

## Vấn đề buộc nó ra đời
Thập niên 1960, các hệ điều hành chia sẻ thời gian (time-sharing) ra đời để phục vụ nhiều người dùng trên một máy tính lớn. Tiến trình (process) thuở đầu có bộ nhớ hoàn toàn độc lập. Việc chia sẻ dữ liệu giữa chúng đòi hỏi cơ chế giao tiếp liên tiến trình chậm chạp.

Năm 1995, Java 1.0 ra đời và đưa luồng vào ngôn ngữ qua lớp `Thread`. Thay vì quản lý tiến trình nặng nề, Java cho phép nhiều luồng cùng chia sẻ chung vùng nhớ Heap.

Tuy nhiên, lạm dụng tạo luồng tự do nhanh chóng đẩy hệ thống vào bế tắc. Tạo hàng trăm luồng khiến hệ điều hành cạn kiệt bảng quản lý, còn CPU lãng phí tài nguyên để hoán đổi qua lại.

## Nó thực sự là cái gì
Thread trong JVM KHÔNG phải một thực thể ảo do ngôn ngữ tự vận hành. Thực chất, trên cả HotSpot lẫn Android ART, mỗi `Thread` của Java ánh xạ trực tiếp 1:1 sang một luồng nhân `pthread` của Linux.

## Cơ chế
Khi gọi `start()`, máy ảo gọi hàm thư viện C để yêu cầu kernel tạo một `pthread` tương ứng. Kernel cấp phát trước không gian địa chỉ ảo (mặc định 1MB trên máy ảo 64-bit) làm ngăn xếp gọi hàm (call stack). Bộ lập lịch của hệ điều hành định kỳ phân bổ các lát cắt thời gian (time slice) để luồng chạy trên nhân CPU. Khi chuyển luồng, CPU phải lưu trữ thanh ghi, nạp ngữ cảnh mới và chịu tổn thất do bộ nhớ đệm (cache) bị nguội lạnh.

## Đánh đổi
Thread mang lại khả năng tận dụng tối đa các nhân phần cứng CPU để xử lý song song. Đổi lại, mỗi luồng ngốn trước địa chỉ ảo và tiêu hao bộ nhớ kernel. Chi phí hoán đổi ngữ cảnh (context switch) tăng vọt khi số luồng vượt quá năng lực xử lý vật lý của chip.

## Nguyên lý gốc
Nguyên lý chia sẻ thời gian (Time-sharing): luân chuyển quyền sử dụng phần cứng qua các lát cắt thời gian siêu nhỏ.

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
Bẫy nguy hiểm nhất là khởi tạo `Thread` trực tiếp trong vòng lặp hoặc sau mỗi yêu cầu mạng. Mỗi luồng chiếm trước 1MB địa chỉ ảo, làm ứng dụng chạm trần giới hạn luồng của Linux và sập `OutOfMemoryError`. Bẫy thứ hai là chạy quá nhiều luồng tính toán, khiến CPU mất phần lớn hiệu năng chỉ để hoán đổi ngữ cảnh. Chống chỉ định tự tạo luồng vô tội vạ; hãy dùng thread pool có giới hạn hoặc chuyển sang [Coroutines](concept:kotlin-coroutines).

## Câu hỏi thực chiến
**Hỏi:** Tạo một Thread mới trong Java có tốn ngay 1MB RAM vật lý không?
Không. Máy ảo chỉ xin nhân Linux cấp không gian địa chỉ ảo, còn RAM vật lý chỉ tốn khi các hàm gọi sâu xuống. Vì vậy ứng dụng thường sập do hết địa chỉ ảo hoặc chạm trần luồng của hệ điều hành trước khi cạn RAM.

**Hỏi:** Đã có Coroutines rồi thì khi nào vẫn bắt buộc phải tự quản lý Thread?
Khi bạn gọi thư viện C/C++ qua JNI hoặc chạy mã chặn luồng tầng thấp không thể ngắt. Các hàm C cũ không có cơ chế tạm dừng của [Coroutines](concept:kotlin-coroutines), nên cần giữ riêng thread để tránh làm nghẽn pool chung.

**Hỏi:** `Thread.sleep()` khác gì `Thread.yield()` và coroutine `delay()`?
Khác nhau ở việc giữ chặt hay nhả luồng. Hàm `sleep()` giữ chặt luồng và đứng đợi, còn `yield()` nhường ngay lát cắt CPU cho luồng khác. Ngược lại, `delay()` giải phóng luồng hoàn toàn để việc khác chạy trong lúc chờ.

**Hỏi:** Đặt số lượng luồng trong ThreadPool bằng bao nhiêu là tối ưu?
Tuỳ vào tác vụ ngốn CPU hay chờ mạng và ổ đĩa. Với tác vụ tính toán nặng, số luồng tối đa chỉ nên bằng số nhân CPU vật lý để tránh hoán đổi ngữ cảnh. Với tác vụ chờ I/O, bạn có thể tăng số luồng lên cao hơn vì phần lớn thời gian luồng nằm đợi dữ liệu.

**Hỏi:** Vì sao dừng Thread bằng `stop()` bị cấm mà phải dùng `interrupt()`?
Vì `stop()` giết luồng đột ngột và tự động nhả mọi khóa bảo vệ. Hành động này khiến các đối tượng dùng chung rơi vào trạng thái hỏng nửa chừng mà không ai hay biết. Ngược lại, `interrupt()` chỉ bật cờ báo hiệu để luồng tự kiểm tra, dọn dẹp tài nguyên và chủ động rút lui.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: Thread trên JVM/ART ánh xạ 1:1 sang pthread của nhân Linux, không phải luồng xanh (green thread).
- [ ] Mỗi thread chiếm trước địa chỉ ảo (mặc định 1MB trên 64-bit) cho call stack, không phải RAM vật lý ngay lúc tạo.
- [ ] Tổn thất hoán đổi ngữ cảnh: kernel lưu khôi phục thanh ghi và làm nguội lạnh bộ nhớ đệm CPU (cold cache).
- [ ] Bẫy khởi tạo luồng trực tiếp gây cạn kiệt threads-max và tràn bộ nhớ OutOfMemoryError.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

