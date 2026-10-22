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
Hàm mở rộng giúp bổ sung tính năng cho một lớp có sẵn mà không cần sửa mã nguồn hoặc kế thừa. Bản chất bên dưới chỉ là hàm tĩnh thông thường nhận đối tượng đó làm tham số đầu vào. Cùng với đó, từ khoá `object` giúp tạo ra đối tượng duy nhất một cách an toàn mà không cần viết mẫu đơn bản.

## Cần biết trước
- jvm-virtual-method-dispatch
- oop-four-pillars

## Vấn đề buộc nó ra đời
Trong các dự án Java, lập trình viên thường tạo hàng loạt lớp tiện ích tĩnh như `StringUtils`. Cách viết này làm mất tính liền mạch của phong cách lập trình hướng đối tượng. Lập trình viên luôn phải gọi hàm lồng nhau thay vì chấm trực tiếp từ đối tượng.

Bên cạnh đó, việc tạo đối tượng đơn bản trong Java rất rườm rà và dễ mắc lỗi. Lập trình viên phải tự viết cơ chế khoá hai lần để bảo đảm an toàn đa luồng. Một sơ suất nhỏ trong xử lý luồng cũng làm hỏng tính duy nhất của đối tượng.

Kotlin được JetBrains công bố năm 2011 và phát hành bản 1.0 năm 2016, hỗ trợ sẵn hàm mở rộng và `object`. Thiết kế này biến các mẫu mã rườm rà thành cú pháp trực quan và an toàn.

## Nó thực sự là cái gì
Hàm mở rộng KHÔNG hề chèn thêm mã lệnh vào bên trong lớp được mở rộng.

Thực chất, compiler biến nó thành phương thức tĩnh trong lớp `<TênFile>Kt`, nhận receiver làm tham số đầu tiên.

Từ khoá `object` cũng KHÔNG phải là các phần tử tĩnh đơn thuần của Java.

Thực chất, `object` là lớp riêng có đúng một thể hiện, khởi tạo khi truy cập lần đầu và an toàn luồng nhờ JLS §12.4.2.

## Cơ chế
Hàm mở rộng được phân giải tĩnh khi biên dịch dựa theo kiểu khai báo của biến.

Vì không sử dụng [bảng hàm ảo](concept:jvm-virtual-method-dispatch), bạn không thể ghi đè hàm mở rộng.

Nếu hàm mở rộng trùng tên với hàm nội tại, compiler ưu tiên hàm nội tại và cảnh báo bị che khuất.

Khi dùng `companion object`, cần gắn nhãn `@JvmStatic` để Java gọi trực tiếp như hàm tĩnh.

## Đánh đổi
Hàm mở rộng không thể truy cập vào các thuộc tính riêng tư của lớp ban đầu.

Bù lại, lập trình viên được lợi cú pháp gọi gọn và tự gợi ý; bytecode bên dưới vẫn là lớp tiện ích tĩnh.

## Nguyên lý gốc
Nguyên lý cú pháp thuận tiện: đóng gói mẫu mã quen thuộc thành cú pháp tự nhiên mà không đổi bản chất thực thi.

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
Nếu định nghĩa hàm mở rộng trùng chữ ký với thành viên lớp, compiler cảnh báo bị che khuất và ưu tiên thành viên. Việc dùng `object` bừa bãi sẽ tạo ra trạng thái toàn cục khó kiểm thử và cản trở việc dọn dẹp bộ nhớ. Khi gọi từ Java, thiếu nhãn `@JvmStatic` sẽ gây lỗi biên dịch nếu bạn không chấm qua thuộc tính Companion.

## Câu hỏi thực chiến
**Hỏi:** Nếu khai báo một extension function trùng tên và kiểu tham số với member function có sẵn của class thì chuyện gì xảy ra?
Hàm thành viên nội tại sẽ được gọi. Trình biên dịch Kotlin sẽ hiển thị cảnh báo hàm mở rộng bị che khuất bởi thành viên. Hàm mở rộng của bạn sẽ không bao giờ được thực thi khi chữ ký trùng khớp.

**Hỏi:** Từ khoá object trong Kotlin có phải tương đương hoàn toàn với từ khoá static trong Java không?
Không tương đương về mặt kiến trúc bộ nhớ. Bản chất `object` là một lớp riêng biệt nắm giữ cá thể đơn bản duy nhất được khởi tạo lười. Nó là một đối tượng thực tế trên bộ nhớ chứ không phải chỉ là trường tĩnh độc lập.

**Hỏi:** Tại sao gọi extension function trên một biến có kiểu giao diện lại không áp dụng đa hình lúc chạy?
Vì hàm mở rộng được phân giải tĩnh tại thời điểm biên dịch. Máy ảo không tra cứu [bảng hàm ảo](concept:jvm-virtual-method-dispatch) để tìm đối tượng thực tế. Hàm được gọi hoàn toàn phụ thuộc vào kiểu dữ liệu bạn khai báo trên mã nguồn.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất bytecode của hàm mở rộng là hàm tĩnh nhận receiver làm tham số đầu tiên.
- [ ] Cơ chế phân giải tĩnh và thứ tự ưu tiên của hàm thành viên trước hàm mở rộng.
- [ ] Cách thức khởi tạo an toàn luồng của từ khoá object và vai trò của @JvmStatic.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

