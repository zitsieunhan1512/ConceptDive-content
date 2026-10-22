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
Một coroutine cần mang theo thông tin về luồng thực thi, vòng đời và chiến lược xử lý lỗi trên suốt đường chạy. `CoroutineContext` đóng vai trò túi hành lý chứa các phần tử cấu hình, còn `Dispatchers` quyết định luồng hệ thống nào sẽ chạy tác vụ. Khi tạo coroutine con, hệ thống tự động sao chép ngữ cảnh từ cha nhưng luôn cấp mới một công việc điều khiển độc lập.

## Cần biết trước
- kotlin-coroutines
- kotlin-coroutine-state-machine
- thread

## Vấn đề buộc nó ra đời
Trước năm 2017, việc lập trình bất đồng bộ bằng Callback hay Thread Pool truyền thống khiến ngữ cảnh bị phân mảnh nghiêm trọng. Lập trình viên phải tự truyền thủ công các biến cờ trạng thái, mã định danh và luồng xử lý qua từng hàm lồng nhau.

Khi xảy ra lỗi hoặc người dùng hủy tác vụ, các luồng con vẫn âm thầm chạy ngầm. Thiếu liên kết cha con khiến tài nguyên bị rò rỉ nghiêm trọng và gây xung đột giữa các luồng.

Đội ngũ Kotlin tạo ra `CoroutineContext` và cơ chế [đồng quy có cấu trúc](concept:kotlin-structured-concurrency) (*Structured Concurrency*) để xâu chuỗi mọi tác vụ. Toàn bộ thông tin định tuyến luồng và vòng đời giờ đây được quản lý tập trung và kế thừa tự động.

## Nó thực sự là cái gì
`CoroutineContext` KHÔNG phải là một bảng băm (*HashMap*) lưu trữ dữ liệu thông thường. Thực chất, đây là một danh sách liên kết lệch trái bất biến (*CombinedContext*) chứa các phần tử dị thể.

Mỗi phần tử có một khóa duy nhất để truy xuất kiểu an toàn mà không cần ép kiểu thủ công.

## Cơ chế
`Dispatchers` thực chất là một bộ chặn điểm tiếp tục (*ContinuationInterceptor*) nằm sẵn bên trong `CoroutineContext`. Khi hàm tạm ngưng được đánh thức, bộ chặn này bọc điểm tiếp tục thành một đối tượng điều phối (*DispatchedContinuation*).

Nó đẩy đối tượng này vào hàng đợi của thread pool hoặc vòng lặp sự kiện trên luồng giao diện. Toán tử cộng ghép các phần tử lại với nhau, trong đó phần tử bên phải sẽ ghi đè phần tử cùng loại bên trái.

## Đánh đổi
Ngữ cảnh bất biến giúp phép cộng tái sử dụng cấu trúc cũ với chi phí cực thấp mà không cần sao chép bảng. Đổi lại, phép tìm kiếm phải duyệt tuần tự, dù phần tử điều phối luôn được đặt ngoài cùng để thấy ngay.

Nếu lạm dụng toán tử cộng trên vòng lặp lớn, hệ thống sẽ cấp phát nhiều đối tượng bao bọc trên bộ nhớ Heap.

## Nguyên lý gốc
Nguyên lý ngữ cảnh thực thi: mọi tiến trình tính toán đều cần một môi trường chứa siêu dữ liệu để định hướng hành vi. Môi trường này phải có khả năng mở rộng linh hoạt mà không phá vỡ cấu trúc của đối tượng tiếp nhận.

## Key insight
> Dispatchers không phải công cụ quản lý luồng độc lập, mà là một bộ chặn ngữ cảnh điều hướng điểm tiếp tục vào hàng đợi.

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
Dùng chung `Job()` thông thường sẽ khiến một coroutine con bị hủy làm sập toàn bộ các nhánh anh em. Chống chỉ định gán `Dispatchers.Main` cho các tác vụ giải mã ảnh hoặc sắp xếp danh sách lớn vì gây nghẽn luồng vẽ. Khi phối hợp với [Flow](concept:kotlin-flow-under-the-hood), gọi `withContext` đổi luồng trong khối phát sẽ vi phạm bất biến và gây crash. Triệu chứng gồm sập app dây chuyền, đóng băng giao diện (*ANR*) hoặc nghẽn luồng do cạn kiệt thread pool.

## Câu hỏi thực chiến
**Hỏi:** Coroutine con có kế thừa Job của coroutine cha không?
Không, coroutine con kế thừa toàn bộ ngữ cảnh từ cha như Dispatcher hay CoroutineName, nhưng ngoại trừ `Job`. Builder luôn sinh một thể hiện `Job` mới riêng biệt và gắn nó làm nhánh con của Job cha. Cơ chế này đảm bảo lệnh hủy lan truyền từ cha xuống con, trừ khi bạn cố tình truyền `Job()` mới vào builder.

**Hỏi:** Có thể dùng CoroutineExceptionHandler cho hàm launch của coroutine con không?
Không có tác dụng. `CoroutineExceptionHandler` chỉ được xử lý khi gắn vào phạm vi gốc (*Root Coroutine*), vì lỗi từ con luôn được đẩy thẳng lên cha.

**Hỏi:** Toán tử cộng hai CoroutineContext có thể ghi đè phần tử cũ không?
Có. Khi cộng hai ngữ cảnh cùng loại, phần tử nằm ở vế bên phải sẽ thay thế hoàn toàn phần tử ở vế trái.

## Ý bắt buộc khi giải thích lại
- [ ] CoroutineContext là danh sách lệch trái bất biến CombinedContext và kế thừa ngữ cảnh cha con.
- [ ] Dispatchers là ContinuationInterceptor bọc và đẩy DispatchedContinuation vào hàng đợi luồng.
- [ ] Bẫy dùng Job thường thay vì SupervisorJob làm lan truyền lỗi sập dây chuyền.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

