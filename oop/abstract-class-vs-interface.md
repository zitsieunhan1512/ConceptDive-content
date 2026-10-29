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
Khi viết phần dùng chung cho nhiều lớp, bạn thường phân vân giữa kế thừa `abstract class` hay cài đặt `interface`. `Interface` cho phép một lớp nhận nhiều hợp đồng hành vi cùng lúc nhưng không được tạo ô nhớ lưu biến. Còn `abstract class` giữ được biến trạng thái trên Heap nhưng bắt lớp con dùng mất suất kế thừa duy nhất.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Thập niên 1980, C++ cho phép một lớp kế thừa nhiều lớp cha nên dẫn đến bài toán kim cương (Diamond Problem). Khi hai lớp cha cùng có một biến trùng tên, lớp con kế thừa cả hai sẽ không biết dùng biến nào.

Năm 1995, James Gosling thiết kế Java chỉ cho phép kế thừa một lớp để dẹp hẳn lỗi đụng độ biến trạng thái. Cùng lúc đó, ông tách riêng `interface` không giữ biến để một lớp nhận được nhiều khả năng hành vi.

Đến Java 8 (2014) và Kotlin (2016), `interface` cũng viết được hàm có thân lệnh. Thay đổi này làm nhiều bạn tưởng nhầm hai thứ đã giống hệt nhau.

## Nó thực sự là cái gì
Khác biệt không nằm ở chuyện hàm có thân lệnh hay không, vì `interface` nay đã viết được thân hàm. Điểm khác thật sự là quyền có `constructor` và quyền giữ ô nhớ lưu biến (`backing field`) trên Heap. `Abstract class` cho biết đối tượng thuộc dòng họ nào (IS-A), còn `interface` chỉ gắn thêm khả năng làm việc (CAN-DO).

## Cơ chế
`Abstract class` cấp ô nhớ trên Heap để chứa biến và gọi `constructor` gán giá trị ban đầu. `Interface` không có trường dữ liệu nên không chiếm thêm ô nhớ nào trong đối tượng. Lúc chạy, máy ảo dùng lệnh `invokevirtual` nhảy thẳng tới vị trí cố định trong bảng `vtable` để gọi hàm của `abstract class`. Còn gọi hàm qua `interface` thì máy ảo dùng `invokeinterface` để dò bảng `itable` qua [Virtual method dispatch](concept:jvm-virtual-method-dispatch).

## Đánh đổi
Dùng `interface` rất linh hoạt vì một lớp có thể ghép bao nhiêu hợp đồng hành vi tùy ý. Đổi lại, lệnh `invokeinterface` tốn thêm bước dò bảng `itable` và không giữ được biến trạng thái chung. Còn `abstract class` cho dùng chung biến và gọi hàm nhanh hơn, nhưng lấy mất suất kế thừa duy nhất của lớp con.

## Nguyên lý gốc
Kế thừa nhiều nơi chỉ an toàn với lời hứa hành vi, còn rất nguy hiểm khi mang theo biến dữ liệu. Vì vậy ngôn ngữ tách riêng danh tính và khả năng làm việc. Thứ giữ biến trên bộ nhớ thì chỉ được nhận một cha, còn thứ chỉ hứa hành vi thì nhận bao nhiêu cũng được.

## Key insight
> Khác biệt giữa `abstract class` và `interface` không nằm ở thân hàm, mà ở quyền có `constructor` và ô nhớ `backing field`.

## Ví von
`Abstract class` giống căn cước công dân: mỗi người chỉ có một bản ghi rõ dòng máu và đặc điểm riêng. `Interface` giống bằng lái xe hay thẻ thư viện: một người có thể thi lấy bao nhiêu chứng chỉ tùy thích.

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
Nếu bạn tạo `abstract class` chỉ để gom vài hàm tiện ích, lớp con sẽ mất oan suất kế thừa duy nhất. Ngược lại, tưởng `interface` không viết được thân hàm sẽ làm bạn đẻ thêm các lớp bọc trung gian thừa thãi. Đừng kế thừa lớp khi chỉ cần chia sẻ hành vi mà không giữ biến trạng thái nào. Triệu chứng là cây kế thừa sâu nhiều tầng, sửa một dòng ở lớp cha làm hỏng dây chuyền lớp con (Fragile Base Class).

## Câu hỏi thực chiến
**Hỏi:** Từ khi Interface cho phép viết hàm có thân thì Abstract Class đã hoàn toàn vô dụng?
Không phải. `Interface` viết được hàm có thân nhưng vẫn không được có `constructor` hay ô nhớ lưu biến (`backing field`). Khi bạn cần giữ biến trạng thái chung hoặc chạy logic khởi tạo, `abstract class` vẫn là lựa chọn bắt buộc.

**Hỏi:** Làm sao biết khi nào nên tạo Interface, khi nào nên kế thừa Abstract Class?
Hãy dùng `interface` khi gắn thêm khả năng làm việc, và dùng `abstract class` khi các lớp cùng một dòng họ có biến chung. Ví dụ khả năng bấm chọn hay gửi log nên viết thành `interface` để lớp con giữ lại suất kế thừa. Ngược lại, nếu các lớp con cùng bản chất và cần dùng chung biến trạng thái thì mới kế thừa `abstract class`.

**Hỏi:** Vì sao trên JVM, gọi hàm qua Interface lại chậm hơn gọi hàm qua Abstract Class?
Nó chỉ chậm hơn một chút khi đoạn code đó chưa được trình biên dịch JIT tối ưu. Lệnh `invokeinterface` phải dò tìm trong bảng `itable` (hoặc `IMT` trên ART), còn `invokevirtual` lấy ngay theo vị trí cố định trong `vtable`. Khi chỗ gọi thực tế chỉ gặp một vài lớp cụ thể, JIT sẽ chép thẳng hàm vào nên tốc độ gần như bằng nhau.

**Hỏi:** Một lớp kế thừa hai Interface có hàm trùng tên và cùng có thân mã lệnh thì xử lý thế nào?
Trình biên dịch sẽ báo lỗi ngay và bắt lớp con phải tự viết lại (`override`) hàm bị trùng đó. Bên trong hàm viết lại, bạn chỉ rõ muốn gọi bản của bên nào bằng cú pháp `super<T>.ham()`. Nhờ bắt chọn rõ ràng lúc biên dịch, ngôn ngữ tránh được cảnh nhập nhằng của bài toán kim cương trong C++.

**Hỏi:** Có nên tạo BaseViewModel hay BaseFragment dạng Abstract Class để dùng chung code không?
Không nên lạm dụng. Nhét đủ thứ hàm dùng chung vào một lớp `Base` rất dễ làm sửa lớp cha thì vỡ lớp con (Fragile Base Class). Thay vì kế thừa, bạn nên [lắp ghép](concept:oop-composition-over-inheritance) các lớp nhỏ độc lập vào đúng màn hình nào thực sự cần.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được: abstract class có constructor và ô nhớ backing field trên Heap, còn interface chỉ giữ hợp đồng hành vi.
- [ ] Giải thích được bài toán kim cương: hai lớp cha có cùng biến thì con bị đụng độ, còn hợp đồng không biến thì ghép thoải mái.
- [ ] Nêu được cơ chế bytecode: invokevirtual tra vtable theo vị trí cố định, còn invokeinterface dò bảng itable qua [Virtual method dispatch](concept:jvm-virtual-method-dispatch).
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
