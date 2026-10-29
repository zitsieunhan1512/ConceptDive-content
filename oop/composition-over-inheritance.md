---
id: oop-composition-over-inheritance
title: Composition over Inheritance
domain: oop
tags: [oop, architecture, design-pattern, composition, inheritance, delegation]
collections: [interview-fundamental]
related: [oop-abstract-class-vs-interface, oop-four-pillars, oop-solid, architecture-dependency-injection]
created: 2026-10-01
sources:
  - Implementation by delegation — Kotlin Documentation | https://kotlinlang.org/docs/delegation.html
  - Composition over inheritance — Wikipedia | https://en.wikipedia.org/wiki/Composition_over_inheritance
  - Fragile base class — Wikipedia | https://en.wikipedia.org/wiki/Fragile_base_class
diagrams: [Cơ chế|composition-fragile-base-class.png|Kế thừa HashSet làm addCount bị đếm trùng do gọi nội bộ, còn lắp ghép ủy quyền bảo toàn tính đóng gói]
---

## Bản 30 giây
Khi bạn kế thừa một lớp chỉ để dùng lại vài hàm, sửa một chút ở lớp cha rất dễ làm hỏng lớp con. Thay vì bắt lớp mới làm con của lớp cũ, bạn hãy đặt các khối chức năng vào bên trong như những biến thành phần. Nhờ vậy, bạn đổi được hành vi ngay lúc chạy và lỗi ở khối nào chỉ nằm yên trong khối đó.

## Cần biết trước
- oop-four-pillars
- oop-abstract-class-vs-interface

## Vấn đề buộc nó ra đời
Năm 1994, nhóm Gang of Four đúc kết nguyên lý ưu tiên lắp ghép hơn kế thừa trong cuốn sách mẫu thiết kế kinh điển. Kế thừa làm vỡ tính đóng gói vì lớp con bị lệ thuộc thẳng vào cách lớp cha tự gọi hàm bên trong.

Năm 2001, Joshua Bloch khuyên ưu tiên lắp ghép hơn kế thừa trong sách Effective Java. Đến năm 2003, Allen Holub mổ xẻ lỗi lớp cơ sở mong manh (*Fragile Base Class*) trong bài viết 'Why extends is evil'.

Vì vậy, Kotlin để mọi lớp mặc định là `final`, đúng theo lời khuyên thiết kế rõ cho kế thừa hoặc khóa lại. Đồng thời, ngôn ngữ thêm từ khóa `by` để bạn chuyển tiếp lệnh gọi hàm thật gọn.

## Nó thực sự là cái gì
Lắp ghép không hề phủ nhận hay đối chọi với tính đa hình. Thay vì khóa cứng quan hệ cha con từ lúc biên dịch, bạn nối các đối tượng với nhau linh hoạt lúc chạy. Kế thừa thể hiện quan hệ "là một" (IS-A), còn lắp ghép thể hiện quan hệ "có chứa" (HAS-A). Đối tượng bọc ngoài giữ biến trỏ tới đối tượng bên trong rồi nhờ nó làm hộ, nên giấu kín được cách cài đặt.

## Cơ chế
Đối tượng bọc ngoài giữ các đối tượng thành phần làm biến riêng trên bộ nhớ Heap. Khi bạn gọi hàm, nó chỉ chuyển lời gọi đó sang cho đối tượng thành phần bên trong xử lý. Với từ khóa `by` trong Kotlin, trình biên dịch tự sinh sẵn các hàm chuyển tiếp cho toàn bộ `interface`. Nhờ đó, bạn không phải gõ tay từng hàm lặp lại mà vẫn được kiểm tra kiểu an toàn.

## Đánh đổi
Lắp ghép giúp code dễ viết unit test và dễ tráo đổi hành vi ngay lúc ứng dụng đang chạy. Đổi lại, chương trình tốn thêm một đối tượng bọc ngoài trên Heap và thêm một bước gọi hàm bắc cầu. Nếu ngôn ngữ không có từ khóa `by`, bạn sẽ phải tự gõ tay rất nhiều hàm chuyển tiếp trung gian.

## Nguyên lý gốc
Hệ thống luôn bền hơn khi ghép từ các khối rời có thể tháo lắp độc lập (loose coupling). Khi cần thêm tính năng, bạn nên ghép thêm khối mới thay vì sửa vào cây kế thừa cũ.

## Key insight
> Kế thừa trói lớp con vào dòng họ lúc biên dịch, còn lắp ghép cho phép đổi khối chức năng lúc chạy.

## Ví von
Kế thừa giống cơ thể sinh học bị gắn chặt gen di truyền từ đời bố mẹ. Lắp ghép giống túi đồ nghề của người thợ: cần kìm thì rút kìm, hỏng tua vít thì đổi cái mới ngay.

## Khuôn mẫu chuẩn
```kotlin
class CountingSet<T>(private val inner: MutableSet<T> = HashSet()) :
    MutableSet<T> by inner {
    var addCount = 0
    override fun add(element: T): Boolean {
        addCount++
        return inner.add(element)
    }
    override fun addAll(elements: Collection<T>): Boolean {
        addCount += elements.size
        return inner.addAll(elements)
    }
}
```

## Bẫy & ranh giới
Nếu kế thừa chỉ để lấy một hàm tiện ích, lớp con phải gánh theo mọi hàm thừa của lớp cha. Đừng dùng kế thừa khi hai lớp không có quan hệ bản chất "là một" (IS-A) rõ ràng. Đặc biệt, ghi đè hàm mà lớp cha tự gọi lại chính nó sẽ làm biến đếm bị cộng trùng. Triệu chứng là nâng cấp thư viện cha cũng làm lớp con chạy sai, hoặc muốn tách lớp đi nơi khác đều vướng.

## Câu hỏi thực chiến
**Hỏi:** Vì sao lớp `CountingSet` kế thừa `HashSet` đếm sai khi gọi `addAll`?
Bên trong `HashSet`, hàm `addAll` tự gọi hàm `add` cho từng phần tử. Vì `add` đã bị lớp con ghi đè nên mỗi phần tử bị cộng hai lần. Khi lắp ghép bằng `by`, lớp bọc chỉ gọi chuyển tiếp vào `inner` nên tránh được vòng gọi ngầm của lớp cha.

**Hỏi:** Cơ chế ủy quyền `by` trong Kotlin có giải quyết được nhược điểm gõ nhiều mã chuyển tiếp của mẫu Wrapper không?
Có. Lúc biên dịch, Kotlin tự sinh toàn bộ các hàm chuyển tiếp cho những hàm có trong `interface`. Bạn chỉ cần viết lại đúng hàm nào muốn thêm logic, vừa gọn code vừa giữ nguyên lợi thế của lắp ghép.

**Hỏi:** Tại sao lớp cơ sở mong manh (*Fragile Base Class*) lại là thảm họa lớn nhất của kế thừa?
Vì ở bản cập nhật sau, tác giả lớp cha có thể đổi cách các hàm bên trong gọi lẫn nhau. Lớp con ghi đè một hàm sẽ vô tình làm chệch luồng chạy mới mà tác giả lớp cha không hề biết. Hậu quả là ứng dụng tính sai dữ liệu âm thầm lúc chạy dù biên dịch không báo một lỗi nào.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được quan hệ IS-A (kế thừa) và HAS-A (lắp ghép đối tượng thành phần).
- [ ] Giải thích được ví dụ CountingSet kế thừa HashSet bị đếm gấp đôi vì addAll gọi ngầm hàm add.
- [ ] Trình bày được cách từ khóa by trong Kotlin tự sinh các hàm chuyển tiếp ở lúc biên dịch.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

