---
id: kotlin-generics-variance
title: Generics & Variance
domain: kotlin
tags: [kotlin, generics, variance, type-erasure, reified]
collections: [interview-fundamental]
related: [kotlin-inline-function, kotlin-sealed-vs-data-class, oop-four-pillars]
sources:
  - Generics: in, out, where — Kotlin | https://kotlinlang.org/docs/generics.html
  - Type Erasure — Oracle Java Tutorials | https://docs.oracle.com/javase/tutorial/java/generics/erasure.html
  - Generics in Java — Wikipedia | https://en.wikipedia.org/wiki/Generics_in_Java
diagrams: [Cơ chế|generics-variance-directions.png|Chiều quan hệ kế thừa trong Generics: out giữ nguyên chiều (hiệp biến), in đảo ngược chiều (nghịch biến)]
created: 2026-10-01
---

## Bản 30 giây
Khi dùng generic, trình biên dịch chặn việc truyền `MutableList<Dog>` vào `MutableList<Animal>` để tránh nhét nhầm `Cat` vào. Để truyền được an toàn, Kotlin tách rõ chiều dữ liệu đi ra và đi vào. Từ khóa `out` cho phép coi `Producer<Dog>` là con của `Producer<Animal>`, còn `in` cho phép coi `Consumer<Animal>` là con của `Consumer<Dog>`.

## Cần biết trước
- oop-four-pillars
- kotlin-inline-function

## Vấn đề buộc nó ra đời
Năm 2004, Java 5 thêm generics nhưng phải xóa thông tin kiểu khi biên dịch (`type erasure`) để chạy trên máy ảo cũ. Ở bytecode, mọi tham số kiểu `T` đều bị đổi về kiểu giới hạn trên (`upper bound`) hoặc `Object`.

Trong Java, mảng thường được phép gán `Dog[]` vào `Animal[]` rồi để tới lúc chạy mới kiểm tra kiểu. Cách này làm chương trình dễ bị sập giữa chừng với lỗi `ArrayStoreException` khi bạn nhét một con `Cat` vào mảng đó.

Để chặn lỗi lúc biên dịch trên tập hợp, Java bắt viết `? extends` và `? super` rất rối ở từng chỗ gọi. Kotlin sửa bằng cách cho bạn khai báo `out` và `in` ngay trên định nghĩa lớp để lúc gọi hàm không phải viết lại nữa.

## Nó thực sự là cái gì
Variance không hề biến đổi hay ép kiểu đối tượng trong bộ nhớ lúc ứng dụng đang chạy. Đây chỉ là quy tắc biên dịch để quyết định khi nào lớp bọc kiểu con thay thế được lớp bọc kiểu cha.

Danh sách sửa được của `Dog` không phải con của danh sách `Animal`, vì làm vậy sẽ nhét được `Cat` vào danh sách `Dog`.

Kotlin dùng `out` cho lớp chỉ trả dữ liệu ra (chỉ đọc), và dùng `in` cho lớp chỉ nhận vào (chỉ ghi).

## Cơ chế
Khi dịch ra bytecode, trình biên dịch xóa sạch các nhãn `<T>` và thay bằng kiểu giới hạn trên (`upper bound`) hoặc `Object`. Tại những chỗ bạn đọc dữ liệu ra khỏi lớp generic, trình biên dịch tự động chèn thêm lệnh ép kiểu `checkcast`.

Khi đánh dấu `out T`, trình biên dịch chỉ cho `T` nằm ở kiểu trả về và cấm làm tham số đầu vào.

Ngược lại, nếu bạn đánh dấu `in T`, `T` chỉ được nằm ở vị trí tham số nhận vào và bị cấm làm kiểu trả về.

## Đánh đổi
Nhờ `out` và `in`, bạn truyền được tập hợp kiểu con vào hàm nhận kiểu cha mà vẫn chặn lỗi sai kiểu. Đổi lại, do kiểu `T` bị xóa ở bytecode nên lúc chạy không viết được `x is T` hay gọi `T()`.

Để giữ kiểu `T` lúc chạy, bạn phải viết [inline function](concept:kotlin-inline-function) kèm `reified`, đổi lại mã bytecode chép ra dài hơn.

## Nguyên lý gốc
Hộp chứa kiểu con chỉ thay thế an toàn cho hộp chứa kiểu cha khi tách bạch hộp lấy ra và hộp bỏ vào.

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
Khi ép kiểu cưỡng bức `list as List<Int>`, lúc ép không báo lỗi vì đã xóa kiểu, nhưng lúc đọc phần tử mới sập `ClassCastException`. Cũng đừng gắn `out` cho những lớp vừa đọc vừa ghi như danh sách `MutableList` hay bộ nhớ đệm. Nếu lạm dụng `List<*>` khắp nơi, bạn sẽ không thêm được phần tử nào và đọc ra chỉ nhận kiểu `Any?`. Triệu chứng là trình biên dịch báo đỏ khi truyền danh sách vào hàm, hoặc sập `ClassCastException` lúc đọc phần tử.

## Câu hỏi thực chiến
**Hỏi:** Vì sao trong Kotlin `List` kiểu `String` gán được cho `List` kiểu `Any`, nhưng `MutableList` kiểu `String` lại không gán được cho `MutableList` kiểu `Any`?
Vì `List` trong Kotlin chỉ đọc và khai báo là `List<out E>`, nên coi danh sách chuỗi như `List<Any>` rất an toàn. Còn `MutableList` có hàm `add` nhận phần tử vào; nếu cho gán vậy thì bạn nhét được `Int` vào danh sách `String`. Do đó trình biên dịch bắt `MutableList` phải giữ nguyên kiểu (`invariant`) để không ai nhét nhầm dữ liệu.

**Hỏi:** Vì sao Java cho gán `String[]` vào `Object[]` nhưng Kotlin cấm gán `Array<String>` vào `Array<Any>`?
Vì mảng Java cho gán kiểu con vào kiểu cha từ trước khi có generics. Lỗi nhét nhầm số vào mảng chuỗi chỉ lộ ra lúc chạy qua `ArrayStoreException`. Kotlin thiết kế lại `Array<T>` giữ nguyên kiểu (`invariant`) để trình biên dịch bắt lỗi ngay lúc bạn đang gõ code.

**Hỏi:** Quy tắc nhớ PECS trong Java tương ứng với từ khóa nào trong Kotlin?
Tương ứng thẳng với hai từ khóa `out` và `in`. Chữ P-E (Producer Extends) bên Java là chỉ lấy dữ liệu ra, tương ứng với từ khóa `out` trong Kotlin. Còn chữ C-S (Consumer Super) bên Java là chỉ nhận dữ liệu vào, tương ứng với từ khóa `in` trong Kotlin.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cơ chế xóa kiểu (type erasure) ở bytecode và việc trình biên dịch tự chèn lệnh ép kiểu checkcast ở chỗ đọc ra.
- [ ] Trình bày được vì sao dùng out cho bên chỉ trả dữ liệu ra (cùng chiều kế thừa) và in cho bên chỉ nhận dữ liệu vào (ngược chiều kế thừa).
- [ ] So sánh được mảng Array giữ nguyên kiểu an toàn từ lúc dịch của Kotlin với mảng dễ nổ lỗi ArrayStoreException lúc chạy của Java.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

