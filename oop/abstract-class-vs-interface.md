---
id: oop-abstract-class-vs-interface
title: Abstract class đối lập Interface
domain: oop
tags: [oop, architecture, polymorphic, design-pattern]
collections: [interview-fundamental]
related: [oop-four-pillars, jvm-virtual-method-dispatch]
created: 2026-09-11
diagrams: [Cơ chế|abstract-class-vs-interface.png|Lớp trừu tượng sở hữu constructor và backing field trên Heap; giao diện chỉ định hình hợp đồng năng lực không trạng thái]
---

## Bản 30 giây
Khi tái sử dụng mã nguồn, lập trình viên phân vân giữa việc kế thừa lớp hay dùng giao diện. Giao diện trao quyền sở hữu nhiều hợp đồng năng lực nhưng tuyệt đối không được nắm giữ dữ liệu. Lớp trừu tượng cho phép lưu trữ trạng thái nhưng trói buộc đối tượng vào một phả hệ cha con duy nhất.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Thập niên 1980, C++ cho phép đa kế thừa khiến chương trình đối mặt thảm họa kim cương (Diamond Problem). Khi hai lớp cha cùng chứa một biến, lớp con kế thừa cả hai sẽ gây xung đột dữ liệu không thể phân giải.

Năm 1995, James Gosling thiết kế Java chỉ cho phép đơn kế thừa lớp nhằm triệt tiêu hoàn toàn xung đột trạng thái. Đồng thời, ông tách riêng giao diện (interface) không mang trạng thái để một lớp có thể tự do đón nhận nhiều năng lực.

Java 8 năm 2014 và Kotlin năm 2016 bổ sung hàm có thân vào giao diện. Cải tiến này làm mờ ranh giới mã lệnh, khiến nhiều người ngộ nhận hai khái niệm đã tương đương nhau.

## Nó thực sự là cái gì
Khác biệt KHÔNG PHẢI là việc có chứa mã lệnh hay không vì giao diện hiện đại đã cho phép hàm có thân. Thực chất đây là ranh giới sở hữu bộ khởi tạo constructor và biến thành viên lưu trữ dữ liệu (backing field). Lớp trừu tượng định hình danh tính thực thể (IS-A), còn giao diện chỉ đóng gói hợp đồng khả năng hành vi (CAN-DO).

## Cơ chế
Lớp trừu tượng cấp phát vùng nhớ thực thể trên Heap chứa các biến và gọi constructor để thiết lập trạng thái ban đầu. Giao diện không sở hữu trường dữ liệu nên hoàn toàn không tham gia vào cấu trúc cấp phát bộ nhớ của đối tượng. Khi chạy, máy ảo phát lệnh bytecode invokevirtual để đọc bảng con trỏ hàm theo chỉ số cố định cho lớp trừu tượng. Với giao diện, máy ảo phải dùng lệnh invokeinterface để tra bảng giao diện itable động qua [Virtual method dispatch](concept:jvm-virtual-method-dispatch).

## Đánh đổi
Giao diện mang lại sự linh hoạt tối đa khi một lớp có thể cài đặt vô số hợp đồng hành vi khác nhau. Đổi lại, lệnh invokeinterface tốn thêm chu kỳ CPU để dò tìm bảng itable và không thể chia sẻ trạng thái chung. Lớp trừu tượng giúp tái sử dụng biến và chạy hàm nhanh, nhưng làm mất suất kế thừa đơn duy nhất của lớp con.

## Nguyên lý gốc
Đa thừa kế an toàn với hợp đồng nhưng nguy hiểm với trạng thái. Hệ thống tách danh tính khỏi năng lực. Thứ mang dữ liệu thì chỉ được nhận một; thứ chỉ hứa hành vi thì nhận bao nhiêu cũng được.

## Key insight
> Khác biệt giữa abstract class và interface không nằm ở mã lệnh, mà ở quyền sở hữu constructor và trường lưu trữ trạng thái.

## Ví von
Lớp trừu tượng giống như Căn cước công dân xác định danh tính bản thể duy nhất với các đặc điểm nhân dạng cố định. Giao diện giống như bằng lái xe hay thẻ thư viện, một công dân có thể mang theo nhiều chứng chỉ năng lực tùy ý.

## Khuôn mẫu chuẩn
```kotlin
interface Flyable { fun fly() = "Flying" }
abstract class Animal(val name: String) { abstract fun sound(): String }
class Duck(name: String) : Animal(name), Flyable {
    override fun sound() = "Quack"
}
val d = Duck("Mallard")
val action = "${d.name}: ${d.sound()}, ${d.fly()}"
```

## Bẫy & ranh giới
Bẫy lạm dụng lớp trừu tượng để gom mã dùng chung khiến lớp con lãng phí mất suất kế thừa đơn duy nhất. Bẫy giả định giao diện không thể có mã lệnh sẽ dẫn đến việc tạo các lớp bao bọc trung gian không cần thiết. Chống chỉ định kế thừa lớp khi chỉ cần chia sẻ năng lực hành vi mà không cần quản lý vòng đời biến trạng thái. Triệu chứng cấu trúc là phả hệ kế thừa phân tầng quá sâu gây hiệu ứng Fragile Base Class làm vỡ hành vi lớp con.

## Câu hỏi thực chiến
**Hỏi:** Từ khi Interface cho phép viết hàm có thân thì Abstract Class đã hoàn toàn vô dụng?
Không phải. Giao diện có thể chứa hàm có thân nhưng tuyệt đối không được phép sở hữu constructor hay biến lưu dữ liệu (backing field). Khi cần quản lý trạng thái nội tại hoặc vòng đời khởi tạo biến, Abstract Class vẫn là công cụ bắt buộc.

**Hỏi:** Làm sao biết khi nào nên tạo Interface, khi nào nên kế thừa Abstract Class?
Dùng Interface khi mô tả năng lực hành vi, dùng Abstract Class khi định nghĩa danh tính kèm trạng thái nội tại. Ví dụ năng lực in ấn hoặc lắng nghe sự kiện nên là Interface để lớp con giữ lại suất đơn kế thừa. Ngược lại, nếu các lớp con có chung phả hệ nguồn gốc và cần chia sẻ biến trạng thái, hãy dùng Abstract Class.

**Hỏi:** Vì sao trên JVM, gọi hàm qua Interface lại chậm hơn gọi hàm qua Abstract Class?
Chỉ chậm hơn chút ít khi chưa được JIT tối ưu. `invokeinterface` phải dò bảng riêng (itable trên HotSpot, IMT trên ART), còn `invokevirtual` tra chỉ số cố định trong vtable. Khi điểm gọi chỉ gặp một vài kiểu, JIT inline luôn và khác biệt gần như biến mất.

**Hỏi:** Một lớp kế thừa hai Interface có hàm trùng tên và cùng có thân mã lệnh thì xử lý thế nào?
Trình biên dịch bắt buộc lớp con phải ghi đè lại hàm đó để tự giải quyết xung đột. Trong thân hàm ghi đè, bạn có thể chỉ định rõ gọi thân hàm của bên nào bằng cú pháp `super<T>.ham()`. Quy tắc minh bạch này giúp loại bỏ hoàn toàn nguy cơ mơ hồ lệnh của thảm hoạ kim cương trong C++.

**Hỏi:** Có nên tạo BaseViewModel hay BaseFragment dạng Abstract Class để dùng chung code không?
Không nên lạm dụng. Tạo các lớp Base gom mã chung thường dẫn đến bẫy Fragile Base Class và trói buộc lớp con vào phả hệ cồng kềnh. Thay vì kế thừa, kiến trúc hiện đại ưu tiên [lắp ghép](concept:oop-composition-over-inheritance) các lớp tiện ích nhỏ độc lập vào nơi cần dùng.

## Ý bắt buộc khi giải thích lại
- [ ] Phân định rõ ràng: abstract class sở hữu constructor và backing field lưu trạng thái, interface chỉ định nghĩa hợp đồng hành vi.
- [ ] Nói ra được trong hai câu: hai lớp cha cùng một field thì con không biết lấy bản nào. Hợp đồng không mang dữ liệu nên nhận bao nhiêu cũng không xung đột.
- [ ] Nêu bản chất bytecode: invokevirtual tra vtable theo index cố định, invokeinterface tra itable động qua [Virtual method dispatch](concept:jvm-virtual-method-dispatch).
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
