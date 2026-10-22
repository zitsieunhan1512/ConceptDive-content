---
id: kotlin-flow-under-the-hood
title: Flow: Bản chất Cold Stream
domain: kotlin
tags: [kotlin, flow, coroutines, reactive, cold-stream, backpressure]
collections: [interview-fundamental]
related: [kotlin-coroutines, kotlin-stateflow-vs-sharedflow, kotlin-coroutine-context-and-dispatchers]
diagrams: [Cơ chế|flow-flowon-channel.png|flowOn tách luồng phát và luồng nhận thành hai coroutine độc lập nối qua Channel buffer để bảo toàn ngữ cảnh]
created: 2026-09-15
---

## Bản 30 giây
Sinh dữ liệu bất đồng bộ liên tục dễ lãng phí tài nguyên nếu không có ai đón nhận. `Flow` dùng mô hình dữ liệu nguội, chỉ bắt đầu tính toán khi có người gọi lệnh thu thập. Khả năng tạm ngưng của coroutine giúp tự động điều tiết áp lực ngược mà không cần bộ đệm phức tạp.

## Cần biết trước
- kotlin-coroutines
- kotlin-stateflow-vs-sharedflow
- kotlin-coroutine-context-and-dispatchers

## Vấn đề buộc nó ra đời
Trước năm 2019, lập trình viên Android phụ thuộc vào RxJava để xử lý các luồng dữ liệu phản ứng (*Reactive Streams*). RxJava rất mạnh nhưng bộ API đồ sộ khiến dự án phình to dung lượng và khó gỡ lỗi.

Nơi phát nhanh hơn nơi nhận tạo ra áp lực ngược (*Backpressure*) đè nặng lên bộ nhớ. Lập trình viên phải chọn chiến lược đệm hay thả rơi rất phức tạp, dễ gây lỗi tràn bộ nhớ.

Đội ngũ Kotlin phát triển `Flow` như một giải pháp thay thế tinh gọn dựa trên Coroutine. Cơ chế tạm ngưng nguyên bản giúp dữ liệu truyền nhịp nhàng mà không cần hệ thống điều tiết cồng kềnh.

## Nó thực sự là cái gì
`Flow` KHÔNG phải là một tiến trình luồng chạy ngầm hay một danh sách lưu sẵn phần tử trên bộ nhớ. Thực chất, đây là một khối lệnh tạm ngưng đóng gói bên trong một giao diện hàm thu thập duy nhất.

Mỗi khi gọi lệnh thu thập, toàn bộ khối mã phát dữ liệu mới thực sự chạy lại từ đầu.

## Cơ chế
Khi gọi `emit`, hàm phát dữ liệu sẽ tạm ngưng cho tới khi hàm xử lý của bên nhận hoàn tất công việc. Để bảo toàn ngữ cảnh (*Context Preservation*), `Flow` cấm tuyệt đối việc tự ý chuyển đổi luồng bằng `withContext` bên trong khối phát.

Toán tử `flowOn` thay đổi luồng phía trên bằng cách tự động tạo một kênh đệm (*Channel*) kết nối giữa hai coroutine độc lập. Nhờ đó, luồng phát và luồng nhận có thể vận hành trên hai [điều phối viên](concept:kotlin-coroutine-context-and-dispatchers) khác nhau một cách an toàn.

## Đánh đổi
Mô hình dữ liệu nguội tiết kiệm pin và RAM vì không tính toán khi vắng bóng người quan sát. Đổi lại, nếu có nhiều người cùng thu thập, khối lệnh phát dữ liệu sẽ bị thực thi lại nhiều lần.

Chi phí chuyển đổi ngữ cảnh qua kênh đệm của `flowOn` cũng tạo ra áp lực cấp phát rác nhỏ trên Heap.

## Nguyên lý gốc
Nguyên lý dòng dữ liệu phản ứng: dữ liệu chỉ nên được tạo ra theo nhu cầu thực tế của nơi tiêu thụ. Sự đồng bộ nhịp độ giữa bên sản xuất và bên tiêu thụ là chìa khóa để ngăn ngừa cạn kiệt bộ đệm.

## Key insight
> Flow không phải công cụ chạy ngầm liên tục, mà là một khối mã đóng băng chỉ thức giấc khi có lệnh thu thập.

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
Gọi hàm `withContext` trực tiếp bên trong khối `flow { ... }` sẽ bị chặn đứng ngay lập tức bằng ngoại lệ bất hợp pháp. Chống chỉ định dùng Cold Flow thay cho [StateFlow](concept:kotlin-stateflow-vs-sharedflow) khi phát sự kiện cho nhiều nơi vì gây gọi mạng lặp lại. Triệu chứng lỗi gồm crash app với `IllegalStateException`, tiêu tốn băng thông do gọi API thừa thãi hoặc tụt khung hình.

## Câu hỏi thực chiến
**Hỏi:** Tại sao không được dùng withContext để đổi luồng bên trong khối flow builder?
Không được phép vì vi phạm quy tắc bảo toàn ngữ cảnh. Trình chạy Flow bắt buộc luồng phát phải giữ nguyên ngữ cảnh của bên thu thập, và bạn phải dùng `flowOn`.

**Hỏi:** Điều gì sẽ xảy ra nếu nhiều nơi cùng gọi collect trên một Cold Flow?
Khối lệnh bên trong flow sẽ bị thực thi lại từ đầu cho mỗi người thu thập. Mỗi người nhận sẽ có một phiên làm việc riêng biệt và không dùng chung luồng dữ liệu.

**Hỏi:** Flow giải quyết bài toán Backpressure như thế nào so với RxJava?
Nó dựa vào cơ chế suspend tự nhiên thay vì bộ đệm phức tạp. Khi bên nhận chưa xử lý xong, lệnh `emit` của bên phát sẽ tạm ngưng và đợi bên nhận sẵn sàng.

## Ý bắt buộc khi giải thích lại
- [ ] Cold Flow chỉ chạy khi có người gọi collect và chạy lại từ đầu cho mỗi người nhận.
- [ ] Nguyên tắc Context Preservation cấm withContext trong flow và bắt buộc dùng flowOn.
- [ ] Cơ chế giải quyết Backpressure tự nhiên bằng hàm tạm ngưng suspend.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

