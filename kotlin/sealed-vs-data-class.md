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
Sealed class và data class giải quyết hai bài toán hoàn toàn khác nhau. Lớp sealed giới hạn danh sách con để compiler bắt bạn xử lý đủ mọi nhánh. Lớp data tự động sinh hàm so sánh giá trị và sao chép để bạn không phải viết tay.

## Cần biết trước

## Vấn đề buộc nó ra đời
Trong Java cũ, mô hình hóa trạng thái màn hình gặp bế tắc lớn. Dùng Enum thì mọi nhánh bị ép chung một cấu trúc dữ liệu, không thể lưu tham số riêng. Mở lớp kế thừa tự do thì lệnh switch buộc phải có nhánh default, dễ bỏ sót lỗi khi thêm class mới.

Bên cạnh đó, việc tạo class chứa dữ liệu đòi hỏi viết tay hàng trăm dòng mã cho `equals`, `hashCode` và `toString`. Khi thêm thuộc tính mới vào class, lập trình viên thường quên cập nhật các hàm này. Hậu quả là phép so sánh hai đối tượng bị sai lệch nghiêm trọng.

Năm 2011, JetBrains thiết kế Kotlin và tách hai bài toán thành hai từ khóa độc lập. Từ khóa `sealed` khóa chặt phân cấp kế thừa, còn `data` tự động hóa toàn bộ hàm so sánh dữ liệu. Mãi đến năm 2021, Java 17 mới bắt kịp mô hình này qua `record` và `sealed` class.

## Nó thực sự là cái gì
Sealed class và data class KHÔNG phải hai phương án kiến trúc đối đầu nhau. Thực chất, sealed là công cụ kiểm soát cây kế thừa, còn data là công cụ tự sinh mã so sánh theo giá trị.

## Cơ chế
Trước Kotlin 1.5, lớp con của `sealed` bắt buộc phải nằm cùng file. Từ bản 1.5, Kotlin bổ sung `sealed interface` và cho phép lớp con nằm khác file, miễn là cùng package và module. Nhờ biết trước mọi lớp con, compiler kiểm tra được tính vét cạn (exhaustive) trong `when` mà không cần `else`. Từ khóa `data` tự sinh bytecode cho [equals và hashCode](concept:kotlin-equals-hashcode), `toString`, `componentN` và `copy` từ constructor chính.

## Đánh đổi
Lập trình viên loại bỏ được mã lặp và có compiler bảo vệ khỏi việc bỏ sót nhánh. Đổi lại, thêm một nhánh vào sealed class sẽ buộc sửa mọi khối `when` trong dự án. Với data class, khai báo thuộc tính biến đổi (`var`) sẽ phá hủy tính toàn vẹn của bảng băm.

## Nguyên lý gốc
Hệ thống kiểu đại số: kiểu tổng (Sum Type - chọn một nhánh) và kiểu tích (Product Type - gom các trường dữ liệu).

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
Bẫy chết người là dùng `var` trong data class, khiến `hashCode` bị đổi sau khi sửa và biến mất khỏi HashMap. Bẫy thứ hai là lạm dụng nhánh `else` trong biểu thức `when` của sealed class. Khi thêm trạng thái mới, `else` nuốt chửng nhánh đó và tái hiện đúng lỗi bỏ sót của Java. Chống chỉ định dùng data class cho thực thể có ID riêng thay vì đối tượng so sánh bằng giá trị.

## Câu hỏi thực chiến
**Hỏi:** Data class có kế thừa được sealed class hay sealed interface không?
Được. Hai từ khoá này nằm trên hai trục độc lập và rất thường xuyên kết hợp cùng nhau. Trong [State holders](concept:android-state-holders), các nhánh dữ liệu thường là data class cài đặt `sealed interface` để compiler kiểm tra tính vét cạn.

**Hỏi:** Khi nào không nên dùng data class mà phải dùng class thông thường?
Khi đối tượng cần định danh riêng thay vì so sánh bằng dữ liệu bên trong. Ví dụ đối tượng có vòng đời hoặc trạng thái đổi liên tục như [ViewModel](concept:android-viewmodel) hay một kết nối mạng. So sánh bằng dữ liệu sẽ coi hai thực thể khác nhau là một khi dữ liệu tình cờ trùng.

**Hỏi:** Vì sao trong biểu thức when với sealed class, thêm nhánh else lại bị coi là chống chỉ định?
Vì nhánh `else` làm vô hiệu hoá khả năng kiểm tra vét cạn của trình biên dịch. Khi bạn thêm một nhánh mới vào sealed class, compiler không thể báo lỗi ở các khối `when` có `else`. Ứng dụng sẽ âm thầm rơi vào nhánh mặc định, tái hiện lại đúng nguy cơ bỏ sót lỗi như trong Java.

**Hỏi:** Nên dùng sealed class hay sealed interface khi mô hình hoá trạng thái UI?
Ưu tiên `sealed interface` trừ khi bạn cần chia sẻ constructor hoặc thuộc tính có backing field. `sealed interface` giúp lớp con không bị mất suất đơn kế thừa và có thể cài đặt nhiều giao diện cùng lúc. Ngoài ra, giao diện không sở hữu constructor nên cấu trúc bytecode nhẹ nhàng và linh hoạt hơn.

**Hỏi:** Data object khác data class chỗ nào khi dùng trong sealed hierarchy?
`data object` dùng cho các nhánh trạng thái không mang dữ liệu riêng như `Loading`. Nó tạo một singleton duy nhất nên không tốn chi phí cấp phát bộ nhớ mỗi lần phát sinh trạng thái. Đồng thời, `data object` tự sinh hàm `toString()` hiển thị rõ tên lớp thay vì chuỗi địa chỉ khó đọc.

## Ý bắt buộc khi giải thích lại
- [ ] Khẳng định câu hỏi so sánh sealed class với data class là sai tiền đề vì hai khái niệm độc lập.
- [ ] Sealed class giúp compiler kiểm tra tính vét cạn (exhaustive when) bằng cách giới hạn phạm vi kế thừa.
- [ ] Data class tự sinh các hàm so sánh nội dung và sao chép theo giá trị từ constructor chính.
- [ ] Bẫy dùng var trong data class phá vỡ bảng băm và bẫy dùng nhánh else làm mất tính vét cạn.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
