---
id: jvm-virtual-method-dispatch
title: Virtual Method Table & Dynamic Dispatch
domain: jvm
tags: [jvm, oop, bytecode, performance]
collections: []
related: [oop-four-pillars, oop-solid, kotlin-inline-function]
created: 2026-09-08
diagrams: [Cơ chế|virtual-method-dispatch.png|Lệnh invokevirtual đọc con trỏ Header sang vtable trong Metaspace rồi nhảy tới mã máy thực thi]
---

## Bản 30 giây
Khi gọi hàm đa hình, máy tính không biết trước phương thức của lớp con nào sẽ thực sự chạy. Máy ảo giải quyết việc này bằng cách tra cứu bảng danh bạ địa chỉ hàm gắn với lớp. Nhờ đó, chương trình gọi đúng hàm lớp con tại thời điểm chạy mà không cần sửa mã nguồn cũ.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Năm 1967, Ole-Johan Dahl và Kristen Nygaard phát minh ngôn ngữ Simula 67 với khái niệm lớp kế thừa và phương thức ảo đầu tiên.

Trong ngôn ngữ C, lập trình viên phải tự lưu con trỏ hàm vào từng `struct` để xử lý đa hình. Khi có nhiều hàm, nhân bản con trỏ trên mỗi biến làm bùng nổ RAM và dễ sập do trỏ nhầm địa chỉ.

Năm 1983, Bjarne Stroustrup tạo ra vtable trong C++, rồi James Gosling đưa vào máy ảo JVM năm 1995. Vtable gom các con trỏ hàm vào bảng duy nhất cho mỗi lớp, triệt tiêu sự lãng phí bộ nhớ trên từng đối tượng.

## Nó thực sự là cái gì
Đa hình động KHÔNG PHẢI phép màu tự động chọn hàm lúc chạy. Thực chất nó là mảng con trỏ hàm nằm trong Class Metadata thuộc vùng nhớ Metaspace ngoài Heap. Trình biên dịch gán một chỉ số cố định cho mỗi hàm ảo và phát lệnh bytecode `invokevirtual` tại điểm gọi. Máy ảo đọc chỉ số này để truy xuất địa chỉ hàm thực tế từ mảng và nhảy tới thực thi.

## Cơ chế
Header của đối tượng trên Heap chứa con trỏ trỏ tới cấu trúc lớp nằm trong vùng nhớ Metaspace. Lệnh `invokevirtual` dereference hai lần: từ Heap sang Metaspace lấy vtable, rồi từ vtable tới mã máy. Trình JIT dùng inline cache lưu thẳng đích gọi để bỏ qua bước tra vtable nếu chỉ có một lớp. Trên Android ART, cấu trúc `ArtMethod` và vtable cũng nằm tại vùng nhớ native LinearAlloc tương tự Metaspace.

## Đánh đổi
Bạn có được kiến trúc mã nguồn linh hoạt tuyệt đối theo [Bốn tính chất OOP](concept:oop-four-pillars) và [SOLID](concept:oop-solid). Đổi lại, chương trình mất thêm hai bước đọc bộ nhớ gián tiếp cho mỗi lần gọi hàm. Lời gọi qua bảng cản trở tối ưu inline hàm khi có quá nhiều lớp con cùng chia sẻ một điểm gọi.

## Nguyên lý gốc
Nguyên lý tầng trung gian gián tiếp (Indirection). Mọi bài toán kiến trúc đều giải quyết được bằng một tầng trung gian, nhưng luôn phải trả giá bằng độ trễ truy xuất.

## Key insight
> Vtable đánh đổi hai bước đọc bộ nhớ gián tiếp để lấy khả năng hoán đổi hành vi hàm linh hoạt lúc chạy.

## Ví von
Giống như danh bạ điện thoại tại sảnh lễ tân tòa nhà lớn. Thay vì nhớ số riêng từng phòng, bạn tra cứu vị trí cố định trên bảng để lấy số máy nhánh. Bạn gọi được đúng người nhưng phải tốn thêm thời gian cho một bước kết nối trung gian.

## Khuôn mẫu chuẩn
```kotlin
open class Payment { open fun process(): Int = 0 }
class FastPay : Payment() { override fun process(): Int = 1 }
class SlowPay : Payment() { override fun process(): Int = 2 }

fun checkout(p: Payment): Int {
    // invokevirtual: CPU đọc p.vtable[offset]
    return p.process()
}
```

## Bẫy & ranh giới
Tầng gián tiếp từng cứu rỗi RAM nay trở thành nút thắt cổ chai của CPU trong các vòng lặp nặng. Điểm gọi megamorphic làm JIT đầu hàng, vô hiệu hóa inline cache và không thể inline hàm. Chống chỉ định mở rộng kế thừa sâu trong các đoạn mã đòi hỏi xử lý thời gian thực. Hãy tận dụng triết lý `final by default` của Kotlin để trình biên dịch chủ động tối ưu hóa gộp mã nguồn trực tiếp.

## Câu hỏi thực chiến
**Hỏi:** Trong Kotlin, mọi lời gọi hàm đều phải tra qua vtable có đúng không?
Không phải. Hàm Kotlin mặc định là `final`, nên JIT hoặc AOT biết chắc chỉ có một đích để gọi thẳng, thậm chí inline luôn. Bytecode vẫn là `invokevirtual`; chỉ hàm `open` hoặc `override` mới thực sự cần tra vtable lúc chạy.

**Hỏi:** Gọi hàm qua interface có dùng chung bảng vtable với lớp cha không?
Không. Gọi qua interface dùng lệnh `invokeinterface` và một bảng riêng: HotSpot gọi là itable, còn ART trên Android dùng IMT. Vì một lớp có thể triển khai nhiều interface theo thứ tự bất kỳ, vị trí hàm không cố định như trên vtable.

**Hỏi:** Khi nào việc tra cứu vtable thực sự làm chậm ứng dụng?
Khi một điểm gọi nóng nhận quá nhiều kiểu đối tượng khác nhau, gọi là megamorphic. Với một hoặc hai kiểu, JIT của HotSpot ghi nhớ và inline luôn thân hàm. Từ ba kiểu trở lên, nó bỏ inline và tra bảng mỗi lần gọi, mất luôn các tối ưu đi kèm.

**Hỏi:** Dùng abstract class hay interface sẽ có tốc độ gọi hàm nhanh hơn?
Abstract class thường nhanh hơn một chút khi chưa được JIT tối ưu. Kế thừa đơn cho phép máy ảo gán chỉ số cố định trên vtable và tra cứu bằng một phép cộng offset. Tuy nhiên, trong thực tế trình JIT sẽ tối ưu hoá cả hai nếu điểm gọi chỉ có đúng một lớp con.

**Hỏi:** Tại sao các hàm private hoặc static lại không có tên trong vtable?
Vì chúng không bao giờ bị lớp con ghi đè. Trình biên dịch xác định được chính xác hàm cần chạy ngay lúc biên dịch mà không cần đợi chương trình chạy. Vì vậy nó phát lệnh gọi trực tiếp: `invokestatic` cho hàm static, còn trên Android hàm private dùng `invoke-direct`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vtable là mảng con trỏ hàm trong Class Metadata ở Metaspace.
- [ ] Trình bày được hai bước dereference bộ nhớ khi chạy lệnh bytecode invokevirtual.
- [ ] Phân tích được tổn thất hiệu năng CPU khi điểm gọi rơi vào trạng thái megamorphic.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
