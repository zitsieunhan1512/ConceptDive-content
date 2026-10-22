---
id: oop-composition-over-inheritance
title: Composition over Inheritance
domain: oop
tags: [oop, architecture, design-pattern, composition, inheritance, delegation]
collections: [interview-fundamental]
related: [oop-abstract-class-vs-interface, oop-four-pillars, oop-solid, architecture-dependency-injection]
created: 2026-10-01
diagrams: [Cơ chế|composition-fragile-base-class.png|Kế thừa HashSet làm addCount bị đếm trùng do gọi nội bộ, còn lắp ghép ủy quyền bảo toàn tính đóng gói]
---

## Bản 30 giây
Kế thừa lớp trói buộc chặt chẽ mã nguồn và dễ làm hỏng hành vi lớp con khi lớp cha thay đổi. Thay vì bắt đối tượng phải là một biến thể của lớp khác, ta lắp ghép các khối chức năng độc lập vào bên trong. Cách làm này giúp hoán đổi tính năng linh hoạt khi chạy và khoanh vùng lỗi trong từng khối.

## Cần biết trước
- oop-four-pillars
- oop-abstract-class-vs-interface

## Vấn đề buộc nó ra đời
Năm 1994, nhóm Gang of Four đúc kết nguyên lý ưu tiên thành phần hơn kế thừa trong cuốn sách mẫu thiết kế kinh điển. Kế thừa phá vỡ tính đóng gói vì lớp con phụ thuộc trực tiếp vào chi tiết cài đặt của lớp cha.

Năm 2001, Joshua Bloch xuất bản cuốn sách Effective Java với lời khuyên ưu tiên thành phần hơn kế thừa. Năm 2003, Allen Holub phân tích thảm họa lớp cơ sở mong manh (*Fragile Base Class*) trong bài viết 'Why extends is evil'.

Kotlin để mọi lớp mặc định là `final`, theo lời khuyên của Effective Java về việc thiết kế cho kế thừa hoặc cấm nó. Đồng thời, ngôn ngữ bổ sung từ khóa `by` để ủy quyền phương thức gọn gàng.

## Nó thực sự là cái gì
Lắp ghép KHÔNG PHẢI là kỹ thuật lập trình đối lập hay phủ nhận hoàn toàn tính đa hình. Thực chất, đây là sự chuyển dịch từ liên kết tĩnh ở thời điểm biên dịch sang liên kết động ở thời gian chạy.

Kế thừa thể hiện quan hệ bản thể (IS-A), trong khi lắp ghép thể hiện quan hệ sở hữu thành phần (HAS-A).

Đối tượng chứa tham chiếu tới các bộ phận và ủy quyền xử lý, giúp che giấu hoàn toàn cấu trúc bên trong.

## Cơ chế
Đối tượng bao bọc lưu các thể hiện hành vi thành các trường dữ liệu riêng biệt trên bộ nhớ Heap. Khi có yêu cầu, nó gọi chuyển tiếp phương thức tới đối tượng thành phần thích hợp.

Với từ khóa `by` trong Kotlin, trình biên dịch tự động sinh mã chuyển tiếp cho mọi hàm của giao diện.

Cơ chế này loại bỏ hoàn toàn mã thừa mà vẫn bảo toàn kiểm tra kiểu an toàn.

## Đánh đổi
Lắp ghép giúp mã nguồn mềm dẻo, dễ kiểm thử đơn vị và dễ thay thế hành vi khi chạy. Đổi lại, hệ thống phải cấp phát thêm đối tượng chuyển tiếp và tốn thêm chi phí gọi gián tiếp.

Chữ ký hàm có thể bị phân mảnh nếu phải tự viết tay nhiều phương thức ủy quyền trung gian.

## Nguyên lý gốc
Nguyên lý ghép nối lỏng: hệ thống luôn bền vững hơn khi cấu tạo từ các khối độc lập có thể tháo rời. Tính mở rộng nên dựa trên sự kết hợp thay vì biến đổi cấu trúc sẵn có.

## Key insight
> Kế thừa trói buộc vào phả hệ tĩnh, còn lắp ghép tổ hợp năng lực độc lập linh hoạt khi chạy.

## Ví von
Kế thừa giống như sinh vật lai ghép bị trói buộc gen từ đời cha mẹ. Lắp ghép giống túi đồ nghề, cần kìm thì lấy kìm, hỏng tua vít thì thay cái mới mà không ảnh hưởng người thợ.

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
Bẫy kế thừa để dùng lại một hàm tiện ích khiến lớp con gánh trọn vẹn mọi phương thức thừa của cha. Chống chỉ định dùng kế thừa khi giữa hai lớp không thỏa mãn mối quan hệ bản thể IS-A một cách tự nhiên. Tránh tạo lớp con ghi đè hàm trong chuỗi gọi nội bộ của cha vì rất dễ gây đếm trùng hoặc sai trạng thái. Triệu chứng dùng sai là lỗi vỡ hành vi lớp con khi thư viện cha cập nhật, hoặc mã nguồn bị trói chặt khó gỡ.

## Câu hỏi thực chiến
**Hỏi:** Vì sao lớp `CountingSet` kế thừa `HashSet` đếm sai khi gọi `addAll`?
`addAll` của lớp cha gọi lại phương thức `add` vốn đã bị ghi đè, khiến mỗi phần tử bị đếm hai lần. Lắp ghép bằng `by` giúp lớp bao bọc chuyển tiếp độc lập mà không phụ thuộc chi tiết cài đặt của cha.

**Hỏi:** Cơ chế ủy quyền `by` trong Kotlin có giải quyết được nhược điểm gõ nhiều mã chuyển tiếp của mẫu Wrapper không?
Có. Trình biên dịch Kotlin sẽ tự động sinh mã chuyển tiếp cho toàn bộ các hàm được khai báo trong giao diện. Bạn không cần phải viết tay từng phương thức ủy quyền, giúp mã nguồn vừa sạch gọn vừa giữ trọn lợi thế lắp ghép.

**Hỏi:** Tại sao lớp cơ sở mong manh (*Fragile Base Class*) lại là thảm họa lớn nhất của kế thừa?
Bởi vì lớp cha có thể thay đổi cách các hàm nội bộ gọi lẫn nhau trong các bản cập nhật tiếp theo. Một lớp con ghi đè phương thức vô tình làm thay đổi luồng thực thi mà tác giả lớp cha không hề lường trước. Kết quả là lớp con bị lỗi logic âm thầm hoặc làm hỏng dữ liệu mà không hề báo lỗi khi biên dịch.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt bản chất quan hệ IS-A (kế thừa) và HAS-A (lắp ghép).
- [ ] Nêu ví dụ CountingSet đếm sai do addAll gọi nội bộ hàm add của lớp cha.
- [ ] Trình bày cơ chế ủy quyền class delegation qua từ khóa by của Kotlin ở tầng compiler.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

