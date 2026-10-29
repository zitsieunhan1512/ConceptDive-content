---
id: android-memory-leak
title: Memory leak
domain: android
tags: [android, memory, garbage-collection, performance]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-viewmodel, android-gc-roots, android-context-application-vs-activity]
created: 2026-09-07
sources:
  - Overview of memory management — Android | https://developer.android.com/topic/performance/memory-overview
  - Capture a heap dump — Android | https://developer.android.com/studio/profile/memory-profiler
  - Garbage collection (computer science) — Wikipedia | https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)
diagrams: [Cơ chế|memory-leak-activity-tree.png|Singleton hoặc luồng ngầm neo giữ Activity đã đóng, kéo theo toàn bộ cây View trên Heap gây OutOfMemoryError]
---

## Bản 30 giây
Khi đóng màn hình mà bộ nhớ không được trả lại, RAM trống sẽ cạn dần tới khi ứng dụng sập. Lỗi xảy ra vì đối tượng sống lâu giữ tham chiếu tới màn hình cũ, khiến bộ thu gom rác không xóa được. Bạn phải cắt mọi tham chiếu trỏ tới màn hình ngay khi nó đóng lại.

## Cần biết trước
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
McCarthy làm bộ thu gom rác (Garbage Collection) cho Lisp năm 1959, công bố năm 1960. Cơ chế này tự động thu hồi vùng nhớ không còn dùng, thay cho việc lập trình viên phải tự xóa bằng tay.

Năm 2008, Android ra mắt trên chiếc HTC Dream chỉ có 192MB RAM. Mỗi ứng dụng chỉ được cấp từ 16MB đến 24MB bộ nhớ Heap. Trong khi đó, mỗi `Activity` lại giữ cả cây `View` và ảnh bitmap nặng hàng megabyte.

Nếu một biến `static` hay luồng chạy ngầm giữ lại `Activity` đã đóng, bộ thu gom rác sẽ không thu hồi được nó. Chỉ cần người dùng xoay ngang màn hình vài lần là ứng dụng tràn Heap và sập vì lỗi `OutOfMemoryError`.

## Nó thực sự là cái gì
Rò rỉ bộ nhớ không phải là RAM bị hỏng hay dữ liệu tự dưng biến mất. Đó là những đối tượng đã dùng xong và bỏ đi. Tuy nhiên, bộ thu gom rác không xóa được vì vẫn còn đường nối từ [GC Roots](concept:android-gc-roots).

## Cơ chế
Bộ thu gom rác bắt đầu tìm kiếm từ các điểm gốc [GC Roots](concept:android-gc-roots) gồm biến `static`, luồng đang chạy và tham chiếu JNI. Bất kỳ đối tượng nào không còn đường nối về `GC Roots` sẽ bị dọn sạch. Nhưng nếu singleton hay luồng ngầm giữ biến trỏ tới `Activity`, cả cây `View` sẽ dính chặt vào `GC Root`. Vì thế, bộ thu gom rác buộc phải chừa toàn bộ màn hình cũ lại trên vùng nhớ [Heap](concept:jvm-stack-vs-heap).

## Đánh đổi
Có bộ thu gom rác tự động giúp bạn không phải tự xóa từng vùng nhớ bằng tay. Đổi lại, khi bộ nhớ gần đầy và phải dọn rác liên tục, máy ảo sẽ chiếm CPU và làm màn hình bị giật (jank). Vì máy ảo không đoán được biến nào bạn đã hết dùng, nó buộc phải giữ lại mọi đối tượng còn đường nối về `GC Root`.

## Nguyên lý gốc
Một đối tượng chỉ được giữ lại trong bộ nhớ khi vẫn còn đường tham chiếu nối từ các điểm gốc `GC Root` tới nó.

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
Gán [Activity](concept:android-context-application-vs-activity) vào biến trong `companion object` hay singleton là lỗi hay gặp nhất làm ứng dụng sập vì `OutOfMemoryError`. Bẫy thứ hai là mở coroutine trong `GlobalScope`, khiến luồng ngầm giữ chặt `Activity` cũ qua mỗi lần người dùng xoay màn hình. Bạn hãy chạy việc ngầm trong `lifecycleScope` theo [Activity Lifecycle](concept:android-activity-lifecycle) và [ViewModel](concept:android-viewmodel) để công việc tự hủy khi màn hình đóng. Đừng bọc `WeakReference` để chữa cháy tạm bợ thay vì cắt đứt tham chiếu đúng theo vòng đời màn hình.

## Câu hỏi thực chiến
**Hỏi:** Rò rỉ bộ nhớ có làm app sập ngay lập tức không?
Không. Rò rỉ một vài đối tượng nhỏ ban đầu hầu như không làm ứng dụng có biểu hiện gì lạ. Lỗi chỉ nổ ra thành `OutOfMemoryError` khi người dùng mở đi mở lại hoặc xoay màn hình nhiều lần làm đầy tràn vùng nhớ Heap.

**Hỏi:** Bọc mọi biến vào WeakReference có giúp chống rò rỉ triệt để không?
Không nên làm vậy. `WeakReference` chỉ là cách chữa cháy và bắt bạn kiểm tra `null` mọi nơi vì đối tượng có thể mất bất ngờ. Cách làm đúng là gắn công việc vào [Activity Lifecycle](concept:android-activity-lifecycle) qua `lifecycleScope` để nó tự hủy gọn gàng khi màn hình đóng.

**Hỏi:** Inner class khác gì static nested class mà hay gây rò rỉ Activity?
Một `inner class` trong Kotlin (hoặc non-static inner class trong Java) luôn ngầm giữ một biến ẩn trỏ về `Activity` bọc ngoài nó. Nếu giao `inner class` cho luồng ngầm chạy lâu, biến ẩn này sẽ giữ chặt cả cây `View` của `Activity` trên Heap.

**Hỏi:** Khi nào nên dùng Application Context thay cho Activity Context, làm sao biết?
Bạn hãy xem đối tượng nhận `Context` đó có sống lâu hơn một màn hình đơn lẻ hay không. Những thứ sống suốt lúc mở ứng dụng như singleton hay kho dữ liệu thì chỉ được nhận `applicationContext` để không giữ chân màn hình. Ngược lại, các việc vẽ giao diện hay bật hộp thoại thì bắt buộc phải truyền `Activity` để có đúng bảng màu `Theme`.

**Hỏi:** Màn hình đóng rồi thì coroutine trong viewModelScope có bị rò rỉ không?
Không. Khi người dùng bấm thoát hẳn màn hình, hệ thống sẽ gọi `onCleared()` và hủy sạch mọi coroutine đang chạy trong `viewModelScope`. Miễn là [ViewModel](concept:android-viewmodel) không giữ biến trỏ thẳng tới `View` hay `Activity`, toàn bộ bộ nhớ màn hình sẽ được dọn sạch.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cơ chế GC Root: bộ thu gom rác chỉ xóa đối tượng khi đứt hết mọi đường dẫn từ gốc.
- [ ] Nêu được hậu quả khi giữ tham chiếu tới Activity đã đóng: kéo theo cả cây View và ảnh bitmap gây OutOfMemoryError.
- [ ] Chỉ ra được bẫy giữ Activity trong biến static hoặc GlobalScope và cách dùng lifecycleScope để tự hủy.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
