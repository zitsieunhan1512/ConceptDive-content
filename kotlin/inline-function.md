---
id: kotlin-inline-function
title: Inline Functions
domain: kotlin
tags: [kotlin, bytecode, performance, optimization]
collections: [hay-quen]
related: [kotlin-scope-functions]
diagrams: [Cơ chế|inline-bytecode-expansion.png|Hàm thường cấp phát đối tượng Function trên Heap; inline function dán thẳng bytecode vào nơi gọi để triệt tiêu chi phí]
created: 2026-09-07
---

## Bản 30 giây
Khi bạn truyền lambda vào hàm thường trong vòng lặp, máy phải tạo đối tượng tạm trên Heap nên dễ gây chậm. Từ khóa `inline` bảo compiler chép thẳng thân hàm và lambda vào ngay chỗ gọi. Nhờ vậy, code chạy nhanh như viết vòng lặp tay mà bạn vẫn tách hàm gọn gàng.

## Cần biết trước

## Vấn đề buộc nó ra đời
Tháng 1/2012, Andrey Breslav viết về lộ trình Kotlin và nêu khó khăn khi chạy lambda trên JVM 6 và 7.

Thời đó, mỗi lambda dùng biến bên ngoài (closure) đều biến thành một anonymous class trên Heap. Gọi hàm nhận lambda trong vòng lặp sẽ tạo đối tượng `Function` liên tục và bắt Garbage Collector dọn rác vất vả.

JetBrains cần một cách viết hàm bậc cao không tốn thêm bộ nhớ (zero-overhead). Họ muốn bạn dùng lambda thoải mái mà tốc độ vẫn ngang vòng lặp `for` thường.

## Nó thực sự là cái gì
Inline function không phải phép màu làm mọi hàm tự chạy nhanh hơn. Khi bạn gắn `inline`, compiler chép toàn bộ bytecode của hàm đó và của lambda dán thẳng vào chỗ gọi.

## Cơ chế
Lúc biên dịch, thân hàm và lambda được chép vào ngay chỗ gọi nên không cần tạo đối tượng `Function` trên Heap. Vì code lambda đã nằm thẳng trong hàm ngoài, lệnh `return` trong lambda sẽ thoát luôn khỏi hàm ngoài đó (non-local return). Với `reified`, compiler nhìn thấy kiểu thật tại chỗ gọi nên thay chữ `T` bằng class thật trong bytecode. Nhờ vậy, bạn viết được `T::class.java` lúc chạy dù JVM vốn xóa kiểu generic (type erasure).

## Đánh đổi
Bạn không tốn bộ nhớ Heap cho lambda và bớt được một lần gọi hàm. Đổi lại, nếu gắn `inline` cho hàm dài mà gọi ở nhiều nơi, file bytecode sẽ phình to. Hàm `inline` public cũng không đọc trực tiếp được biến `private` của class. Ngoài ra, bạn không cất lambda vào biến được nếu không gắn thêm `noinline`.

## Nguyên lý gốc
Chấp nhận chép lặp lại mã lệnh ở nhiều chỗ để lúc chạy không tốn công tạo đối tượng và gọi hàm (Space-Time Tradeoff).

## Key insight
> `inline` không làm thân hàm chạy nhanh hơn, mà bỏ bước tạo đối tượng lambda bằng cách chép bytecode vào chỗ gọi.

## Ví von
Thay vì in sách riêng rồi chuyền tay nhau đọc, bạn sao chép đoạn văn đó dán thẳng vào sổ tay của từng người.

## Khuôn mẫu chuẩn
```kotlin
inline fun <reified T : Parcelable> Intent.getParcelableCompat(key: String): T? {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        getParcelableExtra(key, T::class.java)
    } else {
        @Suppress("DEPRECATION")
        getParcelableExtra(key) as? T
    }
}
```

## Bẫy & ranh giới
Nếu bạn gắn `inline` cho hàm dài được gọi khắp nơi, file APK sẽ phình to và thời gian build tăng lên. Khi đưa lambda vào một object khác hoặc luồng khác, bạn phải thêm `crossinline` để cấm lệnh `return` thoát khỏi hàm ngoài. Muốn cất lambda vào biến hoặc truyền sang hàm thường, bạn phải gắn `noinline` cho tham số đó. Đừng gắn `inline` cho hàm không có tham số lambda, vì chỉ làm to bytecode mà không nhanh hơn.

## Câu hỏi thực chiến
**Hỏi:** Cứ hàm nào gọi thường xuyên thì gắn từ khóa `inline` cũng sẽ giúp chạy nhanh hơn à?
Không phải. Nếu hàm không nhận tham số lambda, gắn `inline` chỉ làm phình kích thước file bytecode mà không giúp chạy nhanh hơn. Máy ảo vốn tự inline các hàm ngắn hay được gọi, nên chính compiler Kotlin cũng cảnh báo lợi ích là không đáng kể.

**Hỏi:** `crossinline` và `noinline` khác nhau như thế nào khi viết hàm inline?
Khác ở chỗ code của lambda có được chép vào nơi gọi hay không. Với `crossinline`, code vẫn được chép vào nhưng cấm viết `return` thoát thẳng khỏi hàm bao ngoài. Còn `noinline` giữ lambda thành một đối tượng như thường để bạn cất vào biến hoặc truyền sang hàm khác.

**Hỏi:** Vì sao từ khóa `reified` bắt buộc phải đi chung với `inline`?
Vì [kiểu generic bị xoá](concept:kotlin-generics-variance) ngay lúc biên dịch, khi chạy `T` không còn thông tin gì. Gắn `inline` thì compiler chép mã tới từng nơi gọi, nơi đó biết rõ `T` là class nào. Nó thay `T` bằng class thật trong bytecode nên `is T` hay `T::class` mới dùng được.

**Hỏi:** Khi nào tuyệt đối không nên gắn `inline` cho một hàm?
Khi thân hàm dài và được gọi ở hàng trăm chỗ trong dự án. Chép lặp lại một đoạn code lớn ra nhiều nơi sẽ làm file APK nặng thêm. Code quá dài còn làm đầy bộ nhớ đệm lệnh của CPU nên chạy chậm hơn.

**Hỏi:** Lệnh `return` bên trong một lambda inline có thoát khỏi chính lambda đó không?
Không, nó thoát thẳng khỏi cả hàm cha đang bọc bên ngoài. Lý do là code của lambda đã được chép hòa vào thân hàm cha (non-local return). Nếu chỉ muốn thoát khỏi riêng lambda đó, bạn phải viết `return@tênHàm` có gắn nhãn.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách compiler chép thẳng bytecode vào chỗ gọi để không phải tạo đối tượng Function trên Heap.
- [ ] Nêu được vì sao inline cho phép non-local return và giúp reified giữ lại class thật cho kiểu T.
- [ ] Chỉ ra được bẫy làm phình bytecode khi inline hàm dài và cách dùng noinline, crossinline.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

