---
id: kotlin-coroutines
title: Coroutines
domain: kotlin
tags: [concurrency, asynchronous, kotlin, state-machine]
collections: [interview-fundamental, hay-quen]
related: [android-activity-lifecycle, android-state-holders, kotlin-coroutine-state-machine]
diagrams: [Cơ chế|coroutine-suspend.png|Hàm suspend cất biến vào Continuation, trả COROUTINE_SUSPENDED để thread làm việc khác, rồi chạy tiếp ở nhãn 1 khi resumeWith được gọi.]
created: 2026-09-07
---

## Bản 30 giây
Mở quá nhiều luồng hệ điều hành sẽ làm cạn kiệt RAM và gây sập ứng dụng. Coroutine cắt nhỏ hàm thành các điểm dừng, lưu trạng thái vào Heap và nhả luồng cho việc khác. Nhờ đó, vài luồng duy nhất có thể luân phiên gánh hàng chục nghìn tác vụ cùng lúc.

## Cần biết trước
- thread

## Vấn đề buộc nó ra đời
Năm 1958, Melvin Conway nghĩ ra coroutine khi viết trình biên dịch COBOL trên máy Burroughs 220. Năm 1963, ông chính thức công bố thuật ngữ này trên tạp chí CACM.

Nửa thế kỷ sau, JetBrains đối mặt giới hạn tài nguyên luồng của hệ điều hành. Mỗi thread chiếm trước một megabyte địa chỉ ảo và tiêu tốn tài nguyên quản lý của kernel. Tạo hàng nghìn thread gây cạn kiệt giới hạn hệ thống và làm CPU tê liệt vì chi phí hoán đổi ngữ cảnh.

Callback và RxJava thời đó giải quyết được luồng nhưng làm code vỡ vụn và khó debug. JetBrains cần giải pháp chạy bất đồng bộ tuần tự mà không tạo thêm thread của hệ điều hành.

## Nó thực sự là cái gì
Coroutine KHÔNG phải là thread nhẹ do hệ điều hành quản lý. Thực chất, nó là một [máy trạng thái](concept:kotlin-coroutine-state-machine) do compiler sinh ra tại lúc build kèm một object Continuation trên Heap để lưu điểm dừng.

## Cơ chế
Khi gặp từ khóa `suspend`, compiler chèn tham số ẩn `Continuation` và sinh khối switch-case quản lý trạng thái. Tại điểm dừng, hàm cất các biến sống qua điểm ngắt vào `Continuation` và trả về `COROUTINE_SUSPENDED` để nhả thread. Khi tác vụ xong, `resumeWith()` được gọi để nạp lại dữ liệu và nhảy tới nhãn lệnh tiếp theo. Mã chạy tiếp trên thread do Dispatcher phân bổ.

## Đánh đổi
Bạn có được cú pháp tuần tự và khả năng chạy hàng chục nghìn tác vụ trên số ít thread. Đổi lại, hàm suspend chỉ gọi được từ hàm suspend khác. Vết ngăn xếp (stacktrace) bị xáo trộn do compiler biến đổi luồng điều khiển. Nếu chạy mã chặn luồng không đúng Dispatcher, các luồng trong pool sẽ bị chiếm dụng gây nghẽn.

## Nguyên lý gốc
Biến đổi Continuation-Passing Style: chia luồng thực thi thành các trạng thái rời rạc có thể đóng băng và phục hồi bất kỳ lúc nào.

## Key insight
> Coroutine không phải thread nhẹ nào cả; nó chỉ biết nhường lượt, còn chạy song song hay không là do Dispatcher.

## Ví von
Thay vì bắt một người đứng xếp hàng suốt hai tiếng, bạn lấy số thứ tự rồi rời đi. Khi đến lượt, bất kỳ nhân viên nào rảnh cũng có thể gọi số tiếp tục phục vụ bạn.

## Khuôn mẫu chuẩn
```kotlin
fun loadData() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading
        val result = withContext(Dispatchers.IO) {
            repository.fetchData()
        }
        _uiState.value = UiState.Success(result)
    }
}
```

## Bẫy & ranh giới
Tác vụ nền bị đình trệ hoặc chạy rất chậm là triệu chứng gọi mã chặn luồng trong `Dispatchers.Default`. Bẫy thứ hai là nuốt `CancellationException` trong khối `catch (e: Exception)`, làm vô hiệu hóa cơ chế hủy tự động. Hãy liên kết coroutine với [Activity Lifecycle](concept:android-activity-lifecycle) và [State Holders](concept:android-state-holders). Chống chỉ định khởi tạo coroutine trong vòng lặp lớn vì gây áp lực cấp phát đối tượng Continuation trên Heap cho Garbage Collector.

## Câu hỏi thực chiến
**Hỏi:** Bọc hàm chặn luồng vào `withContext(Dispatchers.Default)` là an toàn không sợ đơ app?
Không phải. `Dispatchers.Default` có số luồng giới hạn đúng bằng số nhân CPU, chỉ dành riêng cho tính toán nặng. Chạy mã chặn luồng ở đây sẽ chiếm dụng các luồng tính toán gây nghẽn; bạn phải dùng `Dispatchers.IO`.

**Hỏi:** `supervisorScope` khác gì `coroutineScope` khi một coroutine con bị lỗi sập?
Khác ở chiều lan truyền lỗi sang các coroutine anh em. Trong `coroutineScope`, một tác vụ con ném lỗi sẽ làm sập cả phạm vi và huỷ hết các việc còn lại. Với `supervisorScope`, các nhánh khác vẫn chạy, nhưng lỗi chưa bắt của một `launch` con vẫn tới exception handler và có thể làm sập app.

**Hỏi:** Tại sao coroutine đang chạy vòng lặp tính toán thì lệnh `job.cancel()` lại không có tác dụng?
Vì việc huỷ coroutine mang tính hợp tác chứ không ép buộc tức thì. Nếu vòng lặp chỉ tính toán CPU thuần tuý mà không gọi hàm suspend, coroutine không có cơ hội kiểm tra lệnh huỷ. Bạn phải chủ động chèn `yield()` hoặc kiểm tra cờ `isActive` trong mỗi vòng lặp để dừng đúng lúc.

**Hỏi:** Nên chọn `launch` hay `async` khi kích hoạt một coroutine mới?
Tuỳ vào việc bạn có cần nhận lại giá trị kết quả hay không. Hãy dùng `launch` cho tác vụ chạy rồi quên như ghi dữ liệu hay bắn sự kiện phân tích. Dùng `async` khi bạn muốn gọi `await()` để gom nhiều kết quả tính toán hoặc gọi mạng song song.

**Hỏi:** Vì sao bắt ngoại lệ bằng `catch (e: Exception)` lại có thể làm hỏng cơ chế huỷ coroutine?
Vì `CancellationException` cũng là một lớp con của `Exception`. Khối `catch` chung sẽ vô tình nuốt mất tín hiệu huỷ, khiến coroutine cứ chạy ngầm mãi. Bạn phải ném lại `CancellationException` hoặc chỉ bắt đúng lớp ngoại lệ cần xử lý như `IOException`.

## Ý bắt buộc khi giải thích lại
- [ ] Giới hạn của OS thread: chiếm trước bộ nhớ ảo và tê liệt CPU do chi phí hoán đổi ngữ cảnh.
- [ ] Bản chất: compiler sinh state machine và lưu biến sống qua điểm ngắt vào object Continuation trên Heap.
- [ ] Bẫy chạy code chặn luồng làm nghẽn Dispatchers.Default và bẫy nuốt CancellationException.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

