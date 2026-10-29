---
id: kotlin-flow-under-the-hood
title: Flow: Bản chất Cold Stream
domain: kotlin
tags: [kotlin, flow, coroutines, reactive, cold-stream, backpressure]
collections: [interview-fundamental]
related: [kotlin-coroutines, kotlin-stateflow-vs-sharedflow, kotlin-coroutine-context-and-dispatchers]
sources:
  - Asynchronous Flow — Kotlin Documentation | https://kotlinlang.org/docs/coroutines-flow.html
  - Flow API reference — kotlinx.coroutines | https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/
  - Kotlin flows on Android | https://developer.android.com/kotlin/flow
diagrams: [Cơ chế|flow-flowon-channel.png|flowOn tách luồng phát và luồng nhận thành hai coroutine độc lập nối qua Channel buffer để bảo toàn ngữ cảnh]
created: 2026-09-15
---

## Bản 30 giây
Khi đọc cảm biến hay cơ sở dữ liệu liên tục, chạy sẵn lúc chưa ai xem sẽ rất tốn pin. `Flow` là luồng dữ liệu nguội (`Cold Stream`), nghĩa là khối code bên trong chỉ bắt đầu chạy khi có nơi gọi `collect`. Do `emit` và `collect` đều là hàm `suspend`, bên phát tự chờ khi bên nhận chậm mà khỏi cấu hình bộ đệm.

## Cần biết trước
- kotlin-coroutines
- kotlin-stateflow-vs-sharedflow
- kotlin-coroutine-context-and-dispatchers

## Vấn đề buộc nó ra đời
Trước năm 2019, lập trình viên Android thường dùng RxJava để xử lý các luồng dữ liệu phát liên tục (`Reactive Streams`). RxJava rất mạnh nhưng thư viện lớn, nhiều kiểu dữ liệu và toán tử nên khó học và khó lần dấu lỗi.

Khi bên phát bắn dữ liệu nhanh hơn tốc độ bên nhận xử lý, dữ liệu bị ứ đọng gọi là áp lực ngược (`Backpressure`). Bạn phải tự chọn giữ trong bộ đệm hay bỏ bớt. Nếu quên, ứng dụng rất dễ sập vì tràn bộ nhớ.

Đội ngũ Kotlin tạo ra `Flow` gọn nhẹ ngay trên nền Coroutine. Nhờ cơ chế tạm dừng của hàm `suspend`, bên phát tự chờ bên nhận xử lý xong mới phát tiếp. Do đó, bạn không cần bộ điều tiết cồng kềnh.

## Nó thực sự là cái gì
`Flow` không phải là một luồng đang chạy ngầm sẵn hay một danh sách chứa sẵn dữ liệu trong bộ nhớ. Thật ra, nó chỉ là một khối lệnh `suspend` được gói bên trong một giao diện có đúng một hàm `collect`.

Mỗi lần một màn hình gọi `collect`, toàn bộ khối lệnh bên trong `flow { ... }` mới bắt đầu chạy lại từ dòng đầu tiên.

## Cơ chế
Khi khối `flow` gọi `emit(value)`, nó gọi thẳng vào hàm của `collect` và tạm dừng chờ bên nhận chạy xong. Để giữ đúng ngữ cảnh (`Context Preservation`), `Flow` cấm bạn tự ý bọc `withContext` đổi luồng ngay bên trong khối `flow { ... }`.

Để đọc dữ liệu trên luồng khác, bạn gắn `flowOn` nhằm tạo kênh `Channel` nối giữa hai coroutine riêng. Nhờ vậy, coroutine phát và coroutine nhận chạy trên hai [`Dispatcher`](concept:kotlin-coroutine-context-and-dispatchers) khác nhau mà vẫn an toàn.

## Đánh đổi
Là luồng nguội, `Flow` không tốn pin hay RAM khi chưa gọi `collect`, và tự dừng khi coroutine bị hủy. Đổi lại, nếu 3 nơi cùng gọi `collect` trên một `Flow` gọi API, khối lệnh sẽ gọi mạng 3 lần riêng.

Ngoài ra, mỗi lần bạn dùng `flowOn` để đổi luồng, hệ thống phải tạo thêm `Channel` và đối tượng trung gian trên Heap.

## Nguyên lý gốc
Chỉ tạo dữ liệu khi có nơi yêu cầu lấy. Bên phát phải đi theo đúng tốc độ của bên nhận.

## Key insight
> `Flow` không tự chạy ngầm, mà chỉ là một công thức đóng gói sẵn, mỗi lần gọi `collect` mới bắt đầu chạy từ đầu.

## Khuôn mẫu chuẩn
```kotlin
fun fetchSensorData(): Flow<SensorEvent> = flow {
    while (currentCoroutineContext().isActive) {
        val event = readHardwareSensor()
        emit(event)
    }
}.flowOn(Dispatchers.IO)

// Thu thập tại tầng UI hoặc ViewModel:
viewModelScope.launch {
    fetchSensorData().collect { event -> process(event) }
}
```

## Bẫy & ranh giới
Nếu bạn bọc `withContext(Dispatchers.IO)` quanh lệnh `emit` bên trong khối `flow { ... }`, ứng dụng sẽ sập ngay với lỗi `IllegalStateException`. Đừng dùng `Flow` thường thay [`StateFlow`](concept:kotlin-stateflow-vs-sharedflow) khi nhiều nơi cùng nghe kết quả mạng, vì mỗi nơi sẽ gọi lại API. Triệu chứng là crash lúc phát dữ liệu, gọi mạng lặp lại nhiều lần, hoặc giật giao diện do quên gắn `flowOn(Dispatchers.IO)` khi đọc đĩa.

## Câu hỏi thực chiến
**Hỏi:** Tại sao không được dùng withContext để đổi luồng bên trong khối flow builder?
Vì `Flow` tuân thủ bảo toàn ngữ cảnh (`Context Preservation`), nên lệnh `emit` phải chạy đúng luồng nơi gọi `collect`. Nếu `withContext` tự đổi luồng bên trong, khối `collect { ... }` ở tầng giao diện sẽ chạy nhầm trên luồng phụ và gây lỗi. Muốn đổi luồng cho phần việc phía trên, bạn bắt buộc phải viết `.flowOn(...)` ở bên ngoài khối `flow`.

**Hỏi:** Điều gì sẽ xảy ra nếu nhiều nơi cùng gọi collect trên một Cold Flow?
Khối code bên trong `flow { ... }` sẽ chạy lại từ đầu và hoàn toàn độc lập cho từng nơi gọi `collect`. Ví dụ `Flow` mở kết nối đọc file hay gọi API. Khi 2 nơi gọi `collect`, máy sẽ chạy 2 lần riêng.

**Hỏi:** Flow giải quyết bài toán Backpressure như thế nào so với RxJava?
`Flow` tận dụng ngay việc hàm `emit` là một hàm `suspend` thay vì phải dựng bộ điều tiết riêng như RxJava. Khi khối `collect` còn bận xử lý phần tử trước, lệnh `emit` ở bên phát sẽ tự tạm dừng và nhả luồng. Chỉ khi bên nhận xử lý xong phần tử đó thì dòng lệnh sau `emit` mới được đánh thức để tạo phần tử kế tiếp.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được Cold Flow chỉ là khối lệnh đóng gói sẵn, chỉ chạy khi có nơi gọi collect và chạy lại từ đầu cho mỗi người nhận.
- [ ] Trình bày được quy tắc Context Preservation cấm bọc withContext quanh emit và cách flowOn dùng Channel để đổi luồng an toàn.
- [ ] Nêu được cách Flow tự động hãm tốc độ bên phát theo bên nhận (Backpressure) nhờ hàm emit là hàm suspend.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

