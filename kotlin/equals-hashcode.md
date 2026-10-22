---
id: kotlin-equals-hashcode
title: equals & hashCode
domain: kotlin
tags: [kotlin, data-structure, hashmap, memory]
collections: [interview-fundamental]
related: [kotlin-sealed-vs-data-class]
diagrams: [Cơ chế|hashset-bucket-lookup.png|HashSet tính chỉ số ngăn từ hashCode rồi dùng equals so khớp; sửa var làm lệch ngăn dẫn tới mất tích đối tượng]
created: 2026-09-07
---

## Bản 30 giây
Bảng tra cứu nhanh (`HashMap`, `HashSet`) dùng mã băm chia ngăn, còn phép so bằng tìm đúng món đồ trong ngăn đó. Cặp hàm này giữ hợp đồng: hai đối tượng bằng nhau bắt buộc phải có cùng mã băm. Nếu vi phạm, máy tính tìm sai ngăn và coi như đối tượng không tồn tại.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1953, Hans Peter Luhn tại IBM phát minh ra bảng băm (hash table). Mục tiêu là tra cứu dữ liệu tức thì trong thời gian O(1) thay vì quét mảng O(N). Thuật toán băm chuyển đổi khóa thành chỉ số ngăn chứa (bucket index) trong mảng.

Năm 1995, Java đưa `equals` và `hashCode` vào lớp gốc `Object`. Quyết định này đặt ra hợp đồng bất di bất dịch: nếu `a.equals(b)` đúng thì mã băm của chúng phải giống hệt nhau.

Lập trình viên thường chỉ override `equals` mà quên `hashCode`. Hậu quả là `HashMap` và `HashSet` tính sai ngăn chứa, làm mất dấu phần tử một cách bí ẩn trong bộ nhớ.

## Nó thực sự là cái gì
Hợp đồng này KHÔNG phải lý thuyết giáo điều. Thực chất, đó là bất biến (invariant) của bảng băm: `hashCode` định vị ngăn chứa, còn `equals` tìm đúng đối tượng trong ngăn. Trong Kotlin, khai báo [data class](concept:kotlin-sealed-vs-data-class) giúp trình biên dịch tự động sinh cặp hàm chuẩn xác này.

## Cơ chế
`HashSet` gọi `hashCode()` để tính chỉ số ngăn chứa qua phép lấy dư hoặc mặt nạ bit. Sau khi đến đúng ngăn chứa, nó mới dùng `equals()` để so sánh từng phần tử. Khi gọi `contains()`, quy trình lặp lại y hệt để tìm đối tượng. Nếu mã băm bị lệch, máy tính sẽ lục soát nhầm ngăn chứa và báo không tìm thấy.

## Đánh đổi
Tuân thủ hợp đồng giúp tìm kiếm và kiểm tra tồn tại đạt tốc độ O(1). Đổi lại, các trường tham gia tính mã băm bắt buộc phải bất biến. Nếu dùng trường thay đổi (`var`), đối tượng sẽ kẹt vĩnh viễn trong tập hợp thành "hồn ma" không thể xóa.

## Nguyên lý gốc
Nguyên lý phân vùng không gian (spatial partitioning): chia không gian tìm kiếm thành các ô nhỏ để thu hẹp phạm vi quét dữ liệu.

## Key insight
> Hai đối tượng bằng nhau bắt buộc phải cùng hashCode; sửa biến tạo hashCode sau khi lưu vào HashSet sẽ làm mất tích đối tượng.

## Khuôn mẫu chuẩn
```kotlin
class UserId(val value: String) {
    override fun equals(other: Any?): Boolean =
        other is UserId && other.value == value

    override fun hashCode(): Int = value.hashCode()
}

val set = hashSetOf(UserId("A"))
val exists = set.contains(UserId("A"))
```

## Bẫy & ranh giới
Bẫy kinh điển là dùng `var` trong class làm key của `HashMap`. Khi sửa giá trị, mã băm đổi khác khiến `map.get(key)` trả về `null` dù key vẫn nằm trong map. Bẫy thứ hai là viết `equals(other: UserId)` thay vì `equals(other: Any?)`, biến thành hàm overload chứ không ghi đè. Chống chỉ định dùng đối tượng chứa `var` làm key của collection băm vì gây rò rỉ bộ nhớ dưới dạng các đối tượng ma.

## Câu hỏi thực chiến
**Hỏi:** Hai đối tượng có cùng `hashCode` thì chắc chắn sẽ bằng nhau qua `equals` đúng không?
Không phải. Mã băm chỉ có kích thước 32-bit nên hai đối tượng khác nhau hoàn toàn có thể trùng mã do xung đột băm. `HashMap` khi đó xếp chúng chung một ngăn và bắt buộc phải gọi `equals()` để tìm đúng món đồ.

**Hỏi:** Mọi thuộc tính khai báo trong `data class` đều tự động tham gia vào `equals` và `hashCode` à?
Không. Compiler chỉ dùng các thuộc tính khai báo trong hàm khởi tạo chính của `data class`. Mọi biến khai báo bên trong thân class đều bị bỏ qua, khiến hai đối tượng có thân khác nhau vẫn bằng nhau.

**Hỏi:** `data class` chứa thuộc tính kiểu mảng `Array` hoặc `ByteArray` có gì nguy hiểm?
Phép so sánh sẽ so nhầm địa chỉ bộ nhớ thay vì kiểm tra nội dung mảng. Mảng trên JVM không ghi đè `equals`, khiến hai đối tượng có mảng cùng nội dung vẫn bị coi là khác nhau. Muốn so đúng nội dung, bạn bắt buộc phải tự override lại và dùng `contentEquals()` cùng `contentHashCode()`.

**Hỏi:** Khi nào tuyệt đối không được dùng một object làm key trong `HashMap`?
Khi đối tượng chứa bất kỳ biến có thể thay đổi nào. Nếu bạn đổi giá trị biến sau khi đưa vào bảng băm, mã băm sẽ tính ra chỉ số ngăn khác. Khi đó hàm `get()` sẽ trả về `null` dù đối tượng vẫn nằm kẹt lại và gây rò rỉ bộ nhớ.

**Hỏi:** Điều gì xảy ra nếu ta luôn cho hàm `hashCode()` trả về một hằng số cố định như 0?
Chương trình vẫn đúng nhưng mọi phần tử dồn vào một ngăn duy nhất. Từ Java 8, `HashMap` biến ngăn quá đông thành cây đỏ đen nên tra cứu còn O(log N) nếu key là `Comparable`. Key không `Comparable` thì có thể tụt về duyệt tuần tự O(N).

## Ý bắt buộc khi giải thích lại
- [ ] Hợp đồng cốt lõi: hai đối tượng equals bằng nhau thì hashCode bắt buộc phải bằng nhau.
- [ ] Bẫy sửa biến var làm đối tượng biến mất khỏi HashSet dù vẫn nằm trong bộ nhớ.
- [ ] Bẫy viết sai tham số hàm equals biến thành overload thay vì override.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
