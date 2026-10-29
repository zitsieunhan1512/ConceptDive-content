---
id: kotlin-extension-and-object
title: Extension Function & Object Declaration
domain: kotlin
tags: [kotlin, language, oop, bytecode]
collections: [interview-fundamental]
related: [jvm-virtual-method-dispatch, oop-four-pillars, kotlin-val-vs-const]
diagrams: [Cơ chế|extension-static-dispatch.png|Hàm mở rộng được phân giải tĩnh theo kiểu khai báo biến tại thời điểm biên dịch, không qua bảng hàm ảo vtable]
created: 2026-10-01
---

## Bản 30 giây
Khi muốn gắn thêm hàm `toSlug()` cho lớp `String` có sẵn mà không sửa được mã nguồn của nó, bạn viết một `extension function`. Khi biên dịch, Kotlin chỉ biến nó thành một hàm tĩnh bình thường nhận chuỗi đó làm tham số đầu tiên. Từ khóa `object` giúp tạo ngay một đối tượng duy nhất (`Singleton`) mà khỏi tự viết khóa đa luồng.

## Cần biết trước
- jvm-virtual-method-dispatch
- oop-four-pillars

## Vấn đề buộc nó ra đời
Trong dự án Java, mỗi khi cần thêm hàm xử lý chuỗi hay ngày tháng, bạn phải tạo lớp tiện ích như `StringUtils`. Khi ghép nhiều bước xử lý, bạn phải viết các lời gọi hàm lồng ngược vào nhau thay vì chấm nối tiếp trên đối tượng.

Bên cạnh đó, muốn tạo một đối tượng dùng chung duy nhất (`Singleton`) trong Java vừa dài dòng vừa dễ sai. Bạn phải tự viết bước kiểm tra khóa hai lần để hai luồng chạy cùng lúc không tạo ra hai đối tượng khác nhau.

JetBrains giới thiệu Kotlin năm 2011 và ra mắt bản 1.0 năm 2016 với sẵn `extension function` và từ khóa `object`. Hai tính năng này giúp bạn chấm gọi hàm tự nhiên và tạo đối tượng duy nhất chỉ bằng một dòng khai báo.

## Nó thực sự là cái gì
`Extension function` không hề mở khóa hay nhét thêm hàm mới vào bên trong lớp gốc.

Thật ra, trình biên dịch tạo một hàm `static` trong lớp `<TênFile>Kt`, nhận đối tượng trước dấu chấm làm tham số đầu tiên.

Từ khóa `object` cũng không phải biến `static` rời rạc: trình biên dịch tạo lớp giữ biến `INSTANCE`, khởi tạo ở lần gọi đầu. Nó an toàn đa luồng nhờ chuẩn nạp lớp JVM (JLS §12.4.2).

## Cơ chế
Khi bạn gọi `extension function`, trình biên dịch chọn luôn hàm sẽ chạy dựa theo kiểu khai báo của biến ngay lúc dịch code.

Vì lệnh gọi không đi qua [bảng hàm ảo](concept:jvm-virtual-method-dispatch) `vtable`, lớp con không thể ghi đè (`override`) hàm mở rộng của lớp cha.

Nếu hàm mở rộng trùng tên và tham số với hàm có sẵn trong lớp, trình biên dịch luôn gọi hàm của lớp.

Còn khi viết hàm trong `companion object`, bạn gắn thêm `@JvmStatic` nếu muốn phía Java gọi thẳng được như hàm `static` thường.

## Đánh đổi
Vì bản chất chỉ là hàm tĩnh đứng bên ngoài, `extension function` không đọc được các biến `private` hay `protected` của lớp gốc.

Đổi lại, khi gõ dấu chấm, IDE tự gợi ý hàm ngay giúp code gọn và dễ đọc hơn lớp `Util`.

## Nguyên lý gốc
Giữ nguyên cách máy ảo chạy hàm tĩnh bên dưới, chỉ bọc lại bằng cách viết dấu chấm cho người đọc dễ hiểu (syntactic sugar).

## Key insight
> Hàm mở rộng được phân giải theo kiểu khai báo lúc biên dịch, không phải kiểu đối tượng lúc chạy.

## Khuôn mẫu chuẩn
```kotlin
fun String.toSlug(): String = lowercase().replace(" ", "-")
class NetworkClient {
    companion object {
        @JvmStatic
        fun create(): NetworkClient = NetworkClient()
    }
}
object AppDatabase {
    val version = 1
}
```

## Bẫy & ranh giới
Nếu thư viện thêm hàm thật trùng tên với `extension function` của bạn, code sẽ âm thầm gọi hàm của thư viện. Lạm dụng `object` giữ dữ liệu tạo ra biến toàn cục sống suốt đời ứng dụng, gây khó test và giữ bộ nhớ. Khi gọi hàm trong `companion object` từ code Java mà quên gắn `@JvmStatic`, bạn buộc phải viết dài dòng qua `.Companion` mới biên dịch được.

## Câu hỏi thực chiến
**Hỏi:** Nếu khai báo một extension function trùng tên và kiểu tham số với member function có sẵn của class thì chuyện gì xảy ra?
Hàm có sẵn bên trong lớp (`member function`) sẽ luôn thắng và được gọi. Trình biên dịch Kotlin chỉ hiện cảnh báo vàng báo rằng hàm mở rộng của bạn đã bị hàm thật che mất. Hàm mở rộng đó sẽ không bao giờ chạy chừng nào hai bên còn trùng hệt tên và danh sách tham số.

**Hỏi:** Từ khoá object trong Kotlin có phải tương đương hoàn toàn với từ khoá static trong Java không?
Không giống nhau về cách tổ chức trong bộ nhớ. Từ khóa `object` tạo một lớp thật và giữ đối tượng duy nhất trên Heap, khởi tạo khi chạm tới lần đầu. Là đối tượng thật, `object` kế thừa được lớp khác hoặc cài đặt `interface`, điều mà hàm `static` trong Java không làm được.

**Hỏi:** Tại sao gọi extension function trên một biến có kiểu giao diện lại không áp dụng đa hình lúc chạy?
Vì `extension function` được dịch thành lệnh gọi hàm tĩnh `invokestatic` ngay từ lúc biên dịch. Máy ảo không hề tra [bảng hàm ảo](concept:jvm-virtual-method-dispatch) để xem đối tượng thật lúc chạy thuộc lớp con nào. Bạn khai báo biến thuộc kiểu gì trên dòng code thì trình biên dịch khóa cứng vào hàm mở rộng của đúng kiểu đó.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được extension function khi dịch ra bytecode chỉ là hàm static nhận đối tượng gốc làm tham số đầu tiên.
- [ ] Trình bày được vì sao hàm mở rộng chọn theo kiểu khai báo lúc biên dịch và luôn thua hàm có sẵn trong lớp khi trùng tên.
- [ ] Nêu được cách object tạo đối tượng duy nhất an toàn đa luồng khi nạp lớp và tác dụng của nhãn @JvmStatic.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

