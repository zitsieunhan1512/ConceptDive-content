---
id: kotlin-why-over-java
title: Vì sao Kotlin thay Java trên Android
domain: kotlin
tags: [kotlin, java, android, compiler, bytecode]
collections: [interview-fundamental]
related: [kotlin-null-safety, jvm-virtual-method-dispatch]
diagrams: [Cơ chế|kotlin-compilation-pipeline.png|Quy trình biên dịch hội tụ về JVM bytecode và hạ cấp sang Dex chạy trên máy ảo ART của Android]
created: 2026-09-11
---

## Bản 30 giây
Android từng bị kẹt ở các bản Java cũ suốt nhiều năm vì vướng máy ảo trên máy cũ và vụ kiện bản quyền. Kotlin giải quyết bằng cách dịch cú pháp mới ra đúng chuẩn bytecode của Java để chạy ngay trên các máy đó. Nhờ gọi qua lại trực tiếp với Java, bạn viết được code mới mà không cần chờ người dùng nâng cấp hệ điều hành.

## Cần biết trước
- kotlin-null-safety

## Vấn đề buộc nó ra đời
Vào thập niên 2010, phía máy chủ đã lên Java 8 với nhiều tính năng mới. Trong khi đó, Android vẫn kẹt ở Java 6 và 7 vì máy ảo Dalvik và vì phải hỗ trợ hàng triệu điện thoại cũ.

Google cũng không thể tự sửa Java vì không nắm bản quyền và đang vướng vụ kiện với Oracle từ năm 2010. Lập trình viên Android rất cần cú pháp mới nhưng bản thân hệ điều hành lại không tự nâng cấp được.

Năm 2011, JetBrains làm ra Kotlin có khả năng dịch ra cùng chuẩn bytecode với Java. Năm 2017, Google chọn Kotlin làm ngôn ngữ chính thức để hiện đại hóa việc viết app mà không đụng vào Android.

## Nó thực sự là cái gì
Kotlin không chỉ là bộ cú pháp viết tắt để bạn gõ ít chữ hơn Java. Thực chất, nó là cách đem các tính năng ngôn ngữ mới chạy trên một máy ảo cũ đã đóng băng. Kotlin không có máy ảo riêng mà chạy nhờ ngay trên máy ảo Java và máy ảo ART của Android. Mọi cú pháp mới bạn viết đều được compiler đổi về các lệnh bytecode quen thuộc trước khi đóng gói.

## Cơ chế
Trình biên dịch `kotlinc` dịch code của bạn ra bytecode JVM, rồi công cụ `d8` đổi tiếp sang file Dex cho Android. Các thông tin của [Null safety](concept:kotlin-null-safety) không có sẵn trong máy ảo nên được compiler ghi vào các annotation. Để tự bảo vệ khi gọi qua lại với Java, compiler chèn thêm lệnh kiểm tra `Intrinsics` ở đầu các hàm không phải `private`. Nhờ dùng chung bytecode, mọi thứ như [Val vs Const Val](concept:kotlin-val-vs-const) đều biến thành trường `static` hoặc hàm `getter` giống hệt Java.

## Đánh đổi
Bạn dùng được ngay các tính năng mới trên cả những máy Android đời cũ mà không cần chờ cập nhật hệ thống. Đổi lại, file APK phải gánh thêm thư viện `kotlin-stdlib` và thời gian chờ `kotlinc` biên dịch cũng lâu hơn `javac`. Ngoài ra, lớp kiểm tra rỗng vẫn có thể bị lọt khi bạn nhận dữ liệu từ hàm Java không gắn chú thích `@Nullable`.

## Nguyên lý gốc
Tách rời ngôn ngữ khỏi máy ảo bên dưới (Language vs Runtime Decoupling). Khi không sửa được máy ảo trên điện thoại, ta nâng cấp ở bước biên dịch.

## Key insight
> Kotlin thắng không vì cú pháp ngắn hơn, mà là cách duy nhất đưa ngôn ngữ hiện đại lên runtime Android bị đóng băng.

## Ví von
Lắp bộ chuyển đổi điện thông minh cho một mạng lưới dây dẫn cũ kỹ trong nhà. Thay vì đập tường làm lại dây cáp, bạn gắn đầu cắm mới để dùng thiết bị hiện đại ngay trên hạ tầng cũ.

## Khuôn mẫu chuẩn
```kotlin
class UserFormatter {
    // Tham số mặc định là khái niệm chỉ Kotlin hiểu, bytecode không có.
    // @JvmOverloads bắt compiler sinh thêm bản rút gọn cho Java gọi được.
    @JvmOverloads
    fun format(name: String, role: String = "USER"): String {
        return "${name.trim()}:$role"
    }
}
val f = UserFormatter()
val short = f.format("alice")
val full = f.format("bob", "ADMIN")
```

## Bẫy & ranh giới
Nhiều bạn tưởng Kotlin tự chạy nhanh hơn Java, nên dùng lambda vô tội vạ trong vòng lặp làm tốn công dọn rác. Bẫy thứ hai là gọi hàm Java không có chú thích rỗng, nhận về kiểu `Platform Type` và làm app sập lúc chạy. Bạn cũng đừng bấm nút đổi tự động cả file Java sang Kotlin rồi để nguyên không kiểm tra lại các chỗ nhận `null`. Triệu chứng thường gặp là thời gian build dự án dài ra và lỗi `NullPointerException` nổ ngay ở chỗ Kotlin gọi sang Java.

## Câu hỏi thực chiến
**Hỏi:** Viết bằng Kotlin thì ứng dụng Android có tự động chạy nhanh hơn Java không?
Không. Cả hai đều dịch ra cùng một chuẩn bytecode và chạy chung trên máy ảo ART của Android. Nếu truyền lambda vào hàm thiếu `inline`, Kotlin còn tạo thêm đối tượng trên Heap làm tốn công dọn rác.

**Hỏi:** Bẫy nguy hiểm nhất khi gọi mã Java từ Kotlin trong cùng một dự án là gì?
Đó là kiểu `Platform Type` xuất hiện khi hàm bên Java không ghi rõ `@Nullable` hay `@NonNull`. Lúc này compiler Kotlin không biết kết quả có thể `null` hay không nên bỏ qua không bắt lỗi lúc dịch. Nếu bạn gán thẳng vào biến không dấu hỏi mà hàm Java trả về `null`, ứng dụng sẽ sập `NullPointerException` ngay.

**Hỏi:** Dự án Java cũ có nhất thiết phải viết lại toàn bộ sang Kotlin không?
Không cần. Vì dùng chung bytecode, các class Kotlin và Java gọi qua lại với nhau trực tiếp trong cùng một module. Cách làm an toàn là giữ nguyên code Java đang chạy ổn định, chỉ viết Kotlin cho màn hình mới hoặc khi cần sửa lại.

**Hỏi:** Kotlin bảo vệ Null safety thế nào khi một lớp Java gọi vào hàm của Kotlin?
Ở đầu các hàm không phải `private` nhận tham số không-null, compiler tự chèn thêm lệnh `Intrinsics.checkNotNullParameter`. Nếu bên Java truyền `null` vào, hàm sẽ ném lỗi `NullPointerException` ngay ở dòng đầu tiên và chỉ rõ tên tham số bị `null`. Nhờ vậy, giá trị `null` không thể trôi sâu vào các hàm bên trong rồi mới gây sập ở một chỗ khó tìm.

**Hỏi:** Đổi từ Java sang Kotlin thì dự án phải trả giá bằng chi phí gì lớn nhất?
Thường là thời gian biên dịch: `kotlinc` phải suy diễn kiểu nhiều hơn `javac` nên build chậm hơn. Thư viện `kotlin-stdlib` làm APK nặng thêm một chút, nhưng công cụ R8 sẽ cắt bỏ phần lớn những hàm không dùng. Chi phí khó thấy hơn là giai đoạn dài dự án trộn cả hai ngôn ngữ, người mới vào phải đọc hiểu cả hai.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được trong hai câu: Android kẹt ở Java cũ vì Dalvik và vụ kiện Oracle; Kotlin thắng nhờ dịch ra cùng bytecode chạy trên máy cũ.
- [ ] Nêu được Kotlin không có máy ảo riêng, các tính năng như [Null safety](concept:kotlin-null-safety) đều xử lý lúc biên dịch và chèn Intrinsics ở đầu hàm.
- [ ] Trình bày được các đánh đổi: thời gian build của kotlinc lâu hơn, thêm kotlin-stdlib và bẫy Platform Type khi gọi hàm Java.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
