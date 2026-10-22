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
Bộ nhớ chương trình chia thành hai vùng: ngăn xếp riêng cho từng luồng và vùng nhớ chung cho toàn ứng dụng. Ngăn xếp lưu các biến tạm thời khi gọi hàm và tự dọn dẹp ngay khi hàm kết thúc. Vùng nhớ chung lưu trữ các đối tượng dữ liệu lớn và cần bộ dọn rác quét dọn định kỳ.

## Cần biết trước
- thread
- android-gc-roots

## Vấn đề buộc nó ra đời
Năm 1960, Edsger Dijkstra đề xuất cấu trúc ngăn xếp (Call Stack) cho ngôn ngữ Algol 60. Ngăn xếp giải quyết hoàn hảo bài toán gọi hàm đệ quy và quản lý biến cục bộ theo nguyên tắc vào sau ra trước.

Cùng năm đó, John McCarthy phát minh ra vùng nhớ Heap và bộ thu gom rác cho ngôn ngữ Lisp. Heap cho phép dữ liệu sống tự do, không bị trói buộc vào thời gian chạy của hàm khởi tạo.

Nếu chỉ có Stack, chương trình không thể chia sẻ dữ liệu giữa các luồng hoặc trả đối tượng ra ngoài hàm. Ngược lại, nếu lưu mọi thứ lên Heap, chi phí cấp phát và thu gom rác sẽ làm quá tải CPU.

## Nó thực sự là cái gì
KHÔNG PHẢI kiểu nguyên thủy luôn nằm trên Stack còn đối tượng nằm trên Heap. Thực chất, Stack lưu khung thực thi và biến cục bộ, còn Heap lưu trữ mọi thể hiện đối tượng. Thuộc tính nguyên thủy của một đối tượng vẫn nằm trọn vẹn trên Heap cùng đối tượng đó.

## Cơ chế
Mỗi [luồng](concept:thread) Java sở hữu một Stack riêng (mặc định 1040KB trên ART, luồng chính là 8MB) gồm các khung ngăn xếp. Gọi hàm sẽ đẩy con trỏ ngăn xếp để cấp phát biến cục bộ, và hàm trả về sẽ thu hồi tức thì. Đối tượng thông thường cấp phát trên Heap qua TLAB, còn đối tượng lớn đi vào vùng Large Object Space. Các khung ngăn xếp đóng vai trò là các [GC Root](concept:android-gc-roots) neo giữ đồ thị đối tượng sống trên Heap.

## Đánh đổi
Stack thao tác siêu nhanh ở cấp độ con trỏ thanh ghi CPU nhưng giới hạn dung lượng và gắn chặt theo luồng. Đổi lại, Heap chứa được dữ liệu khổng lồ và chia sẻ đa luồng nhưng phải đánh đổi bằng áp lực dọn rác. Trong [Coroutines](concept:kotlin-coroutine-state-machine), biến sống qua điểm dừng buộc phải nhấc từ Stack lên Heap làm tăng tần suất tạo đối tượng.

## Nguyên lý gốc
Nguyên lý phân vùng vòng đời: dữ liệu ngắn hạn gắn với ngăn xếp, dữ liệu dài hạn đưa vào vùng lưu trữ chung.

## Key insight
> Vị trí lưu do vòng đời quyết định chứ không do kiểu: trường nguyên thủy của đối tượng vẫn nằm trọn vẹn trên Heap.

## Ví von
Stack như xấp giấy ghi chú trên bàn làm việc, dùng xong tờ nào vứt ngay tờ đó. Heap như kho chứa đồ chung của công ty, cần thủ kho định kỳ kiểm kê và vứt bỏ đồ cũ.

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
Đệ quy vô tận hoặc cấp phát biến cục bộ quá sâu sẽ làm tràn Stack và sập với `StackOverflowError`. Cấp phát đối tượng vô tội vạ hoặc giữ tham chiếu chết gây [rò rỉ](concept:android-memory-leak) làm cạn Heap dẫn đến `OutOfMemoryError`. Trên Android 8+, mảng pixel của Bitmap được dời sang Native Heap nên không gây áp lực trực tiếp lên Java Heap. Chống chỉ định biến dữ liệu ngắn hạn thành toàn cục vì sẽ đẩy gánh nặng thu hồi từ Stack sang GC của Heap.

## Câu hỏi thực chiến
**Hỏi:** Kích thước Stack của một luồng trong Android là bao nhiêu?
Mặc định máy ảo ART cấp phát 1040KB cho luồng Java, gồm 1MB vùng nhớ thực và 16KB trang bảo vệ. Luồng chính (Main thread) do Linux khởi tạo có kích thước ngăn xếp mặc định lên tới 8MB.

**Hỏi:** Thuộc tính val age: Int bên trong một class nằm ở Stack hay Heap?
Nằm trên Heap. Thuộc tính của một lớp được đóng gói trực tiếp vào bố cục bộ nhớ của đối tượng đó. Vì thể hiện đối tượng nằm trên Heap, toàn bộ trường nguyên thuỷ của nó cũng nằm trên Heap.

**Hỏi:** Tại sao Coroutine lại phải đưa biến cục bộ từ Stack lên Heap?
Vì coroutine có thể tạm dừng và nhả luồng cho tác vụ khác thực thi. Khung Stack bị dọn dẹp khi hàm nhả luồng, nên biến sống bắt buộc phải cất vào đối tượng Continuation trên Heap.

**Hỏi:** Pixel của đối tượng Bitmap trên Android nằm ở Stack, Java Heap hay Native Heap?
Từ Android 8.0 trở đi, mảng pixel của Bitmap được lưu hoàn toàn trên Native Heap của hệ điều hành. Java Heap chỉ giữ một đối tượng vỏ bọc `Bitmap` rất nhẹ mang con trỏ trỏ sang bộ nhớ native.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: Stack lưu khung hàm theo luồng, Heap lưu đối tượng; trường nguyên thuỷ trong object nằm trên Heap.
- [ ] Số liệu ART: Stack luồng Java là 1040KB, Main thread là 8MB, Bitmap chuyển sang Native Heap từ Android 8.
- [ ] Cơ chế Coroutine nhấc biến từ Stack lên Heap Continuation qua điểm ngắt.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
