---
id: kotlin-scope-functions
title: Scope functions
domain: kotlin
tags: [kotlin, idioms, lambda, syntax]
collections: [interview-fundamental]
related: [kotlin-inline-function, kotlin-null-safety]
diagrams: [Cơ chế|scope-functions-matrix.png|Ma trận 2x2 phân loại scope functions theo đối tượng ngữ cảnh (this/it) và kết quả trả về, cùng hàm độc lập with]
created: 2026-09-07
---

## Bản 30 giây
Khi cấu hình một đối tượng, việc lặp lại tên biến khiến mã nguồn trở nên rườm rà. Nhóm năm hàm này tạm thời đưa đối tượng vào một khối làm việc cục bộ. Bạn có thể gọi trực tiếp hàm hoặc biến đổi dữ liệu mà không làm rò rỉ biến tạm ra ngoài.

## Cần biết trước
- kotlin-inline-function

## Vấn đề buộc nó ra đời
Trong Java cũ, khởi tạo đối tượng phức tạp đòi hỏi tạo biến tạm rồi lặp lại tên biến đó nhiều lần. Biến tạm này tiếp tục trôi nổi trong hàm dù không còn dùng tới, gây ô nhiễm phạm vi cục bộ.

Bên cạnh đó, việc biến đổi biến có thể rỗng đòi hỏi nhiều tầng `if (x != null)` lồng nhau. Lập trình viên khao khát một cách viết chuỗi liền mạch như các luồng dữ liệu hàm.

Năm 2016, JetBrains ra mắt Kotlin 1.0 cùng nhóm năm hàm scope functions trong thư viện chuẩn. Nhờ cơ chế [inline function](concept:kotlin-inline-function), chúng mang lại cú pháp ngắn gọn mà hoàn toàn không tốn chi phí cấp phát bộ nhớ lúc chạy.

## Nó thực sự là cái gì
Scope functions KHÔNG phải nhóm hàm rời rạc đặt tên tùy tiện. Thực chất, bốn [extension function](concept:kotlin-extension-and-object) tạo thành ma trận 2x2: gọi bằng `this` hay `it`, và trả về đối tượng hay kết quả lambda. Hàm thứ năm `with` đứng ngoài ma trận vì là hàm độc lập nhận đối tượng làm tham số.

## Cơ chế
Nhóm trả về kết quả lambda gồm `let` (dùng `it`) và `run` (dùng `this`). Nhóm trả về chính đối tượng ngữ cảnh gồm `also` (dùng `it`) và `apply` (dùng `this`). Hàm `with` nhận đối tượng làm tham số nên không thể dùng `?.` như `x?.run` khi gặp biến nullable. Cả năm hàm đều là [inline function](concept:kotlin-inline-function), compiler dán mã tại chỗ nên không cấp phát trên Heap.

## Đánh đổi
Mã nguồn trở nên cô đọng, liền mạch và dọn sạch các biến tạm thời. Đổi lại, lồng ghép nhiều tầng scope functions làm luồng code tối nghĩa và rất khó đặt breakpoint debug. Khi dùng `this` trong các khối lồng nhau, bạn rất dễ gọi nhầm hàm của khối cha bên ngoài.

## Nguyên lý gốc
Phép chiếu ngữ cảnh cục bộ (lexical scoping): thu hẹp phạm vi sống của dữ liệu vào đúng đoạn mã có nhu cầu sử dụng.

## Key insight
> Scope functions không sinh ra để nối chuỗi cho ngắn code, mà để thu hẹp phạm vi biến tạm tránh rò rỉ ngữ cảnh.

## Khuôn mẫu chuẩn
```kotlin
class UserConfig(var name: String = "", var age: Int = 0)

val config = UserConfig().apply {
    name = "Alice"
    age = 25
}.also {
    println("Created: ${it.name}")
}

val summary: String = config.let { "${it.name} - ${it.age}" }
```

## Bẫy & ranh giới
Bẫy tinh vi nhất là viết `x?.let { null } ?: runDefault()`. Khi lambda trả về null, Elvis `?:` kích hoạt ngay cả khi `x` khác null, phá hủy logic [Null safety](concept:kotlin-null-safety). Bẫy thứ hai là lồng `apply` trong `apply`, làm lu mờ con trỏ `this`. Chống chỉ định lồng các hàm dùng `this` quá hai tầng vì gây lu mờ ngữ cảnh và rất khó debug.

## Câu hỏi thực chiến
**Hỏi:** Dùng scope function như let hay apply có làm app chậm đi vì sinh thêm object không?
Không. Cả năm hàm đều mang từ khoá inline nên compiler sẽ chèn thẳng mã lệnh vào vị trí gọi lúc biên dịch. Bạn không tốn thêm bất kỳ đối tượng Function hay chi phí cấp phát bộ nhớ nào trên Heap.

**Hỏi:** Làm sao chọn nhanh giữa let, run, also và apply mà không cần tra bảng?
Tuỳ vào bạn cần đối tượng gốc hay kết quả biến đổi, và gọi qua `this` hay `it`. Cần chính đối tượng để cấu hình hoặc ghi log thì dùng `apply` hoặc `also`. Cần biến đổi sang dữ liệu khác hoặc kiểm tra null thì dùng `let` hoặc `run`.

**Hỏi:** Vì sao viết `user?.let { sendEmail(it) } ?: showLoginDialog()` lại có thể gây lỗi hiển thị sai?
Vì toán tử Elvis kiểm tra giá trị trả về của khối `let`, không phải kiểm tra biến gốc. Nếu biểu thức trong lambda trả về null, nhánh sau `?:` vẫn bị gọi dù `user` không hề null. Hãy thay bằng lệnh `if (user != null)` truyền thống khi nội dung xử lý có thể trả về null.

**Hỏi:** Khi nào nên dùng `also` thay vì `apply`?
Dùng `also` khi bạn thực hiện các tác vụ phụ trợ mà không can thiệp vào thuộc tính của đối tượng. Ví dụ điển hình là in log, ghi nhận sự kiện analytics hoặc kiểm tra ràng buộc dữ liệu. Vì `also` dùng `it`, bạn không sợ vô tình gọi nhầm thuộc tính của khối bao ngoài như `this`.

**Hỏi:** Hàm `with` và `run` khác nhau chỗ nào khi cùng nhận `this` và trả về kết quả?
`with` nhận đối tượng làm tham số truyền vào, còn `run` là extension function gọi trực tiếp trên đối tượng. Do đó `with` không thể kết hợp toán tử an toàn `?.` để tự động bỏ qua khi gặp biến null. Khi làm việc với biến nullable, bạn bắt buộc phải dùng `x?.run` thay vì bọc ngoài bằng `with(x)`.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: 4 extension function tạo ma trận 2x2 (this/it và đối tượng/lambda); with là hàm độc lập.
- [ ] Không cấp phát bộ nhớ trên Heap do toàn bộ năm hàm đều là inline.
- [ ] Bẫy let trả về null kích hoạt nhầm toán tử Elvis và bẫy lồng các khối this làm lu mờ biến.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
