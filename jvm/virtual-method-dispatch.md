---
id: jvm-virtual-method-dispatch
title: Virtual Method Table & Dynamic Dispatch
domain: jvm
tags: [jvm, oop, bytecode, performance]
collections: []
related: [oop-four-pillars, oop-solid, kotlin-inline-function]
created: 2026-09-08
sources:
  - JVMS Chapter 6: invokevirtual instruction | https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html
  - Virtual method table — Wikipedia | https://en.wikipedia.org/wiki/Virtual_method_table
  - Dynamic dispatch — Wikipedia | https://en.wikipedia.org/wiki/Dynamic_dispatch
diagrams: [Cơ chế|virtual-method-dispatch.png|Lệnh invokevirtual đọc con trỏ Header sang vtable trong Metaspace rồi nhảy tới mã máy thực thi]
---

## Bản 30 giây
Khi bạn gọi `payment.process()` trên biến kiểu cha, lúc dịch code máy tính chưa biết đối tượng thật là lớp con nào. Máy ảo giải quyết bằng cách gắn cho mỗi lớp một bảng danh bạ chứa địa chỉ các hàm, gọi là `vtable`. Lúc chạy, nó chỉ cần nhìn vào đối tượng để tra bảng và nhảy tới đúng hàm của lớp con.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Năm 1967, Ole-Johan Dahl và Kristen Nygaard tạo ra ngôn ngữ Simula 67 với khái niệm kế thừa lớp và hàm ảo đầu tiên.

Trong ngôn ngữ C, muốn làm đa hình, bạn phải tự cất các con trỏ hàm vào bên trong từng biến `struct`. Khi tạo hàng triệu biến thuộc lớp nhiều hàm, việc lặp lại con trỏ gây tốn RAM và dễ trỏ nhầm.

Năm 1983, Bjarne Stroustrup đưa `vtable` vào C++, rồi James Gosling mang cơ chế này vào máy ảo JVM năm 1995. Nhờ gom con trỏ hàm vào bảng chung cho từng lớp, mỗi đối tượng không phải ôm theo danh sách hàm.

## Nó thực sự là cái gì
Đa hình động không phải phép màu tự đoán hàm lúc chạy, mà là một mảng chứa các con trỏ hàm gọi là `vtable`. Bảng này nằm trong phần thông tin lớp (`Class Metadata`) ở vùng nhớ Metaspace bên ngoài Heap. Lúc biên dịch, mỗi hàm ảo được đánh một số thứ tự cố định và chỗ gọi hàm sinh ra lệnh bytecode `invokevirtual`. Khi chạy tới đó, máy ảo lấy địa chỉ hàm tại đúng ô số thứ tự trong mảng `vtable` rồi nhảy vào chạy.

## Cơ chế
Phần đầu (`header`) của mỗi đối tượng trên Heap có giữ một con trỏ chỉ sang thông tin lớp ở vùng Metaspace. Gặp lệnh `invokevirtual`, CPU đi từ đối tượng sang Metaspace tìm `vtable`, rồi từ ô trong `vtable` nhảy tới mã máy. Nếu điểm gọi chỉ gặp một lớp con, trình JIT dùng bộ nhớ đệm (`inline cache`) nhớ địa chỉ đích để khỏi tra bảng. Trên Android ART, cấu trúc `ArtMethod` và bảng `vtable` cũng nằm ở vùng nhớ native `LinearAlloc` tương tự như Metaspace.

## Đánh đổi
Nhờ tra bảng lúc chạy, bạn truyền được các lớp con khác nhau vào cùng một hàm theo [Bốn tính chất OOP](concept:oop-four-pillars) và [SOLID](concept:oop-solid). Đổi lại, mỗi lần gọi hàm ảo tốn thêm hai bước đọc bộ nhớ gián tiếp qua con trỏ. Khi chỗ gọi nhận quá nhiều lớp con, trình biên dịch không thể chép thẳng thân hàm (`inline`) vào đó nữa.

## Nguyên lý gốc
Muốn tránh khóa cứng vào một lớp cụ thể thì đặt bảng tra cứu ở giữa. Đổi lại, mỗi lần gọi phải bước qua bảng đó (Indirection).

## Key insight
> Vtable đánh đổi hai bước đọc bộ nhớ gián tiếp để lấy khả năng hoán đổi hành vi hàm linh hoạt lúc chạy.

## Ví von
Giống như bảng danh bạ ở quầy lễ tân tòa nhà lớn. Thay vì thuộc số từng người, bạn chỉ nhìn ô cố định của phòng đó trên bảng để lấy số máy lẻ. Bạn luôn gọi đúng người mới chuyển vào phòng, nhưng tốn thêm một nhịp nhìn bảng.

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
Trong vòng lặp chạy hàng triệu lần, điểm gọi nhận từ 3 lớp con trở lên sẽ rơi vào đa hình quá mức (`megamorphic`). Lúc này JIT xóa `inline cache`, không chép thẳng thân hàm được mà phải tra bảng `vtable` ở từng vòng. Đừng chia cây kế thừa quá nhiều nhánh cho những hàm tính toán nặng cần chạy liên tục theo thời gian thực. Hãy giữ mặc định `final` của Kotlin và chỉ mở `open` khi thật sự cần ghi đè để trình biên dịch gọi thẳng nhanh nhất.

## Câu hỏi thực chiến
**Hỏi:** Trong Kotlin, mọi lời gọi hàm đều phải tra qua vtable có đúng không?
Không phải. Hàm trong Kotlin mặc định là `final`, nên JIT hoặc AOT gọi thẳng hoặc chép luôn thân hàm vào chỗ gọi. Dù bytecode vẫn ghi là `invokevirtual`, chỉ những hàm bạn đánh dấu `open` hoặc `override` mới thật sự phải tra bảng `vtable` lúc chạy.

**Hỏi:** Gọi hàm qua interface có dùng chung bảng vtable với lớp cha không?
Không. Khi gọi qua biến interface, máy ảo dùng lệnh `invokeinterface` và tra bảng riêng: `itable` trên HotSpot hoặc `IMT` trên ART. Do một lớp cài đặt nhiều interface theo thứ tự bất kỳ, hàm interface không có vị trí cố định như `vtable`.

**Hỏi:** Khi nào việc tra cứu vtable thực sự làm chậm ứng dụng?
Khi một dòng gọi hàm nằm trong vòng lặp nóng mà lại nhận đủ loại đối tượng thuộc từ 3 lớp con trở lên (`megamorphic`). Nếu chỉ gặp 1 hoặc 2 lớp con, trình JIT của HotSpot nhớ luôn địa chỉ và chép thẳng thân hàm vào chỗ gọi. Nhưng từ 3 lớp con trở lên, nó phải tra bảng `vtable` mỗi vòng và mất hết các tối ưu.

**Hỏi:** Dùng abstract class hay interface sẽ có tốc độ gọi hàm nhanh hơn?
Nếu trình JIT chưa can thiệp tối ưu, gọi qua `abstract class` thường nhanh hơn một nhịp. Vì mỗi lớp chỉ có một cha, máy ảo biết sẵn vị trí ô trên `vtable` và chỉ cần cộng độ lệch (`offset`). Dù vậy, nếu trong lúc chạy chỗ đó chỉ truyền vào đúng một lớp con, trình JIT sẽ tối ưu cả hai nhanh ngang nhau.

**Hỏi:** Tại sao các hàm private hoặc static lại không có tên trong vtable?
Vì hàm `private` và `static` không bao giờ bị lớp con ghi đè. Ngay lúc dịch code, trình biên dịch đã biết đích xác phải gọi hàm nào mà không cần chờ tới lúc chạy. Do đó nó sinh lệnh gọi thẳng luôn: `invokestatic` cho hàm tĩnh, còn trên Android thì hàm `private` dùng lệnh `invoke-direct`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vtable là mảng chứa địa chỉ hàm nằm trong phần thông tin lớp ở vùng nhớ Metaspace.
- [ ] Trình bày được hai bước đọc con trỏ khi chạy lệnh invokevirtual: từ đối tượng sang Metaspace lấy vtable, rồi từ vtable tới mã máy.
- [ ] Phân tích được vì sao điểm gọi nhận từ 3 lớp con trở lên (megamorphic) lại làm mất inline cache và chậm vòng lặp.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
