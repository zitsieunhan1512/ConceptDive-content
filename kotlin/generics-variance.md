---
id: kotlin-generics-variance
title: Generics & Variance (thử cập nhật)
domain: kotlin
tags: [kotlin, generics, variance, type-erasure, reified]
collections: [interview-fundamental]
related: [kotlin-inline-function, kotlin-sealed-vs-data-class, oop-four-pillars]
diagrams: [Cơ chế|generics-variance-directions.png|Chiều quan hệ kế thừa trong Generics: out giữ nguyên chiều (hiệp biến), in đảo ngược chiều (nghịch biến)]
created: 2026-10-01
---

## Bản 30 giây
Khi làm việc với kiểu generic, lập trình viên thường nhầm rằng danh sách kiểu con có thể thay thế danh sách kiểu cha. Để giữ an toàn kiểu và ngăn lỗi nhét nhầm phần tử, hệ thống phân định rõ chiều truyền dữ liệu. Từ khóa `out` cho phép coi `Producer<Dog>` như `Producer<Animal>`, còn `in` cho phép coi `Consumer<Animal>` như `Consumer<Dog>`.

## Cần biết trước
- oop-four-pillars
- kotlin-inline-function

## Vấn đề buộc nó ra đời
Năm 2004, Java 5 bổ sung generics nhưng giữ tương thích nhị phân với máy ảo cũ bằng cơ chế xóa kiểu (*type erasure*). Mọi thông tin kiểu đều bị xóa ở bytecode và thay bằng cận trên (*upper bound*), hoặc `Object` khi không có cận.

Mảng trong Java vốn có tính hiệp biến tự nhiên nhưng kiểm tra kiểu ở thời gian chạy. Điều này dẫn tới ngoại lệ `ArrayStoreException` khi nhét nhầm đối tượng khác kiểu vào mảng.

Để tránh lỗi này trên tập hợp, Java áp dụng use-site variance với ký hiệu wildcard phức tạp như `? extends` và `? super`. Kotlin chọn khai báo variance ngay tại nơi định nghĩa lớp để bỏ wildcard ở mỗi chỗ dùng.

## Nó thực sự là cái gì
Variance KHÔNG PHẢI là tính năng ép kiểu đối tượng trong bộ nhớ. Thực chất, đây là các quy tắc ở pha biên dịch nhằm kiểm soát quan hệ kế thừa giữa các kiểu phức hợp.

Danh sách `Dog` không là con của danh sách `Animal` vì việc thêm `Cat` vào sẽ phá vỡ an toàn kiểu.

Kotlin dùng `out` cho nhà sản xuất dữ liệu chỉ đọc, và `in` cho nơi tiêu thụ dữ liệu chỉ ghi.

## Cơ chế
Ở tầng bytecode, máy ảo Java xóa bỏ hoàn toàn tham số kiểu và thay bằng kiểu cận trên (*upper bound*) hoặc `Object`. Trình biên dịch chèn các lệnh ép kiểu `checkcast` tự động tại các vị trí đọc dữ liệu ra.

Với tham số `out`, compiler cấm kiểu đó xuất hiện ở vị trí tham số truyền vào của hàm thành viên.

Ngược lại, tham số `in` chỉ được xuất hiện ở vị trí nhận dữ liệu vào và bị cấm ở kiểu trả về.

## Đánh đổi
Khai báo variance giúp tái sử dụng hàm linh hoạt mà trình biên dịch vẫn chặn lỗi nhét sai kiểu. Đổi lại, việc xóa kiểu khiến chương trình không thể kiểm tra `is T` hay tạo thể hiện `T()` khi chạy.

Muốn giữ lại kiểu, bạn phải dùng từ khóa `reified` trong [Inline function](concept:kotlin-inline-function), đánh đổi bằng kích thước mã nhị phân.

## Nguyên lý gốc
Nguyên lý Liskov trong kiểu phức hợp: cấu trúc chỉ an toàn kế thừa khi quyền đọc và quyền ghi được phân tách rạch ròi. Bên cung cấp bảo toàn kiểu hướng lên, bên tiếp nhận bảo toàn kiểu hướng xuống.

## Key insight
> Variance không đổi đối tượng runtime, mà là luật kiểm soát chiều đọc ghi của compiler để bảo vệ kiểu generic.

## Ví von
Một chiếc hộp đựng cam `out` chỉ cho phép nhặt cam ra xem, nên người xem coi nó như hộp hoa quả vẫn an toàn. Chiếc máy ép hoa quả `in` có thể ép mọi loại quả, nên đưa cam vào máy ép hoa quả vẫn hoạt động chính xác.

## Khuôn mẫu chuẩn
```kotlin
interface Producer<out T> { fun produce(): T }
interface Consumer<in T> { fun consume(item: T) }
class Box<T>(var value: T) : Producer<T>, Consumer<T> {
    override fun produce(): T = value
    override fun consume(item: T) { value = item }
}
val p: Producer<Any> = Box<String>("text")
val c: Consumer<String> = Box<Any>(123)
```

## Bẫy & ranh giới
Bẫy ép kiểu không an toàn `as List` sau khi xóa kiểu sẽ không báo lỗi ngay, nhưng gây crash `ClassCastException` khi đọc phần tử. Chống chỉ định dùng `out` cho các cấu trúc dữ liệu vừa đọc vừa ghi như bộ nhớ đệm hay danh sách khả biến. Tránh lạm dụng chiếu sao `List<*>` vì làm mất khả năng ghi và chỉ đọc ra được kiểu `Any?`. Triệu chứng hỏng hóc là lỗi biên dịch từ chối gán kiểu, hoặc sập ứng dụng do sai kiểu ở thời gian chạy.

## Câu hỏi thực chiến
**Hỏi:** Vì sao trong Kotlin `List` kiểu `String` gán được cho `List` kiểu `Any`, nhưng `MutableList` kiểu `String` lại không gán được cho `MutableList` kiểu `Any`?
Bởi vì `List` trong Kotlin chỉ cho đọc (*read-only*) và khai báo `out E`, cho phép hiệp biến an toàn vì chỉ có hàm đọc. `MutableList` có hàm thêm phần tử, nếu cho phép gán thì bạn có thể thêm một số nguyên vào danh sách chuỗi. Trình biên dịch bắt buộc `MutableList` phải bất biến về kiểu (*invariant*) để bảo vệ an toàn dữ liệu.

**Hỏi:** Vì sao Java cho gán `String[]` vào `Object[]` nhưng Kotlin cấm gán `Array<String>` vào `Array<Any>`?
Mảng trong Java có tính hiệp biến nên lỗi gán sai chỉ bị phát hiện lúc chạy qua ngoại lệ `ArrayStoreException`. Kotlin thiết kế `Array<T>` bất biến về kiểu (*invariant*) để trình biên dịch chặn đứng lỗi ngay từ đầu.

**Hỏi:** Quy tắc nhớ PECS trong Java tương ứng với từ khóa nào trong Kotlin?
Tương ứng trực tiếp với `out` và `in`. Chữ P và E là Producer Extends trong Java, tương ứng với từ khóa `out` dành cho bên phát dữ liệu trong Kotlin. Chữ C và S là Consumer Super, tương ứng với từ khóa `in` dành cho bên tiêu thụ dữ liệu trong Kotlin.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được Type Erasure ở bytecode và cơ chế chèn lệnh checkcast tự động của compiler.
- [ ] Giải thích nguyên lý out cho producer (hiệp biến) và in cho consumer (nghịch biến).
- [ ] So sánh Array bất biến của Kotlin với mảng hiệp biến dễ lỗi runtime của Java.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

