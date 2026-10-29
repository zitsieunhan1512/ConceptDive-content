---
id: kotlin-sealed-vs-data-class
title: Sealed class vs data class
domain: kotlin
tags: [kotlin, type-system, oop, data-modeling]
collections: [interview-fundamental]
related: [kotlin-equals-hashcode, android-state-holders]
diagrams: [Cơ chế|sealed-hierarchy-exhaustive.png|Sealed interface đóng kín cây kế thừa giúp compiler kiểm tra vét cạn trong when; data class tự sinh mã theo giá trị]
created: 2026-09-07
---

## Bản 30 giây
Khi biểu diễn trạng thái màn hình, bạn cần giới hạn các trạng thái và cất dữ liệu bên trong. `sealed` đóng kín danh sách lớp con để compiler bắt bạn xử lý đủ mọi nhánh trong `when`. Còn `data class` tự sinh các hàm so sánh nội dung và sao chép để bạn không phải viết tay.

## Cần biết trước

## Vấn đề buộc nó ra đời
Trong Java cũ, biểu diễn các trạng thái của một màn hình rất vướng. Dùng `enum` thì mọi nhánh có chung một cấu trúc nên không cất dữ liệu riêng cho từng nhánh được. Còn dùng class kế thừa tự do thì khối `switch` phải có nhánh `default`, nên thêm trạng thái mới rất dễ quên xử lý.

Ngoài ra, mỗi lần tạo class chứa dữ liệu, bạn phải viết tay hàng chục dòng cho `equals`, `hashCode` và `toString`. Khi thêm một trường mới vào class mà quên sửa các hàm này, phép so sánh hai đối tượng sẽ chạy sai.

Năm 2011, JetBrains tách hai việc này thành hai từ khóa riêng trong Kotlin. Từ khóa `sealed` khóa danh sách lớp con lại, còn `data` tự sinh toàn bộ hàm so sánh theo giá trị. Mãi đến năm 2021, Java 17 mới có `record` và `sealed` class tương tự.

## Nó thực sự là cái gì
`sealed class` và `data class` không phải hai cách đối lập để bạn chọn một bỏ một. `sealed` dùng để khóa danh sách các lớp con, còn `data` dùng để tự sinh hàm so sánh theo dữ liệu bên trong.

## Cơ chế
Trước Kotlin 1.5, các lớp con của `sealed` phải nằm trong cùng một file. Từ Kotlin 1.5, bạn có thêm `sealed interface` và được đặt lớp con ở file khác trong cùng package và module. Nhờ biết hết mọi lớp con lúc biên dịch, compiler kiểm tra được khối `when` đã xét đủ nhánh chưa (exhaustive) mà không cần `else`. Còn từ khóa `data` dựa vào các biến ở constructor chính để tự sinh bytecode cho [equals và hashCode](concept:kotlin-equals-hashcode), `toString`, `componentN` và `copy`.

## Đánh đổi
Bạn không phải viết code lặp lại và được compiler nhắc ngay khi quên xử lý một nhánh trạng thái. Đổi lại, mỗi khi thêm một lớp con vào `sealed`, bạn phải đi cập nhật mọi khối `when` liên quan trong dự án. Với `data class`, nếu bạn dùng biến `var` rồi sửa giá trị, đối tượng sẽ bị thất lạc khi đang nằm trong `HashMap`.

## Nguyên lý gốc
Ghép kiểu chọn một trong nhiều nhánh (Sum Type) với kiểu gom nhiều trường dữ liệu lại (Product Type).

## Key insight
> Đặt `sealed` đối lập `data` là sai tiền đề: chúng là hai trục độc lập thường kết hợp để biểu diễn trạng thái.

## Khuôn mẫu chuẩn
```kotlin
sealed interface ScreenState {
    data object Loading : ScreenState
    data class Success(val items: List<String>) : ScreenState
    data class Error(val message: String) : ScreenState
}

fun render(state: ScreenState): String = when (state) {
    is ScreenState.Loading -> "Loading"
    is ScreenState.Success -> "Items: ${state.items.size}"
    is ScreenState.Error -> state.message
}
```

## Bẫy & ranh giới
Nếu bạn khai báo `var` trong `data class`, việc đổi giá trị biến đó sẽ làm đổi `hashCode` và làm mất đối tượng trong `HashMap`. Bẫy thứ hai là viết thêm nhánh `else` vào khối `when` đang kiểm tra một `sealed class`. Khi có người thêm trạng thái mới, nó sẽ lọt vào nhánh `else` khiến compiler không báo lỗi cho bạn biết. Đừng dùng `data class` cho các đối tượng có vòng đời hoặc định danh riêng mà không so sánh bằng giá trị.

## Câu hỏi thực chiến
**Hỏi:** Data class có kế thừa được sealed class hay sealed interface không?
Được. Hai từ khoá này làm hai việc khác nhau và thường đi cặp với nhau. Trong [State holders](concept:android-state-holders), từng nhánh trạng thái có mang dữ liệu thường là một `data class` cài đặt `sealed interface` chung.

**Hỏi:** Khi nào không nên dùng data class mà phải dùng class thông thường?
Khi đối tượng có định danh hoặc vòng đời riêng chứ không phải một gói dữ liệu thuần túy. Ví dụ như [ViewModel](concept:android-viewmodel) hay một kết nối mạng thì phải dùng class thường. Nếu dùng `data class`, hai đối tượng khác nhau có thể bị coi là một chỉ vì các biến bên trong tình cờ bằng nhau.

**Hỏi:** Vì sao trong biểu thức when với sealed class, thêm nhánh else lại bị coi là chống chỉ định?
Vì nhánh `else` làm tắt mất bước kiểm tra đủ nhánh của compiler. Sau này bạn thêm một trạng thái mới vào `sealed class`, các khối `when` có `else` vẫn biên dịch êm ru mà không báo thiếu. Lúc chạy thật, trạng thái mới đó sẽ âm thầm rơi vào nhánh `else` và gây lỗi khó tìm.

**Hỏi:** Nên dùng sealed class hay sealed interface khi mô hình hoá trạng thái UI?
Nên ưu tiên `sealed interface`, trừ khi các nhánh con cần dùng chung constructor hoặc biến có backing field. Dùng `sealed interface` thì lớp con vẫn còn nguyên quyền kế thừa một class khác và cài đặt được nhiều interface cùng lúc. Ngoài ra, interface không tạo constructor nên bytecode sinh ra cũng gọn hơn.

**Hỏi:** Data object khác data class chỗ nào khi dùng trong sealed hierarchy?
`data object` dùng cho nhánh trạng thái không cầm theo dữ liệu nào, ví dụ như `Loading`. Nó chỉ tạo một đối tượng duy nhất trong bộ nhớ (singleton) thay vì cấp phát mới mỗi lần chuyển trạng thái. Hơn nữa, `data object` tự sinh sẵn hàm `toString()` in ra đúng tên lớp rất dễ đọc khi gỡ lỗi.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao sealed và data không đối lập nhau mà thường kết hợp để biểu diễn trạng thái màn hình.
- [ ] Nêu được cách sealed khóa danh sách lớp con để compiler kiểm tra đủ nhánh (exhaustive) trong khối when.
- [ ] Trình bày được các hàm mà data class tự sinh từ constructor chính, cùng hai cái bẫy dùng biến var và nhánh else.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
