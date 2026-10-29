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
Khi bạn khai báo biến chỉ đọc, `val` sẽ gọi hàm `getter` để lấy giá trị lúc ứng dụng đang chạy. Còn `const val` bảo compiler chép thẳng giá trị hằng số vào từng chỗ dùng ngay lúc biên dịch. Khác biệt này quyết định việc dùng nó trong `annotation` và rủi ro lệch dữ liệu khi nâng cấp thư viện.

## Cần biết trước

## Vấn đề buộc nó ra đời
Khi làm Kotlin, nhóm thiết kế muốn mọi thuộc tính trong class đều đi qua hàm đọc để giữ tính đóng gói. Vì vậy, mỗi biến `val` được dịch thành một trường `private` kèm theo hàm `getter` công khai.

Nhưng cách này lại vướng quy định của máy ảo Java khi bạn truyền giá trị vào `annotation`. `annotation` bắt buộc giá trị phải có sẵn ngay trong bytecode lúc biên dịch, chứ không đợi gọi hàm `getter` lúc chạy được.

Từ khóa `const` ra đời để chép thẳng giá trị vào bytecode cho các chỗ như vậy. Đổi lại, nó gây lệch giá trị giữa các module khi thư viện đổi hằng số mà bên gọi chưa biên dịch lại.

## Nó thực sự là cái gì
`val` và `const val` không phải hai cách viết hằng số giống nhau. `val` là một thuộc tính có hàm `getter`, mỗi lần bạn đọc là máy gọi vào hàm đó lúc đang chạy. Ngược lại, `const val` là hằng số lúc biên dịch, được compiler chép thẳng giá trị vào mọi chỗ gọi. Class gốc vẫn giữ một trường `public static final` cho bên Java dùng, nhưng code Kotlin khi đọc sẽ lấy thẳng giá trị đã chép.

## Cơ chế
Khi bạn viết `val` thường, compiler tạo một trường `private final` và một hàm `getter` trong bytecode. Khi bạn viết `const val`, compiler tạo trường `public static final` nhưng tại mọi chỗ gọi nó chép thẳng giá trị vào luôn. Vì phải chép ngay lúc dịch, `const val` chỉ nhận kiểu nguyên thủy hoặc `String` và không cho viết hàm `getter` riêng. Lúc chạy, chỗ gọi nạp thẳng con số hoặc chuỗi bằng lệnh `ldc` hay `bipush` mà không cần hỏi class gốc.

## Đánh đổi
Dùng `const val` giúp bạn truyền được giá trị vào `annotation` và bớt được một lần gọi hàm `getter`. Đổi lại, bạn không thể gán kết quả của một hàm cho nó hay ghi đè nó ở lớp con. Rủi ro lớn nhất là gãy tương thích nhị phân khi một thư viện đổi giá trị `const val`. Nếu module gọi không biên dịch lại, nó vẫn chạy với giá trị cũ đã chép sẵn trong bytecode.

## Nguyên lý gốc
Chép cứng giá trị lúc biên dịch để dùng ngay, hoặc đọc qua hàm lúc chạy để giữ đồng bộ khi đổi.

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
Bẫy nguy hiểm nhất là sửa `const val` trong một module thư viện mà không biên dịch lại các module đang gọi nó. Module gọi vẫn giữ nguyên con số cũ đã chép vào bytecode, làm hai bên chạy lệch nhau mà không báo lỗi. Vì vậy, đừng dùng `const val` cho các thông số có thể đổi qua các phiên bản thư viện. Với biến công khai cho module khác đọc, hãy dùng `val` để giữ tương thích nhị phân.

## Câu hỏi thực chiến
**Hỏi:** Dùng val thì giá trị của biến có luôn luôn bất biến không?
Không. Từ khóa `val` chỉ cấm bạn gán biến đó sang đối tượng khác, chứ không cấm sửa dữ liệu bên trong đối tượng. Ngoài ra, một biến `val` tự viết hàm `get()` hoặc dùng [by lazy](concept:kotlin-lateinit-vs-lazy) vẫn có thể trả về kết quả khác nhau.

**Hỏi:** Cả hai đều không sửa được, khi nào bắt buộc phải dùng const val?
Bạn bắt buộc dùng `const val` khi truyền giá trị vào `annotation` hoặc dùng trong `switch-case` phía Java. Những chỗ này đòi hỏi giá trị phải nằm sẵn trong bytecode ngay lúc biên dịch. Còn ở các chỗ bình thường khác, bạn cứ dùng `val` để sau này đổi giá trị không sợ lỗi.

**Hỏi:** Vì sao đổi giá trị const val trong một library module lại làm app chạy sai dù không báo lỗi biên dịch?
Vì lúc biên dịch lần trước, compiler đã chép thẳng giá trị cũ của `const val` vào bytecode của module gọi. Khi bạn chỉ thay file `.jar` của thư viện mà không biên dịch lại module gọi, chỗ gọi vẫn đọc con số cũ của nó. Thế là thư viện chạy giá trị mới còn ứng dụng lại chạy giá trị cũ.

**Hỏi:** Vì sao Kotlin cấm khai báo const val cho đối tượng class thông thường hoặc bên trong hàm?
Vì `const val` phải chép được ngay vào bytecode lúc dịch nên chỉ nhận kiểu nguyên thủy hoặc `String`. Còn một đối tượng class thông thường phải đợi lúc chạy mới tạo trên Heap qua constructor. Ngoài ra, `const val` chỉ đặt được ở đầu file, trong `object` hoặc `companion object`, nơi sinh được trường `static final`.

**Hỏi:** const val có giúp ứng dụng chạy nhanh hơn val đáng kể không?
Không đáng kể. Dùng `const val` chỉ bớt được một lệnh gọi hàm `getter` và một lần đọc biến tĩnh. Trên Android, máy ảo ART vốn tự chép thẳng các hàm `getter` ngắn vào chỗ gọi nên tốc độ gần như nhau.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được val gọi hàm getter lúc chạy, còn const val chép thẳng giá trị vào bytecode lúc biên dịch.
- [ ] Nêu được điều kiện dùng const val: chỉ nhận kiểu nguyên thủy hoặc String, đặt ở cấp file hoặc trong object.
- [ ] Trình bày được vì sao đổi const val trong thư viện mà không biên dịch lại bên gọi sẽ làm lệch dữ liệu.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

