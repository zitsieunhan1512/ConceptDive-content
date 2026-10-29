---
id: jvm-race-condition-memory-model
title: Race Condition & Memory Model
domain: jvm
tags: [jvm, concurrency, race-condition, memory-model, volatile, synchronized]
collections: [interview-fundamental]
related: [thread, jvm-stack-vs-heap, kotlin-stateflow-vs-sharedflow]
created: 2026-10-01
sources:
  - JLS Chapter 17: Threads and Locks | https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html
  - JSR 133 Java Memory Model FAQ | https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html
  - Java memory model — Wikipedia | https://en.wikipedia.org/wiki/Java_memory_model
diagrams: [Cơ chế|race-condition-memory-barriers.png|So sánh rào chắn volatile bảo đảm nhìn thấy, CAS phi khóa nguyên tử và synchronized chiếm monitor lock]
---

## Bản 30 giây
Khi hai luồng cùng tăng một biến đếm chung, kết quả dễ bị hụt vì chúng đọc trùng số cũ rồi ghi đè nhau. CPU còn giữ biến trong thanh ghi và đảo thứ tự lệnh nên luồng này không thấy giá trị luồng kia vừa sửa. Mô hình bộ nhớ Java đặt ra các ranh giới đồng bộ để luồng sau luôn đọc đúng dữ liệu mới.

## Cần biết trước
- thread
- jvm-stack-vs-heap

## Vấn đề buộc nó ra đời
Năm 2000, Bill Pugh chỉ ra lỗi nền tảng của mô hình bộ nhớ Java qua bài 'The Java Memory Model is Fatally Flawed'. Nhóm của ông sau đó cho thấy mẫu Double-Checked Locking lúc ấy bị hỏng hoàn toàn.

Một luồng có thể thấy con trỏ đối tượng đã khác null nhưng các trường bên trong vẫn chưa gán xong. CPU giữ biến trong thanh ghi và bộ đệm ghi (*store buffer*) nên luồng khác vẫn đọc phải giá trị cũ.

Năm 2004, quy chuẩn JSR-133 trong Java 5 viết lại mô hình bộ nhớ Java. Quy chuẩn này định nghĩa quan hệ xảy ra trước (*happens-before*) và siết lại rào chắn bộ nhớ cho từ khóa `volatile`.

## Nó thực sự là cái gì
Race condition không phải chuyện luồng nhanh hay chậm, mà là hai luồng chạm cùng ô nhớ nhưng thiếu quan hệ xảy ra trước (*happens-before*). Máy ảo tách bạch hai việc: nhìn thấy giá trị mới (*visibility*) và làm trọn gói một thao tác (*atomicity*). `volatile` chỉ giúp luồng khác thấy ngay giá trị vừa ghi, còn muốn đọc-sửa-ghi trọn gói thì phải dùng khóa hoặc lệnh phần cứng.

## Cơ chế
Khi bạn gắn `volatile`, trình biên dịch chèn rào chắn bộ nhớ (*memory barrier*) để cấm đảo lệnh qua chỗ đọc ghi biến đó. Mọi thay đổi trước lúc ghi biến `volatile` đều hiện rõ cho luồng đọc biến đó về sau. Với `synchronized`, mỗi đối tượng Java có sẵn một khóa monitor (*monitor lock*), luồng nào vào khối lệnh phải cầm được khóa này. Còn CAS dùng một lệnh CPU để so sánh và đổi giá trị ô nhớ ngay tại chỗ mà không khóa luồng.

## Đánh đổi
Khi mọi chỗ đọc ghi đều qua cùng một cơ chế đồng bộ, bạn tránh được cảnh hai luồng dẫm chân làm sai số. Đổi lại, rào chắn bộ nhớ chặn bớt việc gom và đảo lệnh của CPU nên code chạy chậm hơn. Còn dùng khóa monitor có thể làm luồng phải chờ, tốn công đổi ngữ cảnh và dễ dẫn tới khóa chéo (*deadlock*).

## Nguyên lý gốc
Muốn nhiều nhân CPU nhìn thấy cùng một giá trị, bạn phải chấp nhận đặt điểm đồng bộ và chậm lại ở điểm đó.

## Key insight
> `volatile` chỉ giúp luồng khác thấy giá trị mới; phép đọc-sửa-ghi vẫn cần khoá hoặc CAS.

## Ví von
Hai thư ký cùng nhìn một bảng thông báo ở sảnh nhưng lại chép số liệu vào sổ tay riêng tại bàn. Nếu sửa sổ riêng mà không ra sảnh chép lại lên bảng, người kia sẽ tiếp tục tính toán theo số cũ.

## Khuôn mẫu chuẩn
```kotlin
class SafeCounter {
    private val count = java.util.concurrent.atomic.AtomicInteger(0)
    fun increment(): Int = count.incrementAndGet()
    fun get(): Int = count.get()
}
val counter = SafeCounter()
val value = counter.increment()
```

## Bẫy & ranh giới
Gắn `volatile` cho biến đếm rồi gọi `count++` vẫn mất số, vì phép tăng gồm ba bước đọc, cộng và ghi tách rời. Đừng bọc lệnh đọc file hay gọi mạng trong khối `synchronized`, vì luồng giữ khóa lâu sẽ bắt mọi luồng khác đứng chờ. Cũng tránh khóa trên đối tượng công khai hoặc biến có thể bị gán lại địa chỉ mới. Khi hỏng, app sẽ tính sai số lúc tải cao, hoặc đứng hình vì hai luồng giữ khóa chéo nhau.

## Câu hỏi thực chiến
**Hỏi:** Từ khóa `volatile` có thay thế được `synchronized` khi hai luồng cùng tăng một biến đếm không?
Không. `volatile` chỉ bảo đảm luồng khác đọc được giá trị mới nhất, chứ không gom ba bước đọc, cộng và ghi thành một bước. Hai luồng vẫn có thể cùng đọc số 5, cùng cộng lên 6 rồi ghi đè nhau, làm mất một lần tăng.

**Hỏi:** Nguyên lý quan hệ xảy ra trước (*happens-before*) trong Java Memory Model nghĩa là gì?
Đó là cam kết: nếu bước A xảy ra trước bước B, mọi thứ A ghi vào bộ nhớ đều hiện rõ cho B. Ví dụ việc một luồng nhả khóa monitor luôn xảy ra trước việc luồng kế tiếp cầm được chính khóa đó.

**Hỏi:** Vì sao các lớp `AtomicInteger` hay `AtomicReference` lại chạy nhanh hơn dùng khối `synchronized`?
Vì chúng dùng lệnh phần cứng CAS để đổi giá trị trực tiếp mà không khóa luồng (*lock-free*). Khi ít đụng độ, luồng không bị hệ điều hành cho ngủ và đánh thức lại, nên đỡ tốn chi phí đổi ngữ cảnh.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được tính nhìn thấy (visibility) và tính nguyên tử (atomicity) khi nhiều luồng cùng sửa biến.
- [ ] Giải thích được vì sao thanh ghi, bộ đệm ghi (store buffer) và việc đảo lệnh làm luồng đọc phải số cũ.
- [ ] So sánh được lệnh phần cứng CAS của AtomicInteger với khóa monitor của khối synchronized.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
