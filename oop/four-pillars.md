---
id: oop-four-pillars
title: Bốn tính chất OOP
domain: oop
tags: [oop, paradigm, software-design]
collections: [interview-fundamental]
related: [oop-solid, jvm-virtual-method-dispatch, oop-abstract-class-vs-interface]
created: 2026-09-07
---

## Bản 30 giây
Khi phần mềm lớn dần, việc phụ thuộc trực tiếp vào chi tiết của nhau sẽ gây đổ vỡ dây chuyền. Thay vì tự do truy cập, chương trình giao tiếp qua các thông điệp và giao diện trừu tượng được bảo vệ. Bốn tính chất là bộ khung giúp cô lập biến động và kiểm soát ranh giới phụ thuộc lúc chạy.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1967, Ole-Johan Dahl và Kristen Nygaard phát triển Simula 67 cho các bài toán mô phỏng sự kiện rời rạc. Lập trình thủ tục dùng biến rời rạc làm việc theo dõi trạng thái các thực thể độc lập trở thành thảm họa logic. Họ gom biến và lệnh vào một thực thể gọi là đối tượng (object).

Khoảng năm 1966 đến 1967, Alan Kay đặt ra thuật ngữ lập trình hướng đối tượng tại Đại học Utah trước khi phát triển Smalltalk. Ông nhấn mạnh bản chất là message passing (truyền thông điệp), đóng gói trạng thái và late binding.

Thập niên 1990, C++ và Java bùng nổ trong giảng dạy. Các giáo trình đúc kết thành bốn tính chất: đóng gói, trừu tượng, kế thừa và đa hình. Cách dạy này biến chúng thành lý thuyết cú pháp, xa rời mô hình sinh học truyền thông điệp ban đầu của Kay.

## Nó thực sự là cái gì
Bốn tính chất OOP KHÔNG phải là định nghĩa nguyên bản của Alan Kay. Thực chất, chúng là phân loại do giáo trình thập niên 1990 đúc kết lại để dạy học cho các ngôn ngữ class-based. Thay vì nhấn mạnh message passing, bốn cột trụ này lập quy tắc kiểm soát phụ thuộc mã nguồn.

## Cơ chế
Đóng gói dùng từ khóa truy cập để phong tỏa dữ liệu, còn trừu tượng để lộ giao diện tương tác tối thiểu. Kế thừa sao chép memory layout lớp cha vào con, khác với hợp đồng không trạng thái trong [Abstract class đối lập Interface](concept:oop-abstract-class-vs-interface). Đa hình dùng [bảng phương thức ảo (vtable)](concept:jvm-virtual-method-dispatch) để phân giải con trỏ hàm lúc chạy. Nhờ đó bên gọi hoàn toàn không bị trói chặt vào bất kỳ lớp cụ thể nào.

## Đánh đổi
Hệ thống đạt tính mô-đun cao và an toàn dữ liệu. Đổi lại, việc tạo nhiều lớp trung gian làm tăng chi phí đọc code. Tra cứu bảng hàm ảo vtable và cấp phát đối tượng trên Heap tốn tài nguyên hơn hàm tuần tự. Nguy hiểm nhất, lạm dụng kế thừa thay vì lắp ghép (composition) sẽ tạo liên kết cứng, phá vỡ chính tính đóng gói.

## Nguyên lý gốc
Nguyên lý che giấu thông tin do David Parnas đề xuất năm 1972. Mỗi khối phần mềm giấu kín cấu trúc bên trong và chỉ để lộ giao diện tối thiểu.

## Key insight
> OOP không nhằm tái sử dụng mã qua kế thừa, mà để cô lập biến động bằng ranh giới đóng gói và đa hình.

## Ví von
Hệ thống lái ô tô: Đóng gói giấu động cơ dưới nắp capo. Trừu tượng là vô lăng và chân phanh. Kế thừa là xe điện và xe xăng cùng là ô tô. Đa hình là đạp phanh thì xe nào cũng dừng, dù bên dưới dùng má phanh cơ hay phanh điện tái sinh.

## Khuôn mẫu chuẩn
```kotlin
// Minh họa đóng gói, kế thừa và điểm gọi đa hình
open class Vehicle(private var speed: Int) {
    fun drive() { speed += 10 }
    open fun horn(): String = "Beep"
}
class Truck : Vehicle(0) {
    override fun horn(): String = "Honk"
}
fun alert(v: Vehicle): String = v.horn()
```

## Bẫy & ranh giới
Coi kế thừa là cốt lõi thay vì message passing dẫn tới bẫy lạm dụng kế thừa (Fragile Base Class). Ghi đè hàm thêm danh sách nhưng lớp cha tự gọi lại hàm thêm đơn lẻ làm kết quả đếm bị nhân đôi. Để dẹp bỏ bẫy này, Kotlin mặc định khóa mọi class bằng `final` và hỗ trợ ủy quyền qua từ khóa `by`. Chống chỉ định kế thừa để tái sử dụng mã; hãy ưu tiên composition để bảo toàn tính đóng gói.

## Câu hỏi thực chiến
**Hỏi:** Mục đích chính của OOP có phải là để tái sử dụng code thông qua kế thừa không?
Không phải. Mục đích cốt lõi của OOP là cô lập các biến động để khi thay đổi một chỗ không làm đổ vỡ nơi khác. Lạm dụng kế thừa để tái sử dụng mã thường phá huỷ tính đóng gói và làm hệ thống trở nên cứng nhắc.

**Hỏi:** Khi nào nên dùng kế thừa, khi nào nên dùng lắp ghép (composition)?
Hầu như luôn ưu tiên lắp ghép, chỉ kế thừa khi mối quan hệ là IS-A thực sự bất biến. Nếu bạn chỉ muốn tận dụng vài tính năng của lớp khác, hãy biến nó thành một biến phụ thuộc bên trong. Trong Kotlin, bạn có thể dùng từ khoá `by` để uỷ quyền triển khai giao diện mà không tạo liên kết cứng.

**Hỏi:** Trừu tượng (Abstraction) khác Đóng gói (Encapsulation) chỗ nào khi cả hai đều giấu thông tin?
Đóng gói là bảo vệ dữ liệu nội tại, còn Trừu tượng là che giấu độ phức tạp của cách cài đặt. Đóng gói dùng từ khoá private để ngăn truy cập và biến đổi biến trạng thái trái phép từ bên ngoài. Trừu tượng cung cấp giao diện tương tác tối giản để bên gọi sử dụng mà không cần biết thuật toán bên dưới.

**Hỏi:** Đa hình (Polymorphism) vận hành ở tầng máy ảo như thế nào để biết phải gọi hàm nào?
Máy ảo tra cứu địa chỉ hàm trong [bảng phương thức ảo (vtable)](concept:jvm-virtual-method-dispatch) gắn với đối tượng thực tế lúc chạy. Mỗi đối tượng trên Heap đều chứa một con trỏ tham chiếu đến cấu trúc lớp và bảng hàm tương ứng. Khi có lệnh gọi, CPU đọc con trỏ từ bảng này rồi nhảy đến đoạn mã thực thi của lớp con.

**Hỏi:** Vì sao Kotlin lại mặc định khoá mọi class bằng final thay vì mở như Java?
Để ngăn ngừa bẫy Fragile Base Class bắt nguồn từ thói quen lạm dụng kế thừa tự do trong Java. Khi kế thừa không kiểm soát, sửa đổi ngầm ở lớp cha rất dễ làm biến đổi hành vi ngoài ý muốn ở lớp con. Kotlin buộc bạn phải gõ từ khoá `open` như một cam kết rằng lớp này đã được thiết kế an toàn để kế thừa.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: 4 tính chất là phân loại sách giáo trình đúc kết lại để dạy học, kế thừa sao chép memory layout.
- [ ] Bản chất Trừu tượng là che giấu chi tiết cài đặt; Đa hình là phân phối động tại runtime qua vtable.
- [ ] Phân tích bẫy Fragile Base Class và lý do Kotlin để class mặc định là final, hỗ trợ uỷ quyền bằng từ khoá by.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
