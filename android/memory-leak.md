---
id: android-memory-leak
title: Memory leak
domain: android
tags: [android, memory, garbage-collection, performance]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-viewmodel, android-gc-roots, android-context-application-vs-activity]
created: 2026-09-07
diagrams: [Cơ chế|memory-leak-activity-tree.png|Singleton hoặc luồng ngầm neo giữ Activity đã đóng, kéo theo toàn bộ cây View trên Heap gây OutOfMemoryError]
---

## Bản 30 giây
Khi đóng màn hình mà bộ nhớ không được giải phóng, RAM trống giảm dần đến khi ứng dụng sập. Sự cố xảy ra do một vật thể sống lâu vẫn giữ tham chiếu, khiến trình dọn rác không thể thu hồi màn hình. Bạn phải cắt đứt mọi liên kết trỏ tới màn hình ngay khi nó kết thúc.

## Cần biết trước
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Năm 1959, John McCarthy tạo ra Garbage Collection (thu gom rác) cho Lisp. Cơ chế này tự động thu hồi vùng nhớ không còn dùng, thay thế việc giải phóng thủ công.

Năm 2008, Android ra mắt trên HTC Dream với vỏn vẹn 192MB RAM. Mỗi ứng dụng chỉ được cấp từ 16MB đến 24MB Heap. Mỗi Activity lại giữ toàn bộ cây View và ảnh bitmap nặng hàng megabyte.

Khi biến tĩnh hoặc luồng ngầm giữ Activity sau khi đóng, bộ dọn rác bất lực không thể thu hồi. Người dùng xoay máy vài lần là app tràn Heap và sập với lỗi `OutOfMemoryError`.

## Nó thực sự là cái gì
Rò rỉ bộ nhớ KHÔNG phải là dữ liệu biến mất hay hỏng phần cứng. Thực chất, đó là đối tượng bỏ đi nhưng trình dọn rác không thể thu hồi vì còn đường dẫn từ [gốc thu gom](concept:android-gc-roots).

## Cơ chế
Bộ dọn rác duyệt từ các [gốc thu gom (GC Root)](concept:android-gc-roots) gồm biến static, thread đang chạy và tham chiếu JNI. Bất kỳ vùng nhớ nào không nối tới GC Root sẽ bị dọn sạch. Nếu một singleton hay thread ngầm giữ tham chiếu Activity, toàn bộ cây View sẽ bị neo chặt vào GC Root. Do đó, Garbage Collector bắt buộc phải giữ chúng lại trên [Heap](concept:jvm-stack-vs-heap).

## Đánh đổi
Thu gom rác tự động giúp lập trình viên không phải quản lý con trỏ thủ công. Đổi lại, dọn rác dồn dập tranh chấp CPU và làm nghẽn cấp phát gây giật màn hình (jank). Máy ảo không thể đoán ý định người viết nên buộc phải giữ mọi vật thể còn đường nối tới GC Root.

## Nguyên lý gốc
Nguyên lý khả năng vươn tới: dữ liệu chỉ được giữ lại khi còn đường dẫn tham chiếu từ các điểm gốc GC Root.

## Key insight
> Trình dọn rác không dọn thứ bạn không dùng, nó chỉ dọn thứ không thể chạm tới từ bất kỳ GC Root nào.

## Khuôn mẫu chuẩn
```kotlin
class ProfileActivity : ComponentActivity() {
    fun loadData() {
        lifecycleScope.launch {
            val result = repository.fetchData()
            updateUi(result)
        }
    }
}
```

## Bẫy & ranh giới
Gán [Activity](concept:android-context-application-vs-activity) vào biến `companion object` hay singleton là nguyên nhân số một làm sập app với `OutOfMemoryError`. Bẫy thứ hai là chạy coroutine trong `GlobalScope`, khiến luồng ngầm giữ chặt Activity qua nhiều đợt xoay màn hình. Hãy dùng `lifecycleScope` theo [Activity Lifecycle](concept:android-activity-lifecycle) và [ViewModel](concept:android-viewmodel) để tác vụ tự hủy theo vòng đời. Chống chỉ định dùng `WeakReference` để chắp vá thay vì triệt tiêu liên kết qua các thành phần nhận biết vòng đời.

## Câu hỏi thực chiến
**Hỏi:** Rò rỉ bộ nhớ có làm app sập ngay lập tức không?
Không. Rò rỉ một đối tượng nhỏ thường không phát sinh triệu chứng nhìn thấy được ở thời điểm ban đầu. Sự cố chỉ bùng nổ thành `OutOfMemoryError` khi rò rỉ tích tụ dần qua nhiều lần xoay màn hình làm cạn sạch Heap.

**Hỏi:** Bọc mọi biến vào WeakReference có giúp chống rò rỉ triệt để không?
Không phải. `WeakReference` chỉ là miếng dán chắp vá và bắt bạn liên tục xử lý trường hợp giá trị bị thu hồi thành `null`. Cách chuẩn mực là gắn tác vụ vào [Activity Lifecycle](concept:android-activity-lifecycle) bằng `lifecycleScope` để nó tự hủy khi màn hình đóng.

**Hỏi:** Inner class khác gì static nested class mà hay gây rò rỉ Activity?
Inner class ngầm giữ một tham chiếu ẩn trỏ về đối tượng bao ngoài là Activity, còn static nested class thì không. Khi đưa inner class vào tác vụ nền chạy dài, tham chiếu ẩn này sẽ neo toàn bộ cây View lại trên Heap.

**Hỏi:** Khi nào nên dùng Application Context thay cho Activity Context, làm sao biết?
Tuỳ vào việc đối tượng lưu giữ có sống lâu hơn một màn hình đơn lẻ hay không. Các đối tượng sống lâu như Singleton hay kho dữ liệu chỉ được giữ Application Context để tránh rò rỉ. Ngược lại, nạp giao diện và hiển thị hộp thoại bắt buộc phải dùng Activity Context để lấy đúng Theme hiển thị.

**Hỏi:** Màn hình đóng rồi thì coroutine trong viewModelScope có bị rò rỉ không?
Không. Khi người dùng thoát hẳn màn hình, hệ thống sẽ kích hoạt hàm `onCleared()` để hủy toàn bộ công việc trong `viewModelScope`. Miễn là [ViewModel](concept:android-viewmodel) không giữ tham chiếu tới View hay Activity, bộ nhớ sẽ được giải phóng hoàn toàn sạch sẽ.

## Ý bắt buộc khi giải thích lại
- [ ] Cơ chế GC Root: chỉ thu hồi đối tượng khi đứt toàn bộ đường dẫn từ gốc.
- [ ] Tác hại neo giữ Activity: kéo theo toàn bộ cây View và bitmap gây OutOfMemoryError.
- [ ] Bẫy biến static, bẫy luồng chạy ngầm GlobalScope và giải pháp tự hủy bằng lifecycleScope.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
