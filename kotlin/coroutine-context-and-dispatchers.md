---
id: kotlin-coroutine-context-and-dispatchers
title: CoroutineContext & Dispatchers
domain: kotlin
tags: [kotlin, coroutines, coroutine-context, dispatchers, continuation-interceptor]
collections: [interview-fundamental]
related: [kotlin-coroutines, kotlin-coroutine-state-machine, thread, kotlin-flow-under-the-hood]
diagrams: [Cơ chế|coroutine-context-dispatcher-intercept.png|ContinuationInterceptor bọc Continuation thành DispatchedContinuation rồi đẩy vào hàng đợi thread pool]
created: 2026-09-15
---

## Bản 30 giây
Khi một coroutine chạy, nó cần biết mình chạy trên luồng nào, gắn với `Job` nào và xử lý lỗi ra sao. `CoroutineContext` là chiếc túi giữ các thông tin cài đặt đó, còn `Dispatchers` quyết định luồng nào sẽ nhận việc. Khi bạn mở một coroutine con, nó tự chép lại túi cài đặt từ cha nhưng luôn tạo một `Job` con riêng.

## Cần biết trước
- kotlin-coroutines
- kotlin-coroutine-state-machine
- thread

## Vấn đề buộc nó ra đời
Trước năm 2017, viết code bất đồng bộ bằng Callback hay `ThreadPool` làm thông tin tác vụ bị rời rạc khắp nơi. Bạn phải tự truyền biến cờ trạng thái, tên tác vụ và đối tượng quản lý luồng qua từng hàm lồng nhau.

Khi có lỗi hoặc khi người dùng thoát màn hình, các luồng con bên dưới vẫn cứ âm thầm chạy tiếp. Do thiếu liên kết giữa tác vụ cha và con, tài nguyên rất dễ rò rỉ và các luồng dễ đụng độ nhau.

Đội ngũ Kotlin tạo ra `CoroutineContext` cùng cơ chế [đồng quy có cấu trúc](concept:kotlin-structured-concurrency) (`Structured Concurrency`) để gom mọi tác vụ về một mối. Mọi cài đặt về luồng chạy và vòng đời đều được giữ chung trong một túi và tự động truyền từ cha xuống con.

## Nó thực sự là cái gì
`CoroutineContext` không phải là một bảng `HashMap` thông thường, dù bạn vẫn lấy phần tử ra bằng dấu ngoặc vuông như `context[Job]`. Thật ra, nó là danh sách liên kết lệch trái và bất biến (`CombinedContext`), chứa nhiều phần tử cài đặt khác nhau.

Mỗi phần tử có một khóa (`Key`) riêng mang sẵn kiểu dữ liệu, nên bạn lấy ra dùng ngay mà khỏi ép kiểu.

## Cơ chế
Mỗi `Dispatcher` thực chất là một bộ chặn `ContinuationInterceptor` nằm sẵn bên trong túi `CoroutineContext`. Khi một hàm `suspend` vừa chờ mạng xong và chuẩn bị chạy tiếp, bộ chặn này đón lấy `Continuation` rồi bọc thành `DispatchedContinuation`.

Sau đó nó đẩy gói việc này vào hàng đợi của nhóm luồng hoặc gửi vào vòng lặp sự kiện của luồng giao diện. Khi bạn ghép hai túi cài đặt bằng dấu `+`, phần tử ở bên phải sẽ đè lên phần tử cùng loại ở bên trái.

## Đánh đổi
Do `CoroutineContext` bất biến, dùng dấu `+` thêm phần tử sẽ nối thẳng vào cấu trúc cũ mà khỏi chép lại. Đổi lại, tìm phần tử phải dò từng mắt xích, riêng `ContinuationInterceptor` được đặt ngoài cùng để lấy ngay.

Nếu dùng dấu `+` nối ngữ cảnh liên tục trong vòng lặp lớn, chương trình sẽ tạo nhiều `CombinedContext` thừa trên Heap.

## Nguyên lý gốc
Mọi tác vụ đang chạy đều mang theo túi thông tin môi trường bên cạnh. Bạn có thể gắn thêm phần tử mới mà không phải sửa hàm nhận.

## Key insight
> `Dispatchers` không tự giữ luồng riêng, mà là bộ chặn `ContinuationInterceptor` đón `Continuation` rồi đẩy vào hàng đợi của luồng đích.

## Khuôn mẫu chuẩn
```kotlin
val customScope = CoroutineScope(
    Dispatchers.Default + CoroutineName("ComputeWorker") + SupervisorJob()
)
customScope.launch(Dispatchers.IO) {
    val currentThread = Thread.currentThread().name
    val taskName = coroutineContext[CoroutineName]?.name
}
```

## Bẫy & ranh giới
Nếu dùng `Job()` thường cho cả `CoroutineScope`, chỉ cần một coroutine con gọi API bị lỗi là nó hủy luôn các coroutine anh em khác. Cũng đừng chạy việc nặng như giải mã ảnh hay sắp xếp danh sách lớn trên `Dispatchers.Main`, vì sẽ làm đứng màn hình (`ANR`). Khi đang viết trong khối `flow { ... }` của [Flow](concept:kotlin-flow-under-the-hood), nếu bạn bọc `withContext` để đổi luồng phát thì ứng dụng sẽ sập ngay. Triệu chứng thường gặp là cả màn hình ngừng chạy sau một lỗi nhỏ, đơ giao diện, hoặc kẹt cứng do cạn luồng.

## Câu hỏi thực chiến
**Hỏi:** Coroutine con có kế thừa Job của coroutine cha không?
Không, coroutine con chép mọi phần tử từ cha như `Dispatcher` hay `CoroutineName`, trừ `Job`. Hàm tạo luôn sinh một `Job` mới cho con rồi móc làm nhánh dưới của `Job` cha. Nhờ vậy khi hủy cha thì con mới tự hủy theo, trừ khi bạn tự tay truyền một `Job()` lẻ vào hàm tạo.

**Hỏi:** Có thể dùng CoroutineExceptionHandler cho hàm launch của coroutine con không?
Không có tác dụng. `CoroutineExceptionHandler` chỉ bắt được lỗi khi gắn vào coroutine gốc (`Root Coroutine`), vì coroutine con gặp lỗi sẽ đẩy thẳng lên cha.

**Hỏi:** Toán tử cộng hai CoroutineContext có thể ghi đè phần tử cũ không?
Có. Khi dùng dấu `+` ghép hai phần tử cùng khóa, phần tử bên phải sẽ thay thế phần tử bên trái.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được CoroutineContext là danh sách liên kết bất biến CombinedContext và cách con chép ngữ cảnh từ cha nhưng tạo Job mới.
- [ ] Trình bày được Dispatchers thực chất là ContinuationInterceptor bọc Continuation thành DispatchedContinuation rồi đẩy vào hàng đợi luồng.
- [ ] Chỉ ra được bẫy dùng Job thường thay vì SupervisorJob làm một nhánh lỗi kéo sập cả các nhánh anh em.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

