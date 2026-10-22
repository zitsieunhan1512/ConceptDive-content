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
Truyền lambda vào hàm thường ép hệ thống tạo đối tượng tạm trên Heap, gây chậm máy trong vòng lặp. Từ khóa inline chỉ thị compiler sao chép mã nguồn của hàm và lambda dán thẳng vào nơi gọi. Nhờ đó, chương trình chạy nhanh như viết code tay mà vẫn giữ được tính mô-đun.

## Cần biết trước

## Vấn đề buộc nó ra đời
Tháng 1/2012, Andrey Breslav công bố bài viết về lộ trình Kotlin. Ông chỉ ra thách thức lớn khi đưa lambda lên máy ảo JVM 6 và 7.

Mỗi lambda bắt biến (closure) thời đó đều bị biên dịch thành một anonymous class trên Heap. Việc khởi tạo đối tượng `Function` liên tục trong vòng lặp tạo áp lực khủng khiếp lên Garbage Collector.

JetBrains cần một giải pháp không tốn phụ phí bộ nhớ (zero-overhead) cho các hàm bậc cao. Họ muốn lập trình viên dùng lambda thoải mái mà tốc độ vẫn ngang ngửa vòng lặp for truyền thống.

## Nó thực sự là cái gì
Inline function KHÔNG phải là công cụ tăng tốc thần kỳ cho mọi hàm. Thực chất, nó là chỉ thị cho compiler copy toàn bộ bytecode của hàm và lambda dán thẳng vào vị trí gọi.

## Cơ chế
Tại thời điểm biên dịch, compiler dán thẳng thân hàm và lambda vào vị trí gọi, triệt tiêu cấp phát `Function` trên Heap. Cơ chế dán mã cho phép `return` thoát thẳng khỏi hàm bao quanh (non-local return). Với `reified`, compiler biết rõ kiểu cụ thể tại nơi gọi nên thay thế trực tiếp `T` bằng class thật trong bytecode. Nhờ đó, chương trình truy cập được `T::class.java` lúc chạy mà không vi phạm cơ chế xóa kiểu (type erasure) của JVM.

## Đánh đổi
Bạn triệt tiêu hoàn toàn việc cấp phát bộ nhớ cho lambda và tăng tốc thực thi. Đổi lại, file bytecode sẽ phình to nếu lạm dụng inline cho các hàm dài được gọi nhiều nơi. Hàm inline public không thể truy cập các trường private của class. Bạn cũng không thể lưu trữ lambda inline vào biến nếu không dùng `noinline`.

## Nguyên lý gốc
Đánh đổi không gian lấy thời gian (Space-Time Tradeoff): nhân bản mã lệnh tại nơi gọi để triệt tiêu chi phí gián tiếp khi chạy.

## Key insight
> Inline function không tăng tốc bản thân hàm, mà triệt tiêu việc cấp phát object lambda bằng cách dán bytecode vào nơi gọi.

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
Lạm dụng inline cho các hàm dài khiến kích thước APK phình to và thời gian biên dịch tăng vọt. Khi gọi lambda trong object lồng hoặc luồng khác, compiler sẽ chặn biên dịch nếu thiếu `crossinline` để cấm non-local return. Muốn lưu trữ lambda vào biến hoặc truyền sang hàm không-inline, bạn bắt buộc phải dùng `noinline` để tránh lỗi biên dịch. Chống chỉ định gắn `inline` cho hàm không nhận tham số lambda vì chỉ gây phình bytecode mà không đem lại lợi ích hiệu năng.

## Câu hỏi thực chiến
**Hỏi:** Cứ hàm nào gọi thường xuyên thì gắn từ khóa `inline` cũng sẽ giúp chạy nhanh hơn à?
Không phải. Nếu hàm không nhận tham số lambda, gắn `inline` chỉ làm phình kích thước file bytecode mà không giúp chạy nhanh hơn. Máy ảo vốn tự inline các hàm ngắn hay được gọi, nên chính compiler Kotlin cũng cảnh báo lợi ích là không đáng kể.

**Hỏi:** `crossinline` và `noinline` khác nhau như thế nào khi viết hàm inline?
Khác ở việc có dán mã bytecode của lambda vào nơi gọi hay không. Tham số `crossinline` vẫn được dán mã nhưng bị cấm `return` thoát ngang khỏi hàm bao quanh. Còn `noinline` giữ nguyên lambda thành một đối tượng thông thường để có thể gán vào biến hoặc truyền đi.

**Hỏi:** Vì sao từ khóa `reified` bắt buộc phải đi chung với `inline`?
Vì [kiểu generic bị xoá](concept:kotlin-generics-variance) ngay lúc biên dịch, khi chạy `T` không còn thông tin gì. Gắn `inline` thì compiler chép mã tới từng nơi gọi, nơi đó biết rõ `T` là class nào. Nó thay `T` bằng class thật trong bytecode nên `is T` hay `T::class` mới dùng được.

**Hỏi:** Khi nào tuyệt đối không nên gắn `inline` cho một hàm?
Khi thân hàm có nhiều dòng code phức tạp hoặc được gọi rải rác khắp dự án. Việc nhân bản một khối mã lớn vào hàng trăm chỗ sẽ làm phình to kích thước file APK. Hơn nữa, bytecode quá lớn còn làm tràn bộ nhớ đệm lệnh của CPU và phản tác dụng hiệu năng.

**Hỏi:** Lệnh `return` bên trong một lambda inline có thoát khỏi chính lambda đó không?
Không, nó sẽ thoát thẳng khỏi cả hàm cha bao quanh bên ngoài. Đặc tính này gọi là non-local return, do mã lambda đã được dán hòa tan vào hàm cha. Nếu chỉ muốn dừng riêng lambda đó, bạn bắt buộc phải dùng lệnh `return` có nhãn định danh.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: compiler dán thẳng bytecode vào call-site, triệt tiêu cấp phát object Function trên Heap.
- [ ] Hỗ trợ non-local return; reified thay T bằng class thật trong bytecode tại nơi gọi.
- [ ] Bẫy phình to kích thước bytecode và bẫy lưu trữ lambda khi thiếu noinline/crossinline.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

