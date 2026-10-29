---
id: kotlin-coroutines
title: Coroutines
domain: kotlin
tags: [concurrency, asynchronous, kotlin, state-machine]
collections: [interview-fundamental, hay-quen]
related: [android-activity-lifecycle, android-state-holders, kotlin-coroutine-state-machine]
sources:
  - Coroutines overview — Kotlin Documentation | https://kotlinlang.org/docs/coroutines-overview.html
  - Coroutines basics — Kotlin Documentation | https://kotlinlang.org/docs/coroutines-basics.html
  - Kotlin coroutines on Android | https://developer.android.com/kotlin/coroutines
  - Coroutine — Wikipedia | https://en.wikipedia.org/wiki/Coroutine
diagrams: [Cơ chế|coroutine-suspend.png|Hàm suspend cất biến vào Continuation, trả COROUTINE_SUSPENDED để thread làm việc khác, rồi chạy tiếp ở nhãn 1 khi resumeWith được gọi.]
created: 2026-09-07
---

## Bản 30 giây
Nếu mỗi lần gọi API hay đọc dữ liệu bạn lại tạo một luồng mới, bộ nhớ RAM sẽ nhanh chóng cạn sạch. Coroutine cắt hàm của bạn tại các điểm chờ, cất biến dở dang vào bộ nhớ Heap rồi nhường luồng cho việc khác chạy. Nhờ vậy, chỉ vài luồng sẵn có cũng đủ thay phiên nhau chạy hàng chục nghìn công việc cùng lúc.

## Cần biết trước
- thread

## Vấn đề buộc nó ra đời
Năm 1958, Melvin Conway đặt ra khái niệm coroutine khi viết một chương trình hợp ngữ (assembler). Năm 1963, ông công bố bài báo trên tạp chí CACM dùng coroutine để thiết kế trình biên dịch COBOL.

Nửa thế kỷ sau, lập trình viên vướng phải giới hạn của luồng hệ điều hành. Mỗi `thread` giữ trước khoảng một megabyte bộ nhớ ảo và tốn tài nguyên quản lý của nhân hệ điều hành. Tạo hàng nghìn `thread` sẽ làm hết bộ nhớ và làm CPU mất nhiều thời gian đổi qua lại giữa các luồng.

Dùng callback hay RxJava tránh được việc tạo nhiều luồng nhưng lại làm code đứt đoạn và rất khó đọc khi lỗi. JetBrains cần một cách viết code chạy từ trên xuống tự nhiên mà không phải xin thêm luồng của hệ điều hành.

## Nó thực sự là cái gì
Coroutine không phải là một luồng nhẹ do hệ điều hành tạo ra hay quản lý. Nó là [máy trạng thái](concept:kotlin-coroutine-state-machine) do trình biên dịch tạo lúc build, kèm đối tượng `Continuation` trên Heap để nhớ chỗ vừa dừng.

## Cơ chế
Khi thấy từ khóa `suspend`, trình biên dịch thêm tham số `Continuation` vào cuối hàm và chia thân hàm thành các nhánh đánh số. Tới điểm chờ, hàm cất các biến còn cần dùng sau đó vào đối tượng `Continuation` rồi trả về `COROUTINE_SUSPENDED` để nhả luồng. Khi việc chờ đã xong, hàm `resumeWith()` được gọi để lấy lại biến và nhảy tới đúng nhánh kế tiếp. Đoạn code sau đó chạy trên luồng nào là do `Dispatcher` chỉ định.

## Đánh đổi
Bạn viết được code bất đồng bộ thẳng một mạch từ trên xuống và chạy hàng chục nghìn việc trên vài luồng ít ỏi. Đổi lại, hàm `suspend` chỉ gọi được từ một hàm `suspend` khác hoặc bên trong một coroutine. Khi có lỗi, stack trace nhìn ngắn hơn vì trình biên dịch đã cắt hàm thành các chặng rời nhau. Nếu bạn đặt lệnh đọc file chặn luồng vào nhầm `Dispatcher`, cả nhóm luồng ở đó sẽ bị kẹt theo.

## Nguyên lý gốc
Chia một hàm dài thành nhiều chặng nhỏ theo kiểu Continuation-Passing Style để dừng ở bất kỳ chặng nào và gọi chạy tiếp về sau.

## Key insight
> Coroutine không phải thread nhẹ nào cả; nó chỉ biết nhường lượt, còn chạy song song hay không là do Dispatcher.

## Ví von
Thay vì bắt bạn đứng chôn chân trước quầy suốt hai tiếng chờ làm thủ tục, quầy đưa bạn tấm phiếu ghi số thứ tự. Bạn ra ghế ngồi làm việc khác, khi tới số thì bất kỳ nhân viên nào đang rảnh cũng gọi bạn lên làm tiếp được.

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
Đừng tạo coroutine mới trong vòng lặp triệu lần, vì mỗi lần cấp thêm đối tượng trên Heap và tốn công dọn rác. Cũng đừng gọi lệnh đọc đĩa hay chờ mạng chặn luồng trong `Dispatchers.Default`, vì các tác vụ nền khác sẽ đứng im chờ theo. Khi bắt lỗi bằng `catch (e: Exception)`, nếu bạn nuốt luôn `CancellationException` thì coroutine sẽ không dừng lại được khi màn hình đã đóng. Hãy gắn coroutine với [Activity Lifecycle](concept:android-activity-lifecycle) và [State Holders](concept:android-state-holders) để tự hủy đúng lúc.

## Câu hỏi thực chiến
**Hỏi:** Bọc hàm chặn luồng vào `withContext(Dispatchers.Default)` là an toàn không sợ đơ app?
Không đúng. `Dispatchers.Default` chỉ mở số luồng bằng số nhân CPU của máy, dành riêng cho việc tính toán nặng. Đưa lệnh đọc file hay chờ mạng chặn luồng vào đây sẽ giam hết số luồng đó; bạn phải dùng `Dispatchers.IO`.

**Hỏi:** `supervisorScope` khác gì `coroutineScope` khi một coroutine con bị lỗi sập?
Khác ở chỗ lỗi có làm hủy luôn các coroutine anh em đang chạy cùng hay không. Trong `coroutineScope`, chỉ cần một việc con ném lỗi là cả phạm vi dừng lại và hủy hết các việc con còn lại. Trong `supervisorScope`, các nhánh anh em vẫn chạy tiếp, nhưng lỗi chưa bắt của khối `launch` con vẫn báo lên handler và làm sập app.

**Hỏi:** Tại sao coroutine đang chạy vòng lặp tính toán thì lệnh `job.cancel()` lại không có tác dụng?
Vì lệnh hủy chỉ bật một cờ báo dừng, còn coroutine phải tự kiểm tra cờ đó mới dừng lại. Vòng lặp chỉ tính toán thuần mà không gọi hàm `suspend` nào thì không có điểm dừng để kiểm tra cờ hủy. Bạn cần gọi `yield()` hoặc kiểm tra biến `isActive` trong mỗi vòng lặp để thoát ra kịp thời.

**Hỏi:** Nên chọn `launch` hay `async` khi kích hoạt một coroutine mới?
Tùy xem bạn có cần lấy kết quả trả về từ công việc đó hay không. Dùng `launch` cho những việc làm xong là thôi như lưu lịch sử hay gửi sự kiện thống kê. Dùng `async` khi bạn cần gọi `await()` để lấy kết quả trả về, nhất là lúc gọi 2 API song song rồi gộp dữ liệu.

**Hỏi:** Vì sao bắt ngoại lệ bằng `catch (e: Exception)` lại có thể làm hỏng cơ chế huỷ coroutine?
Vì coroutine dùng chính ngoại lệ `CancellationException` để báo tin hủy và thoát khỏi hàm đang chạy. Do `CancellationException` là lớp con của `Exception`, khối `catch` chung sẽ bắt luôn cả tín hiệu hủy này làm coroutine cứ chạy tiếp. Bạn chỉ nên bắt đúng lỗi cần xử lý như `IOException`, hoặc kiểm tra nếu là `CancellationException` thì ném trả lại.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao tạo hàng nghìn OS thread làm tốn bộ nhớ ảo và mất thời gian đổi ngữ cảnh CPU.
- [ ] Trình bày được cách trình biên dịch biến hàm suspend thành máy trạng thái và cất biến vào đối tượng Continuation trên Heap.
- [ ] Chỉ ra được bẫy gọi lệnh chặn luồng trong Dispatchers.Default và bẫy nuốt mất CancellationException trong khối catch.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
