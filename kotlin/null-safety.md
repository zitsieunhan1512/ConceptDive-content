---
id: kotlin-null-safety
title: Null safety
domain: kotlin
tags: [kotlin, type-system, nullability, compiler]
collections: [interview-fundamental]
related: [kotlin-sealed-vs-data-class, kotlin-scope-functions, kotlin-why-over-java]
diagrams: [Cơ chế|null-safety-type-hierarchy.png|Hệ thống kiểu phân nhánh giữa non-nullable và nullable; Platform Type từ Java là lỗ thủng bypass compile-time]
created: 2026-09-07
---

## Bản 30 giây
Khi gọi API mà một trường trả về `null`, ứng dụng rất dễ sập nếu bạn quên kiểm tra. Kotlin tách rõ kiểu có thể null và kiểu không thể null ngay từ lúc bạn khai báo biến. Compiler bắt bạn xử lý trường hợp `null` trước khi biên dịch xong nên lỗi không lọt tới tay người dùng.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1965, Tony Hoare đưa con trỏ null vào ngôn ngữ ALGOL W chỉ vì cách đó rất dễ làm trong compiler.

Từ quyết định đó, lỗi `NullPointerException` xuất hiện khắp nơi và làm tốn hàng triệu giờ gỡ lỗi của lập trình viên.

Trên Java và Android, `NullPointerException` luôn là lỗi làm sập ứng dụng nhiều nhất. Khi làm Kotlin, JetBrains muốn bắt hết lỗi này ngay lúc biên dịch chứ không đợi tới lúc chạy.

## Nó thực sự là cái gì
Null safety không làm giá trị `null` biến mất khỏi bộ nhớ máy tính. Nó chỉ dời việc kiểm tra `null` từ lúc ứng dụng đang chạy (runtime) về lúc bạn đang viết code (compile-time).

## Cơ chế
Kotlin chia mỗi kiểu thành hai nhánh: không nhận `null` (`String`) và có thể nhận `null` (`String?`). Trong code thuần Kotlin, compiler kiểm tra hết lúc biên dịch nên lúc chạy không tốn thêm bước nào. Ở đầu các hàm không phải `private`, compiler chèn thêm lệnh `Intrinsics` để chặn `null` truyền từ code Java sang. Với biến có dấu hỏi, compiler bắt bạn dùng dấu `?.`, toán tử Elvis `?:` hoặc kiểm tra trước để smart cast.

## Đánh đổi
Code thuần Kotlin giúp bạn tránh được hầu hết lỗi sập app vì `null`. Đổi lại, bạn phải khai báo kiểu kỹ hơn và luôn viết sẵn nhánh xử lý khi thiếu dữ liệu. Lớp bảo vệ này vẫn thủng khi gọi sang code Java qua kiểu `Platform Type` trong [Vì sao Kotlin thay Java trên Android](concept:kotlin-why-over-java). Bù lại, các [Scope functions](concept:kotlin-scope-functions) như `let` đi cùng dấu `?.` giúp bạn xử lý biến có thể `null` rất gọn.

## Nguyên lý gốc
Bắt lỗi ngay lúc đang gõ code để lỗi không lọt sang lúc người dùng đang mở app (Fail-Fast).

## Key insight
> Null safety dời việc bắt lỗi `null` từ lúc chạy về lúc biên dịch nhờ tách kiểu, nhưng vẫn thủng ở biên giới với Java.

## Khuôn mẫu chuẩn
```kotlin
fun formatName(raw: String?): String {
    val clean = raw?.trim()?.takeIf { it.isNotEmpty() }
    return clean ?: "ANONYMOUS"
}

val valid = formatName("  android ")
val empty = formatName(null)
```

## Bẫy & ranh giới
Khi bạn gõ `!!`, bạn tự tắt lớp kiểm tra của compiler và chịu rủi ro sập app. Bẫy hay gặp nhất là gọi hàm Java thiếu `@Nullable`, lúc đó Kotlin nhận về `Platform Type` (`String!`). Nếu bạn gán thẳng kết quả đó vào biến không có dấu hỏi mà Java trả về `null`, app sẽ ném `NullPointerException` ngay. Đừng dùng `!!` trong code chạy thật, hãy dùng `?:` để lấy giá trị mặc định hoặc tự ném lỗi có ghi rõ lý do.

## Câu hỏi thực chiến
**Hỏi:** Viết 100% mã bằng Kotlin thì ứng dụng không bao giờ bị lỗi `NullPointerException` nữa đúng không?
Không phải. Ứng dụng vẫn sập nếu bạn dùng `!!` hoặc thư viện đọc JSON nhét `null` thẳng vào biến không có dấu hỏi. Ngoài ra, các hàm trong Android SDK viết bằng Java thiếu chú thích cũng có thể trả `null` về cho bạn.

**Hỏi:** Ký hiệu dấu chấm than `String!` (Platform Type) từ Java nghĩa là gì?
Đó là kiểu đến từ Java khi hàm bên đó không ghi rõ `@Nullable` hay `@NonNull`. Vì compiler Kotlin không biết biến đó có `null` hay không, nó để bạn tự quyết định. Nếu bạn gán vào biến không dấu hỏi mà hàm Java trả về `null`, ứng dụng sẽ sập `NullPointerException` ngay.

**Hỏi:** Tại sao đã kiểm tra `if (item != null)` rồi mà compiler vẫn không cho smart cast thuộc tính `var`?
Vì biến `var` của class có thể bị gán lại bằng `null` bất cứ lúc nào. Một luồng khác có thể đổi nó thành `null` ngay sau dòng `if` của bạn. Để dùng được, bạn hãy chép nó sang một biến cục bộ `val` hoặc dùng `item?.let` rồi mới xử lý.

**Hỏi:** Có trường hợp nào được phép dùng toán tử ép buộc `!!` không?
Tuỳ vào việc bạn viết code ứng dụng hay viết bài test. Trong unit test với dữ liệu mẫu chắc chắn khác `null`, bạn dùng `!!` để sai là báo đỏ ngay. Còn trong code chạy thật, bạn hãy dùng `?:` hoặc `requireNotNull` kèm câu báo lỗi rõ ràng.

**Hỏi:** Các lệnh `Intrinsics.checkNotNullParameter` do compiler sinh ra có làm chậm app Kotlin không?
Không đáng kể. Đó chỉ là một lệnh so sánh với `null` rất nhẹ ở đầu hàm để chặn `null` từ phía Java truyền sang. Hàm `private` không có lệnh này vì Java không gọi được, còn hàm `public` và `internal` thì có.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách Kotlin dời việc kiểm tra null từ lúc chạy về lúc biên dịch bằng cách tách kiểu T và T?.
- [ ] Nêu được vì sao compiler chèn lệnh Intrinsics ở đầu các hàm không phải private để chặn null từ Java.
- [ ] Chỉ ra được bẫy Platform Type (T!) khi gọi hàm Java thiếu chú thích và tác hại của toán tử !!.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
