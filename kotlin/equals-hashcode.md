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
Khi tìm người dùng trong `HashMap` hay `HashSet`, máy dùng `hashCode` chọn ngăn chứa rồi gọi `equals` so từng món. Vì vậy hai đối tượng bằng nhau qua `equals` bắt buộc phải trả về cùng một số `hashCode`. Nếu làm sai luật này, bạn vừa thêm đối tượng vào xong tìm lại ngay cũng không thấy.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1953, Hans Peter Luhn tại IBM tạo ra bảng băm (hash table) để tìm dữ liệu với tốc độ O(1). Nhờ vậy, máy không phải dò từng ô từ đầu mảng với tốc độ O(N). Hàm băm biến mỗi khóa thành một số thứ tự ngăn chứa (`bucket index`) trong mảng.

Năm 1995, Java đưa hai hàm `equals` và `hashCode` vào lớp gốc `Object` cho mọi đối tượng. Từ đó có một luật bắt buộc: nếu `a.equals(b)` trả về `true` thì `a.hashCode()` và `b.hashCode()` phải ra cùng một con số.

Rất nhiều người mới học chỉ viết lại hàm `equals` để so nội dung mà quên viết lại hàm `hashCode`. Khi đó, `HashMap` và `HashSet` cất vào một ngăn nhưng tìm ở ngăn khác, làm đối tượng như bốc hơi.

## Nó thực sự là cái gì
Luật đi đôi giữa `equals` và `hashCode` không phải để học vẹt, mà là điều kiện để bảng băm chạy đúng. Bạn cứ hình dung `hashCode` giúp nhảy tới đúng ngăn tủ, còn `equals` giúp nhìn mặt từng món đồ trong ngăn đó. Trong Kotlin, khi khai báo [data class](concept:kotlin-sealed-vs-data-class), trình biên dịch tự sinh hai hàm này theo các thuộc tính trong ngoặc tròn.

## Cơ chế
Khi bạn cất đối tượng vào `HashSet`, nó gọi `hashCode()` rồi chia lấy dư (hoặc lọc bit) để tính ra số thứ tự ngăn chứa. Tới đúng ngăn đó, nó mới gọi `equals()` so với các phần tử đang nằm sẵn xem có bị trùng không. Lúc bạn gọi `contains()` để kiểm tra, `HashSet` cũng tính lại số ngăn từ `hashCode()` y hệt như vậy. Nếu hai đối tượng bằng nhau lại có mã băm khác nhau, máy mở nhầm ngăn và báo không có.

## Đánh đổi
Nhờ chia ngăn bằng mã băm, việc tìm kiếm hay kiểm tra trùng lặp đạt tốc độ O(1) dù có hàng vạn phần tử. Đổi lại, mọi thuộc tính dùng để tính `hashCode` bắt buộc phải giữ nguyên (`val`) suốt thời gian đối tượng nằm trong bảng. Nếu dùng thuộc tính `var` rồi sửa giá trị, đối tượng kẹt ở ngăn cũ và không thể tìm hay xóa.

## Nguyên lý gốc
Thay vì lục tung kho hàng lớn, ta chia thành nhiều ngăn đánh số. Khi tìm chỉ cần lục một ngăn (spatial partitioning).

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
Bẫy hay gặp nhất là dùng đối tượng có thuộc tính `var` làm khóa (`key`) của `HashMap` rồi sửa biến đó. Khi đổi giá trị, mã băm trỏ sang ngăn mới nên `map[key]` trả về `null`, dù đối tượng cũ vẫn chiếm RAM. Bẫy thứ hai là viết `fun equals(other: UserId)` thiếu `override` và kiểu `Any?`, thành nạp chồng thay vì ghi đè. Đừng bao giờ lấy đối tượng có thuộc tính `var` tham gia tính mã băm đi làm khóa cho `HashMap` hay phần tử của `HashSet`.

## Câu hỏi thực chiến
**Hỏi:** Hai đối tượng có cùng `hashCode` thì chắc chắn sẽ bằng nhau qua `equals` đúng không?
Không, vì `hashCode` chỉ là số nguyên 32-bit nên hai đối tượng khác nhau vẫn có thể trùng mã (đụng độ băm). Khi đó `HashMap` xếp cả hai vào chung một ngăn, rồi gọi tiếp `equals()` từng cái để lấy ra đúng đối tượng bạn cần.

**Hỏi:** Mọi thuộc tính khai báo trong `data class` đều tự động tham gia vào `equals` và `hashCode` à?
Không, trình biên dịch chỉ lấy các thuộc tính khai báo trong ngoặc tròn của hàm khởi tạo chính (`primary constructor`). Thuộc tính viết trong ngoặc nhọn thân lớp sẽ bị bỏ qua. Hai đối tượng khác nhau ở phần thân vẫn được coi là bằng nhau.

**Hỏi:** `data class` chứa thuộc tính kiểu mảng `Array` hoặc `ByteArray` có gì nguy hiểm?
Hàm `equals` tự sinh sẽ so sánh địa chỉ bộ nhớ của mảng chứ không so từng phần tử bên trong. Do mảng trên JVM không so nội dung, hai đối tượng chứa hai mảng giống hệt nhau vẫn bị báo khác nhau. Khi có mảng trong `data class`, bạn phải tự viết lại `equals` và `hashCode` bằng `contentEquals()` và `contentHashCode()`.

**Hỏi:** Khi nào tuyệt đối không được dùng một object làm key trong `HashMap`?
Khi đối tượng đó dùng biến `var` có thể thay đổi để tính `hashCode`. Nếu bạn nhét nó vào `HashMap` rồi mới sửa biến `var` đó, lần sau tìm lại hàm `hashCode()` sẽ chỉ sang một ngăn khác. Lúc đó gọi `get()` sẽ trả về `null`, còn đối tượng cũ thì nằm kẹt mãi trong ngăn cũ gây rò rỉ bộ nhớ.

**Hỏi:** Điều gì xảy ra nếu ta luôn cho hàm `hashCode()` trả về một hằng số cố định như 0?
Code vẫn chạy ra kết quả đúng, nhưng mọi phần tử bạn thêm vào đều bị dồn hết vào chung một ngăn số 0. Từ Java 8, nếu ngăn quá 8 phần tử và khóa cài đặt `Comparable`, `HashMap` chuyển ngăn thành cây đỏ đen O(log N). Còn nếu khóa không so sánh lớn nhỏ được, tốc độ tìm kiếm sẽ tụt về dò từng cái O(N) như một danh sách thường.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được luật bắt buộc: hai đối tượng bằng nhau qua equals thì phải trả về cùng một số hashCode.
- [ ] Trình bày được vì sao sửa thuộc tính var sau khi đưa vào HashSet lại làm đối tượng bị kẹt ở ngăn cũ và tìm không thấy.
- [ ] Chỉ ra được lỗi viết fun equals(other: UserId) thay vì override fun equals(other: Any?) và bẫy dùng mảng trong data class.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
