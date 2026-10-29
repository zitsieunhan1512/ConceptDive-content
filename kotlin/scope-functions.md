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
Vừa tạo `Dialog` hay `Intent` rồi gán 4 thuộc tính liên tiếp, bạn phải lặp lại tên biến ở mọi dòng. Năm hàm scope functions mở ra một khối lệnh nhỏ ngay trên đối tượng đó để bạn gán thuộc tính hoặc đổi kiểu dữ liệu. Nhờ vậy, biến tạm không bị tràn ra phần còn lại của hàm.

## Cần biết trước
- kotlin-inline-function

## Vấn đề buộc nó ra đời
Trong Java cũ, muốn tạo và cài đặt đối tượng nhiều bước, bạn phải tạo biến tạm rồi viết lại tên biến ở từng dòng. Biến tạm này vẫn nằm lại trong hàm sau khi dùng xong, dễ bị các dòng bên dưới dùng nhầm.

Khi làm việc với biến có thể null, bạn lại phải viết nhiều tầng `if (x != null)` lồng nhau. Lập trình viên cần một cách nối các bước xử lý liền mạch mà không phải đặt thêm tên biến phụ.

Kotlin 1.0 (2016) đưa sẵn `let`, `run`, `with`, `apply` vào thư viện chuẩn, rồi Kotlin 1.1 (2017) bổ sung thêm `also`. Nhờ là [inline function](concept:kotlin-inline-function), cả năm hàm giúp code gọn lại mà không tạo thêm đối tượng lúc chạy.

## Nó thực sự là cái gì
Năm hàm này không phải được đặt tên ngẫu nhiên, mà bốn [extension function](concept:kotlin-extension-and-object) ghép thành một bảng 2x2 theo hai câu hỏi. Câu hỏi một là bên trong ngoặc nhọn bạn gọi đối tượng bằng `this` hay `it`. Câu hỏi hai là chạy xong ngoặc nhọn thì hàm trả về chính đối tượng đó hay trả về kết quả dòng cuối. Riêng hàm thứ năm `with` nằm ngoài bảng vì nhận đối tượng làm tham số thường.

## Cơ chế
Hai hàm trả về kết quả dòng cuối là `let` (gọi bằng `it`) và `run` (gọi bằng `this`). Hai hàm trả về lại chính đối tượng ban đầu là `also` (gọi bằng `it`) và `apply` (gọi bằng `this`). Còn `with` nhận đối tượng qua ngoặc tròn `with(x)` nên không ghép được với dấu `?.` như `x?.run` khi biến có thể null. Vì cả năm đều là [inline function](concept:kotlin-inline-function), compiler chép thẳng thân hàm vào chỗ gọi nên không tốn thêm bộ nhớ Heap.

## Đánh đổi
Code của bạn gọn lại, đọc một mạch từ trên xuống và không còn biến tạm thừa. Đổi lại, nếu bạn nối hoặc lồng 3 khối scope functions liên tiếp, người đọc sẽ rối và rất khó đặt breakpoint gỡ lỗi. Nhất là khi lồng hai khối cùng dùng `this`, bạn rất dễ gọi nhầm thuộc tính của khối bên ngoài.

## Nguyên lý gốc
Chỉ mở biến ra trong đúng vài dòng cần dùng nó, xong việc là đóng lại ngay (lexical scoping).

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
Bẫy hay gặp nhất là viết `x?.let { work() } ?: runFallback()` khi hàm `work()` có thể trả về null. Lúc đó dù `x` khác null, toán tử Elvis `?:` vẫn chạy nhánh dự phòng và làm hỏng ý đồ [Null safety](concept:kotlin-null-safety) của bạn. Đừng lồng các khối dùng `this` như `apply` hay `run` quá hai tầng vì sẽ không biết `this` đang chỉ vào đâu. Triệu chứng là hàm gán nhầm biến của lớp ngoài, hoặc chạy cả nhánh đúng lẫn nhánh báo lỗi trong một lần bấm.

## Câu hỏi thực chiến
**Hỏi:** Dùng scope function như let hay apply có làm app chậm đi vì sinh thêm object không?
Không. Cả năm hàm đều được đánh dấu `inline`, nên lúc biên dịch compiler chép thẳng các dòng bên trong ra ngay chỗ gọi. Ứng dụng không tốn thêm đối tượng lambda nào trên Heap.

**Hỏi:** Làm sao chọn nhanh giữa let, run, also và apply mà không cần tra bảng?
Hãy tự hỏi mình cần nhận lại đối tượng cũ hay kết quả mới, và muốn viết `this` hay `it`. Muốn cài đặt thuộc tính hoặc in log rồi giữ nguyên đối tượng thì chọn `apply` hoặc `also`. Muốn đổi sang giá trị khác hoặc kiểm tra khác null bằng `?.` thì chọn `let` hoặc `run`.

**Hỏi:** Vì sao viết `user?.let { sendEmail(it) } ?: showLoginDialog()` lại có thể gây lỗi hiển thị sai?
Vì dấu `?:` nhìn vào kết quả trả về của cả khối `let`, chứ không chỉ nhìn vào biến `user`. Nếu `sendEmail(it)` trả về null, hộp thoại đăng nhập vẫn bật lên dù người dùng đã đăng nhập rồi. Khi hàm bên trong có thể trả về null, bạn hãy viết `if (user != null)` bình thường cho rõ nghĩa.

**Hỏi:** Khi nào nên dùng `also` thay vì `apply`?
Dùng `also` khi bạn chỉ làm việc phụ như in log hay bắn sự kiện thống kê mà không sửa thuộc tính của đối tượng. Vì `also` gọi đối tượng bằng tên `it`, bạn đọc vào thấy rõ ngay mình đang truyền cả đối tượng đi. Nó cũng tránh được việc gọi nhầm biến `this` của lớp bên ngoài.

**Hỏi:** Hàm `with` và `run` khác nhau chỗ nào khi cùng nhận `this` và trả về kết quả?
`with` nhận đối tượng làm tham số trong ngoặc `with(user)`, còn `run` gọi trực tiếp sau dấu chấm `user.run`. Vì vậy khi biến có thể null, bạn viết `user?.run` rất gọn, còn dùng `with` thì không gắn dấu `?.` vào được.

## Ý bắt buộc khi giải thích lại
- [ ] Vẽ lại được bảng 2x2 của 4 hàm let, run, also, apply theo hai trục this/it và kết quả trả về, kèm hàm with đứng riêng.
- [ ] Giải thích được vì sao dùng 5 hàm này không tạo thêm đối tượng lambda trên Heap nhờ từ khóa inline.
- [ ] Chỉ ra được bẫy x?.let trả về null làm chạy nhầm nhánh Elvis và bẫy lồng hai khối dùng this.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
