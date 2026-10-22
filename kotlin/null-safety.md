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
Truy cập dữ liệu null là nguyên nhân gây sập ứng dụng phổ biến nhất. Kotlin phân tách rõ kiểu có thể null và không thể null ngay trong hệ thống kiểu. Trình biên dịch bắt buộc xử lý giá trị null từ sớm, bảo vệ chương trình trước khi mã thực thi.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1965, Tony Hoare tạo ra con trỏ null trên ngôn ngữ ALGOL W. Ông thiết kế null đơn giản vì nó quá dễ cài đặt vào compiler thời đó.

Thiết kế này khiến lỗi `NullPointerException` bùng nổ, làm bốc hơi hàng triệu giờ làm việc của kỹ sư toàn cầu.

Trong Java và Android, `NullPointerException` luôn là nguyên nhân số một gây sập ứng dụng. Khi tạo Kotlin, JetBrains đặt mục tiêu dẹp bỏ triệt để lỗi này ngay từ khâu biên dịch.

## Nó thực sự là cái gì
Null safety KHÔNG phải là việc xóa sổ giá trị null khỏi bộ nhớ. Thực chất, nó là cơ chế chuyển việc kiểm tra rỗng từ thời điểm chạy (runtime) về thời điểm biên dịch (compile-time).

## Cơ chế
Kotlin chia kiểu dữ liệu thành hai nhánh: không thể null (`String`) và có thể null (`String?`). Trong mã thuần Kotlin, compiler kiểm soát tính an toàn hoàn toàn tại compile-time mà không tốn chi phí runtime. Compiler chèn các lệnh `Intrinsics` ở đầu hàm không riêng tư để phòng vệ khi gọi chéo với Java. Với kiểu có thể null, compiler ép lập trình viên dùng toán tử an toàn `?.`, Elvis `?:` hoặc smart cast.

## Đánh đổi
Mã nguồn thuần Kotlin gần như triệt tiêu hoàn toàn nguy cơ sập app do null. Đổi lại, bạn phải viết cú pháp chặt chẽ hơn và liên tục xử lý giá trị dự phòng. Tấm khiên này bị thủng khi giao tiếp Java qua kiểu nền tảng trong [Vì sao Kotlin thay Java trên Android](concept:kotlin-why-over-java). Bù lại, các [Scope functions](concept:kotlin-scope-functions) như let kết hợp dấu hỏi giúp biến đổi dữ liệu rỗng vô cùng an toàn và tiện lợi.

## Nguyên lý gốc
Nguyên lý bắt lỗi sớm (Fail-Fast): chuyển việc bắt lỗi từ lúc người dùng chạy app về lúc lập trình viên đang gõ phím.

## Key insight
> Null safety chuyển việc bắt lỗi rỗng từ runtime về compile-time nhờ hệ thống kiểu phân nhánh, nhưng sẽ thủng khi gọi sang Java.

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
Toán tử `!!` là hành vi tự tay vô hiệu hóa tấm khiên bảo vệ của compiler. Bẫy nguy hiểm nhất là gọi hàm Java trả về Platform Type (`String!`) không có chú thích `@Nullable`. Nếu bạn gán thẳng vào biến không dấu hỏi, app sẽ sập `NullPointerException` tức thì lúc chạy. Chống chỉ định dùng `!!` trong mã nguồn chính; hãy dùng Elvis `?:` hoặc ném ngoại lệ tường minh với thông điệp rõ ràng.

## Câu hỏi thực chiến
**Hỏi:** Viết 100% mã bằng Kotlin thì ứng dụng không bao giờ bị lỗi `NullPointerException` nữa đúng không?
Không phải. Ứng dụng vẫn sập nếu bạn lạm dụng toán tử `!!` hoặc thư viện parse JSON đổ `null` vào biến không-null. Ngoài ra, các thư viện Android SDK viết bằng Java thiếu chú thích cũng có thể âm thầm tuồn `null` vào.

**Hỏi:** Ký hiệu dấu chấm than `String!` (Platform Type) từ Java nghĩa là gì?
Đó là kiểu nền tảng do Kotlin không rõ biến phía Java có thể null hay không. Khi mã Java thiếu chú thích `@Nullable` hay `@NonNull`, compiler thả lỏng kiểm soát cho bạn tự chọn kiểu. Nếu bạn chọn kiểu không-null mà Java trả về `null`, app sẽ sập `NullPointerException` lập tức.

**Hỏi:** Tại sao đã kiểm tra `if (item != null)` rồi mà compiler vẫn không cho smart cast thuộc tính `var`?
Vì thuộc tính `var` có thể bị sửa đổi giá trị bất kỳ lúc nào. Một luồng khác hoặc hàm ghi đè có thể đổi giá trị thành `null` ngay giữa lúc kiểm tra và sử dụng. Để sửa lỗi, bạn hãy gán giá trị đó vào một biến cục bộ `val` trước khi kiểm tra rỗng.

**Hỏi:** Có trường hợp nào được phép dùng toán tử ép buộc `!!` không?
Tuỳ vào việc bạn viết mã chạy app hay viết kiểm thử. Trong unit test khi dữ liệu giả đã dàn dựng chắc chắn có giá trị, dùng `!!` giúp code gọn gàng. Trong mã sản phẩm chính, hãy luôn dùng toán tử Elvis `?:` hoặc ném ngoại lệ có thông điệp rõ ràng.

**Hỏi:** Các lệnh `Intrinsics.checkNotNullParameter` do compiler sinh ra có làm chậm app Kotlin không?
Không đáng kể. Đây chỉ là một phép so sánh địa chỉ với `null` siêu nhanh ở đầu hàm để chặn mã bẩn từ Java. Hàm private không bị chèn vì mã Java không gọi tới được, còn hàm `internal` vẫn bị chèn như hàm public.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: chuyển kiểm tra null từ thời điểm chạy về thời điểm biên dịch qua hệ thống kiểu.
- [ ] Compiler chèn kiểm tra Intrinsics ở đầu hàm không riêng tư phòng vệ cho Java interop.
- [ ] Bẫy Platform Type (T!) từ Java và bẫy lạm dụng toán tử ép buộc hai chấm than (!!).
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
