---
id: jvm-stack-vs-heap
title: Stack vs Heap
domain: jvm
tags: [jvm, memory, art, performance]
collections: [interview-fundamental]
related: [kotlin-coroutine-state-machine, android-gc-roots, thread, android-memory-leak]
created: 2026-09-21
diagrams: [Cơ chế|stack-vs-heap-memory.png|Khung ngăn xếp luồng neo giữ đối tượng Heap qua GC Root, trường nguyên thuỷ nằm trọn vẹn trên Heap]
---

## Bản 30 giây
Khi bạn gọi một hàm, biến tạm được đặt lên ngăn xếp Stack của riêng luồng đó và tự mất ngay khi hàm xong. Còn các đối tượng bạn tạo ra thì nằm trên vùng nhớ chung Heap và chờ bộ dọn rác thu hồi. Kể cả biến kiểu nguyên thủy như `Int`, nếu là thuộc tính của một đối tượng thì vẫn nằm trọn trên Heap.

## Cần biết trước
- thread
- android-gc-roots

## Vấn đề buộc nó ra đời
Năm 1960, Edsger Dijkstra đề xuất ngăn xếp Call Stack cho ngôn ngữ Algol 60. Ngăn xếp xử lý gọn việc gọi hàm đệ quy và quản lý biến cục bộ theo thứ tự vào sau ra trước.

Trong khi đó, dữ liệu của ngôn ngữ Lisp nằm trên vùng nhớ Heap và được bộ thu gom rác của John McCarthy tự dọn. Nhờ nằm trên Heap, dữ liệu sống tự do mà không bị xóa mất ngay khi hàm tạo ra nó kết thúc.

Nếu chỉ có Stack, bạn không thể chia sẻ dữ liệu giữa các luồng hay trả một đối tượng ra khỏi hàm. Ngược lại, nếu đặt mọi biến tạm lên Heap, máy sẽ tốn rất nhiều công cấp phát và dọn rác.

## Nó thực sự là cái gì
Nhiều người lầm tưởng kiểu nguyên thủy như `Int` luôn nằm trên Stack còn đối tượng nằm trên Heap. Thật ra, vị trí lưu phụ thuộc vào nơi bạn khai báo biến đó. Stack chỉ lưu các khung hàm (`stack frame`) cùng biến cục bộ, còn Heap lưu mọi đối tượng. Vì vậy, một thuộc tính `Int` nằm bên trong lớp `User` vẫn nằm trọn trên Heap cùng đối tượng `User` đó.

## Cơ chế
Mỗi [luồng](concept:thread) Java có một Stack riêng (mặc định 1040KB trên ART, luồng chính là 8MB) chia thành các khung hàm. Khi gọi hàm, CPU dịch con trỏ ngăn xếp lấy chỗ cho biến cục bộ và trả ngay lúc xong. Đối tượng mới được cấp nhanh trên Heap qua vùng đệm TLAB, còn đối tượng quá lớn thì vào thẳng Large Object Space. Các biến trên khung Stack đang chạy chính là những [GC Root](concept:android-gc-roots) giữ cho đối tượng trên Heap không bị dọn mất.

## Đánh đổi
Cấp phát trên Stack cực nhanh vì CPU chỉ cần dịch con trỏ thanh ghi, nhưng dung lượng nhỏ và chỉ dùng trong một luồng. Đổi lại, Heap chứa được dữ liệu lớn và cho nhiều luồng cùng đọc, nhưng tốn công bộ dọn rác quét dọn. Trong [Coroutines](concept:kotlin-coroutine-state-machine), biến cục bộ cần giữ qua điểm dừng phải chuyển từ Stack lên Heap, làm tốn thêm đối tượng.

## Nguyên lý gốc
Biến chỉ dùng trong một lần gọi hàm thì đặt ở ngăn xếp. Dữ liệu cần sống lâu hơn hàm thì cất vào kho chung.

## Key insight
> Vị trí lưu do vòng đời quyết định chứ không do kiểu: trường nguyên thủy của đối tượng vẫn nằm trọn vẹn trên Heap.

## Ví von
Stack như xấp giấy nháp trên bàn làm việc của riêng bạn, tính xong bước nào bạn vò tờ giấy đó đi ngay. Heap như nhà kho chung của công ty, ai cũng vào lấy đồ được. Nhưng kho phải có nhân viên định kỳ đi dọn đồ bỏ không.

## Khuôn mẫu chuẩn
```kotlin
class User(val id: Int) // id là nguyên thủy nhưng nằm trên Heap cùng User

fun allocateMemory() {
    val localPrimitive = 42 // Nằm trên Stack frame của hàm
    val userRef = User(1) // userRef trên Stack, thể hiện User trên Heap
    println("$localPrimitive: ${userRef.id}")
} // Thoát hàm: Stack frame bị hủy; User trên Heap chờ dọn rác
```

## Bẫy & ranh giới
Gọi đệ quy không có điểm dừng sẽ làm đầy các khung Stack và sập ứng dụng với lỗi `StackOverflowError`. Ngược lại, tạo quá nhiều đối tượng hoặc giữ tham chiếu gây [rò rỉ](concept:android-memory-leak) sẽ làm cạn Heap và ném `OutOfMemoryError`. Đừng khai báo biến tạm của hàm thành biến của lớp, vì bộ dọn rác trên Heap phải gánh thêm việc. Lưu ý từ Android 8+, mảng điểm ảnh của `Bitmap` đã chuyển sang Native Heap nên không còn chiếm chỗ trực tiếp trên Java Heap.

## Câu hỏi thực chiến
**Hỏi:** Kích thước Stack của một luồng trong Android là bao nhiêu?
Trên máy ảo ART, một luồng Java mặc định được cấp 1040KB, gồm 1MB vùng nhớ dùng thật và 16KB trang bảo vệ. Riêng luồng chính (Main thread) do nhân Linux tạo từ đầu nên có ngăn xếp mặc định lên tới 8MB.

**Hỏi:** Thuộc tính val age: Int bên trong một class nằm ở Stack hay Heap?
Nằm trên Heap. Các thuộc tính của lớp được gom chung vào vùng nhớ của chính đối tượng đó. Vì đối tượng được tạo trên Heap, mọi thuộc tính nguyên thủy bên trong nó cũng nằm ngay trên Heap.

**Hỏi:** Tại sao Coroutine lại phải đưa biến cục bộ từ Stack lên Heap?
Vì khi coroutine tạm dừng, nó phải nhả luồng hiện tại cho việc khác nhảy vào chạy. Lúc nhả luồng thì khung Stack của hàm bị xóa mất, nên các biến cần dùng tiếp phải cất vào đối tượng `Continuation` trên Heap.

**Hỏi:** Pixel của đối tượng Bitmap trên Android nằm ở Stack, Java Heap hay Native Heap?
Từ Android 8.0 trở đi, mảng điểm ảnh của `Bitmap` được cất hoàn toàn bên vùng nhớ Native Heap của hệ điều hành. Bên Java Heap chỉ giữ một đối tượng vỏ `Bitmap` rất nhỏ chứa con trỏ chỉ sang vùng nhớ native đó.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được Stack lưu khung hàm theo từng luồng, còn Heap lưu đối tượng và cả các thuộc tính nguyên thủy của nó.
- [ ] Nêu được kích thước Stack trên ART là 1040KB cho luồng Java, 8MB cho Main thread, và việc Bitmap nằm ở Native Heap từ Android 8.
- [ ] Trình bày được lý do Coroutine phải cất biến cục bộ từ Stack lên đối tượng Continuation trên Heap khi qua điểm dừng.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
