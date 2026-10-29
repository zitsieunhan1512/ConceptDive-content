---
id: oop-four-pillars
title: Bốn tính chất OOP
domain: oop
tags: [oop, paradigm, software-design]
collections: [interview-fundamental]
related: [oop-solid, jvm-virtual-method-dispatch, oop-abstract-class-vs-interface]
diagrams: [Cơ chế|four-pillars.png|Bốn trụ cột OOP: đóng gói giấu dữ liệu, trừu tượng chỉ lộ vài hàm, kế thừa nhận sẵn trường và hàm, đa hình chọn hàm lúc chạy]
created: 2026-09-07
---

## Bản 30 giây
Khi phần mềm lớn dần, việc phụ thuộc trực tiếp vào chi tiết của nhau sẽ gây đổ vỡ dây chuyền. Thay vì tự do truy cập, chương trình giao tiếp qua các thông điệp và giao diện trừu tượng được bảo vệ. Bốn tính chất là bộ khung giúp cô lập biến động và kiểm soát ranh giới phụ thuộc lúc chạy.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1967, Ole-Johan Dahl và Kristen Nygaard phát triển ngôn ngữ Simula 67 để mô phỏng các sự kiện rời rạc. Khi viết kiểu thủ tục với biến rời rạc, theo dõi trạng thái nhiều thực thể cùng lúc rất dễ rối. Hai ông gom luôn biến và hàm xử lý vào chung một khối gọi là đối tượng (object).

Khoảng năm 1966 đến 1967, Alan Kay đặt ra thuật ngữ lập trình hướng đối tượng tại Đại học Utah trước khi làm Smalltalk. Ông xem cốt lõi là đối tượng gửi thông điệp (message passing), giữ kín trạng thái và nối hàm muộn (late binding).

Đến thập niên 1990, C++ và Java bùng nổ trong các trường đại học. Sách giáo trình đúc kết lại thành bốn tính chất: đóng gói, trừu tượng, kế thừa và đa hình. Cách dạy này biến chúng thành bài học cú pháp, làm người học quên mất ý tưởng gửi thông điệp ban đầu của Kay.

## Nó thực sự là cái gì
Bốn tính chất OOP không phải là định nghĩa gốc của Alan Kay. Chúng là cách sách giáo trình thập niên 1990 gom nhóm lại để dạy các ngôn ngữ xoay quanh `class`. Thay vì xoay quanh việc gửi thông điệp, bốn tính chất này đặt ranh giới để code không phụ thuộc bừa vào nhau.

## Cơ chế
Đóng gói dùng từ khóa `private` khóa biến lại bên trong, còn trừu tượng chỉ mở ra vài hàm cần thiết cho bên ngoài gọi. Kế thừa chép cấu trúc ô nhớ (memory layout) từ lớp cha, khác với hợp đồng trong [Abstract class đối lập Interface](concept:oop-abstract-class-vs-interface). Đa hình dựa vào [bảng phương thức ảo (vtable)](concept:jvm-virtual-method-dispatch) để chọn đúng hàm cần chạy lúc ứng dụng đang mở. Nhờ vậy, nơi gọi hàm không bị khóa cứng vào một lớp cụ thể nào.

## Đánh đổi
Code được chia thành từng khối rõ ràng và biến bên trong không bị sửa bậy. Đổi lại, bạn phải đọc qua nhiều lớp trung gian hơn để lần ra nơi chạy thật. Máy ảo cũng tốn thêm bộ nhớ Heap cho đối tượng và tốn một bước tra bảng `vtable` khi gọi hàm. Nếu lạm dụng kế thừa thay vì lắp ghép (composition), lớp con sẽ dính chặt lớp cha và hỏng tính đóng gói.

## Nguyên lý gốc
Nguyên lý giấu thông tin của David Parnas năm 1972: mỗi khối tự giấu cách làm và chỉ mở cửa giao tiếp nhỏ nhất.

## Key insight
> OOP không nhằm tái sử dụng mã qua kế thừa, mà để cô lập biến động bằng ranh giới đóng gói và đa hình.

## Ví von
Với chiếc ô tô, đóng gói là giấu động cơ dưới nắp capo, còn trừu tượng là chỉ lộ vô lăng và phanh. Kế thừa là xe điện và xe xăng cùng thuộc dòng ô tô. Đa hình là bạn cứ đạp phanh thì xe nào cũng dừng, dù bên dưới dùng má phanh cơ hay phanh điện tái sinh.

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
Nếu coi kế thừa là trung tâm thay vì gửi thông điệp, bạn dễ dính bẫy lớp cha dễ vỡ (Fragile Base Class). Khi ghi đè hàm thêm danh sách, lớp cha lại tự gọi hàm thêm từng phần tử làm kết quả đếm nhân đôi. Để chặn bẫy này, Kotlin khóa mọi `class` mặc định là `final` và cho phép ủy quyền qua từ khóa `by`. Đừng kế thừa chỉ để lấy lại vài dòng code; hãy dùng lắp ghép (composition) để giữ an toàn cho lớp.

## Câu hỏi thực chiến
**Hỏi:** Mục đích chính của OOP có phải là để tái sử dụng code thông qua kế thừa không?
Không phải. Mục đích chính của OOP là giữ cho khi bạn sửa một chỗ thì không làm hỏng dây chuyền sang chỗ khác. Nếu lạm dụng kế thừa chỉ để dùng lại code, lớp con sẽ dính chặt vào ruột lớp cha và làm hỏng tính đóng gói.

**Hỏi:** Khi nào nên dùng kế thừa, khi nào nên dùng lắp ghép (composition)?
Bạn nên ưu tiên lắp ghép, và chỉ kế thừa khi lớp con thật sự là một loại của lớp cha (IS-A). Nếu chỉ cần dùng vài hàm của lớp khác, bạn hãy đặt nó làm một biến bên trong lớp mình. Trong Kotlin, bạn dùng thêm từ khóa `by` để chuyển tiếp hàm của `interface` mà không bị trói vào lớp cha.

**Hỏi:** Trừu tượng (Abstraction) khác Đóng gói (Encapsulation) chỗ nào khi cả hai đều giấu thông tin?
Đóng gói là khóa biến dữ liệu lại, còn trừu tượng là giấu đi các bước xử lý rắc rối bên dưới. Bạn đóng gói bằng từ khóa `private` để bên ngoài không tự ý gán sai giá trị biến. Còn trừu tượng là bạn chỉ đưa ra một hàm gọn gàng để nơi khác gọi mà không cần biết thuật toán bên trong.

**Hỏi:** Đa hình (Polymorphism) vận hành ở tầng máy ảo như thế nào để biết phải gọi hàm nào?
Lúc chạy, máy ảo tra địa chỉ hàm trong [bảng phương thức ảo (vtable)](concept:jvm-virtual-method-dispatch) gắn với đối tượng thực tế. Mỗi đối tượng trên Heap đều giữ một con trỏ chỉ tới thông tin lớp và bảng `vtable` của lớp đó. Khi gặp lệnh gọi hàm, CPU nhìn vào bảng này để nhảy tới đúng hàm mà lớp con đã ghi đè.

**Hỏi:** Vì sao Kotlin lại mặc định khoá mọi class bằng final thay vì mở như Java?
Để tránh lỗi lớp cha dễ vỡ (Fragile Base Class) do thói quen kế thừa tùy tiện trong Java. Khi lớp cha chưa chuẩn bị cho kế thừa, sửa một hàm nhỏ ở cha cũng dễ làm lớp con chạy sai. Vì vậy Kotlin bắt gõ rõ từ khóa `open` để xác nhận lớp đã được thiết kế an toàn cho kế thừa.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được 4 tính chất là cách sách giáo trình đúc kết để dạy học, và kế thừa chép memory layout của lớp cha sang lớp con.
- [ ] Phân biệt được đóng gói là khóa biến bằng private, trừu tượng là giấu cách cài đặt, còn đa hình tra bảng vtable lúc chạy.
- [ ] Nêu được bẫy Fragile Base Class và lý do Kotlin để class mặc định là final kèm từ khóa ủy quyền by.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
