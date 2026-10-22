---
id: kotlin-val-vs-const
title: Val vs Const Val
domain: kotlin
tags: [kotlin, bytecode, performance]
collections: []
related: [jvm-virtual-method-dispatch, kotlin-inline-function]
diagrams: [Cơ chế|val-vs-const-bytecode.png|val sinh getter gọi hàm lúc runtime bảo toàn tương thích nhị phân trong khi const val inlining literal trực tiếp vào bytecode]
created: 2026-09-10
---

## Bản 30 giây
`val` là biến chỉ đọc nhưng giá trị được tính toán khi chương trình đang chạy. `const val` là hằng số được trình biên dịch thay thế trực tiếp giá trị vào mã nguồn ngay lúc dịch. Sự khác biệt này quyết định tính an toàn khi nâng cấp thư viện và khả năng dùng trong các chú thích khai báo.

## Cần biết trước

## Vấn đề buộc nó ra đời
Khi thiết kế Kotlin, ngôn ngữ muốn mọi thuộc tính đều có tính bao đóng để đảm bảo an toàn truy cập. Khai báo `val` được dịch thành một biến riêng tư kèm hàm đọc công khai (getter). Cách tiếp cận này giúp mã nguồn dễ mở rộng và kiểm soát tính bất biến.

Tuy nhiên, cấu trúc này vấp phải giới hạn của máy ảo Java với chú thích khai báo (annotation). Annotation đòi hỏi giá trị phải nung sẵn vào bytecode lúc biên dịch. Chúng không thể đợi hàm getter chạy khi ứng dụng khởi động.

Từ khóa `const` ra đời để tháo gỡ bế tắc này. Tuy nhiên, nó tạo ra nguy cơ gãy tương thích nhị phân khi hằng số thay đổi.

## Nó thực sự là cái gì
`val` và `const val` KHÔNG phải hai cách viết hằng số tương đương chỉ khác nhau ở độ ngắn gọn cú pháp. Thực chất, `val` là một thuộc tính (property), mặc định sinh ra phương thức getter để truy cập khi chương trình vận hành. Ngược lại, `const val` là hằng số biên dịch, được compiler thay thế thẳng giá trị thô vào mọi vị trí sử dụng mã nguồn. Lớp khai báo vẫn giữ trường tĩnh công khai cho tương tác Java, nhưng bên gọi Kotlin không hề phát sinh lệnh gọi hàm getter.

## Cơ chế
Khi khai báo `val` thông thường, trình biên dịch tạo một trường `private final` và hàm getter công khai trong tệp bytecode. Khi khai báo `const val`, compiler sinh trường `public static final` nhưng tại mọi vị trí gọi sẽ nhúng thẳng giá trị hằng số. Do nhúng thẳng, `const val` chỉ nhận kiểu nguyên thủy hoặc String và cấm hàm getter. Bên gọi nạp trực tiếp giá trị bằng `ldc` hay `bipush` mà không tham chiếu lớp gốc.

## Đánh đổi
Dùng `const val` giúp triệt tiêu chi phí gọi hàm getter và hỗ trợ annotation. Bạn phải đánh đổi tính linh hoạt: không thể gán biểu thức runtime hay override. Nguy hiểm nhất là đứt gãy tương thích nhị phân khi thư viện đổi giá trị hằng số. Ứng dụng cũ sẽ tiếp tục chạy sai lệch vì vẫn ôm giá trị nhúng cứng.

## Nguyên lý gốc
Đây là sự đánh đổi giữa liên kết tĩnh lúc biên dịch và liên kết động khi chạy. Liên kết tĩnh đem lại tốc độ tối đa nhưng làm chặt chẽ hóa phụ thuộc giữa các mô-đun. Liên kết động qua hàm bọc giúp che giấu chi tiết và bảo toàn ranh giới độc lập.

## Key insight
> `const val` không phải để chạy nhanh hơn `val`, mà là cái bẫy gãy tương thích nhị phân mỗi khi bạn đổi giá trị.

## Ví von
`val` giống như bạn cho người khác số điện thoại của mình để họ gọi hỏi giá vé mỗi khi cần. `const val` giống như in sẵn giá vé 50 nghìn đồng lên từng tờ rơi phát cho mọi người. Khi giá vé tăng lên 100 nghìn, người gọi điện thoại biết ngay giá mới, còn người cầm tờ rơi vẫn đinh ninh giá cũ.

## Khuôn mẫu chuẩn
```kotlin
const val TIMEOUT_MS = 5000L

annotation class HttpConfig(val timeout: Long = TIMEOUT_MS)

class ApiClient {
    val baseUrl: String get() = "https://api.domain.com"
    val timeoutMs: Long = TIMEOUT_MS
}
```

## Bẫy & ranh giới
Bẫy tai hại nhất là đổi `const val` trong thư viện mà không biên dịch lại các mô-đun phụ thuộc. Mô-đun cũ vẫn giữ giá trị nhúng cứng trong bytecode, gây sai lệch dữ liệu mà không báo lỗi. Chống chỉ định dùng `const val` cho các cấu hình có thể thay đổi giữa các phiên bản. Hãy dùng `val` cho biến công khai xuyên mô-đun để bảo vệ tính tương thích nhị phân.

## Câu hỏi thực chiến
**Hỏi:** Dùng val thì giá trị của biến có luôn luôn bất biến không?
Không. `val` chỉ cấm gán lại tham chiếu chứ không cấm sửa đổi trạng thái bên trong đối tượng được trỏ tới. Ngoài ra, một `val` có custom getter hoặc ủy quyền bằng [by lazy](concept:kotlin-lateinit-vs-lazy) hoàn toàn có thể trả về giá trị khác nhau.

**Hỏi:** Cả hai đều không sửa được, khi nào bắt buộc phải dùng const val?
Bắt buộc dùng `const val` khi giá trị cần truyền vào Annotation hoặc làm hằng số switch-case bên Java. Các trường hợp này đòi hỏi giá trị phải được nung cứng trực tiếp vào bytecode ngay lúc biên dịch. Với các trường hợp còn lại, bạn nên ưu tiên dùng `val` để đảm bảo an toàn khi cập nhật thư viện.

**Hỏi:** Vì sao đổi giá trị const val trong một library module lại làm app chạy sai dù không báo lỗi biên dịch?
Vì compiler đã nhúng thẳng giá trị cũ của `const val` vào bytecode của module gọi lúc biên dịch. Nếu bạn nâng cấp file jar thư viện mà không biên dịch lại module gọi, nó vẫn giữ nguyên giá trị số cũ. Hậu quả là hai phần của ứng dụng chạy cùng lúc nhưng lại nhìn thấy hai giá trị cấu hình lệch nhau.

**Hỏi:** Vì sao Kotlin cấm khai báo const val cho đối tượng class thông thường hoặc bên trong hàm?
Vì `const val` đòi hỏi giá trị phải biết trước lúc biên dịch và chỉ nhận kiểu nguyên thuỷ hoặc String. Đối tượng class đòi hỏi cấp phát bộ nhớ trên Heap qua constructor, điều chỉ diễn ra khi ứng dụng đang vận hành. Còn vị trí thì `const` chỉ được khai báo ở cấp file, trong `object` hoặc `companion object`, nơi compiler sinh được trường `static final`.

**Hỏi:** const val có giúp ứng dụng chạy nhanh hơn val đáng kể không?
Không đáng kể. `const val` chỉ bớt được một lệnh gọi hàm getter và một lần đọc trường tĩnh trong bytecode. Máy ảo ART trên Android thường tự động inline các getter đơn giản, nên khác biệt tốc độ thực tế là không đáng kể.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: val thông thường biên dịch thành getter lúc chạy, const val nhúng thẳng giá trị vào bytecode lúc biên dịch.
- [ ] Ràng buộc của const val: chỉ áp dụng cho kiểu nguyên thuỷ hoặc String, nằm ở top-level hoặc trong object.
- [ ] Bẫy tương thích nhị phân: thay đổi const val ở thư viện khiến ứng dụng chạy giá trị cũ nếu không biên dịch lại.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

