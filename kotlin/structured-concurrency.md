---
id: kotlin-structured-concurrency
title: Structured Concurrency
domain: kotlin
tags: [kotlin, coroutines, structured-concurrency, job, supervisor-job]
collections: [interview-fundamental]
related: [kotlin-coroutines, kotlin-coroutine-context-and-dispatchers, thread]
diagrams: [Cơ chế|job-tree.png|Khi async getNews ném lỗi, coroutineScope nhận lỗi và huỷ async getUser; huỷ viewModelScope thì tín hiệu huỷ lan xuống cả cây.]
created: 2026-10-01
---

## Bản 30 giây
Lập trình bất đồng bộ truyền thống dễ làm thất lạc tác vụ ngầm khi một phần việc bị lỗi. Đồng quy có cấu trúc (*structured concurrency*) trói buộc vòng đời các tác vụ con vào tác vụ cha thành một cây gia phả. Khi cha bị hủy hoặc con gặp sự cố, toàn bộ nhánh việc liên quan sẽ tự động dừng lại.

## Cần biết trước
- kotlin-coroutines
- kotlin-coroutine-context-and-dispatchers
- thread

## Vấn đề buộc nó ra đời
Năm 2018, Nathaniel J. Smith công bố bài luận so sánh câu lệnh khởi chạy luồng tự do với lệnh nhảy goto tai hại của thập niên 1960. Các tiến trình chạy ngầm không có điểm neo vòng đời thường bị bỏ quên khi gặp lỗi.

Trong ứng dụng di động, người dùng rời màn hình nhưng các luồng tải mạng vẫn âm thầm thực thi. Hậu quả là ứng dụng rò rỉ bộ nhớ, hao pin và thậm chí gây sập khi cập nhật vào giao diện đã chết.

Tháng 9 năm 2018, Roman Elizarov đưa mô hình này vào thư viện `kotlinx.coroutines 0.26.0`. Bản phát hành này đánh dấu lỗi thời (*deprecated*) các builder toàn cục, đồng thời bổ sung `CoroutineScope` và `GlobalScope`.

## Nó thực sự là cái gì
Đồng quy có cấu trúc KHÔNG PHẢI là một luồng chạy đặc biệt hay tính năng tối ưu tốc độ CPU. Thực chất, đây là kỷ luật quản lý vòng đời dựa trên cấu trúc cây quan hệ giữa các đối tượng `Job`.

Khối lệnh cha chỉ hoàn thành khi toàn bộ các coroutine con bên trong đã kết thúc mỹ mãn.

Ngoại lệ chưa bắt ở một nhánh con sẽ lập tức hủy cha và các nhánh anh em, trừ khi dùng [SupervisorJob](concept:kotlin-coroutine-context-and-dispatchers) hoặc `supervisorScope`.

## Cơ chế
Mỗi khi coroutine con khởi chạy từ một `CoroutineScope`, builder tự động tạo một đối tượng `Job` mới. `Job` mới này gắn vào danh sách con của `Job` cha để tạo nên một cây phân cấp.

Khi cha bị hủy, tín hiệu hủy lan truyền từ trên xuống dưới tới mọi nút lá.

Khi một con ném ngoại lệ, nó gửi tín hiệu hủy ngược lên nút cha để hủy toàn bộ nhánh còn lại.

## Đánh đổi
Mô hình cây đảm bảo tuyệt đối không có tác vụ nào bị rò rỉ chạy ngầm vô thừa nhận. Đổi lại, một lỗi nhỏ ở tác vụ phụ nếu không xử lý kỹ sẽ kéo sập toàn bộ luồng công việc chính.

Hệ thống phải duy trì danh sách liên kết các nút `Job` trên bộ nhớ Heap trong suốt thời gian thực thi.

## Nguyên lý gốc
Nguyên lý đóng gói phạm vi: một thao tác xử lý chỉ kết thúc khi mọi phần việc ủy thác bên dưới đã dừng hẳn. Trách nhiệm dọn dẹp lỗi luôn thuộc về cấp chỉ huy trực tiếp.

## Key insight
> Đồng quy có cấu trúc biến việc quản lý luồng từ lệnh nhảy tự do thành khối đóng gói có cây gia phả vòng đời.

## Ví von
Một người quản lý dự án giao việc cho ba nhân viên cấp dưới. Người quản lý chỉ được về khi nhân viên làm xong, và khi dự án bị hủy thì cả nhóm ngừng làm việc.

## Khuôn mẫu chuẩn
```kotlin
suspend fun fetchDashboard(): Dashboard = coroutineScope {
    val userDeferred = async { api.getUser() }
    val newsDeferred = async { api.getNews() }
    Dashboard(userDeferred.await(), newsDeferred.await())
}
```

## Bẫy & ranh giới
Bẫy dùng `GlobalScope.launch` phá vỡ hoàn toàn cây `Job`, biến tác vụ thành tiến trình mồ côi gây rò rỉ bộ nhớ. Chống chỉ định truyền trực tiếp `Job()` vào builder vì thao tác này cắt đứt liên kết cha con với phạm vi hiện tại. Nếu muốn một nhánh con thất bại không làm hủy nhánh khác, hãy dùng `supervisorScope` hoặc `SupervisorJob`. Triệu chứng vi phạm là rò rỉ [Memory leak](concept:android-memory-leak), hoặc một cuộc gọi mạng phụ bị lỗi kéo theo sập toàn bộ màn hình.

## Câu hỏi thực chiến
**Hỏi:** Vì sao `launch(Job())` bên trong `viewModelScope` vẫn chạy tiếp sau khi màn hình đã đóng?
`Job()` truyền vào làm tham số trở thành cha mới và cắt đứt liên kết với `Job` của scope. Do đó, lệnh hủy từ `viewModelScope` khi màn hình đóng không thể lan truyền tới tác vụ này. Bạn nên tránh truyền `Job()` mới vào builder trừ khi cố ý tạo tiến trình độc lập.

**Hỏi:** Khối `runBlocking` hoặc `coroutineScope` có chờ các coroutine con không gắn `await` kết thúc không?
Có. Theo quy tắc này, phạm vi cha luôn đình chỉ và đợi toàn bộ các con kết thúc trước khi hoàn tất chính nó. Bạn không cần gọi `join` thủ công cho từng nhánh con trừ khi cần đồng bộ thứ tự chạy.

**Hỏi:** Việc truyền `CoroutineScope(Job())` vào ViewModel có gây rò rỉ bộ nhớ không?
Có nguy cơ rất cao. Nếu bạn không gọi `cancel()` khi ViewModel bị hủy trong [ViewModel](concept:android-viewmodel), các coroutine chạy bên trong sẽ tiếp tục sống. Do đó, bạn nên luôn dùng `viewModelScope` tích hợp sẵn để tự động hủy công việc theo vòng đời màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Phân tích cấu trúc cây phân cấp Job cha con và cơ chế tự động chờ mọi con kết thúc.
- [ ] Giải thích nguyên lý lan truyền tín hiệu hủy hai chiều giữa cha và con.
- [ ] Nêu rõ bẫy truyền Job() vào builder làm mất liên kết hủy của viewModelScope.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

