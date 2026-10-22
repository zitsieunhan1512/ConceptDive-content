---
id: kotlin-coroutine-state-machine
title: Continuation & Compiler State Machine
domain: kotlin
tags: [kotlin, coroutines, compiler, bytecode]
collections: []
related: [kotlin-coroutines, android-activity-lifecycle, kotlin-inline-function]
diagrams: [Cơ chế|coroutine-state-machine.png|Trình biên dịch sinh switch-case theo biến label, nhả luồng khi gặp COROUTINE_SUSPENDED và phục hồi qua resumeWith]
created: 2026-09-08
---

## Bản 30 giây
Khi tạm dừng hàm, máy tính cần nhớ vị trí dừng và dữ liệu dở dang mà không được khóa cứng luồng. Trình biên dịch giải quyết việc này bằng cách cắt nhỏ hàm thành một máy trạng thái có đánh số thứ tự. Dữ liệu được cất vào một gói nhớ trên Heap để khi xong việc có thể gọi hàm chạy tiếp.

## Cần biết trước
- kotlin-coroutines

## Vấn đề buộc nó ra đời
Năm 1975, Gerald Jay Sussman và Guy L. Steele định nghĩa mô hình Continuation-Passing Style (CPS) trong ngôn ngữ Scheme.

Thời điểm đó, chuỗi hàm gọi lại (callback hell) xé nát mã nguồn tuần tự thành cây lồng nhau phân mảnh. Cấu trúc lồng nhau làm tê liệt khối `try-catch`, vỡ luồng xử lý vòng lặp và gây rò rỉ bộ nhớ luồng.

Năm 2017, Roman Elizarov cùng đội ngũ JetBrains ứng dụng CPS kết hợp máy trạng thái vào Kotlin Coroutines. Giải pháp này giúp viết code bất đồng bộ tuần tự mà máy trạng thái ngầm tự quản lý tiến trình bên dưới.

## Nó thực sự là cái gì
Hàm suspend KHÔNG PHẢI là hàm ma thuật tự biết đóng băng luồng đang chạy. Thực chất, trình biên dịch chia nhỏ hàm tại các điểm ngắt thành khối lệnh `when (label)` bên trong một lớp con `ContinuationImpl`. Các biến cục bộ sống qua điểm ngắt được chuyển hóa thành trường trên đối tượng Heap thay vì nằm ở ngăn xếp luồng. Điểm gọi suspend được cấp một số nhãn để máy trạng thái biết chính xác nơi cần nhảy vào sau khi được đánh thức.

## Cơ chế
Trình biên dịch đổi chữ ký hàm bằng cách thêm tham số `completion: Continuation<Any?>` vào cuối và đổi kiểu trả về sang `Any?`. Hàm kiểm tra đối tượng truyền vào để tái sử dụng máy trạng thái hoặc cấp phát bản mới trên Heap. Trước mỗi điểm ngắt, biến `label` tăng lên và hàm con trả về hằng số `COROUTINE_SUSPENDED` để nhả luồng lập tức. Khi dữ liệu sẵn sàng, hệ thống gọi `continuation.resumeWith()` để tái kích hoạt máy trạng thái nhảy tới nhãn tiếp theo.

## Đánh đổi
Bạn viết được mã bất đồng bộ tự nhiên như mã đồng bộ với đầy đủ cấu trúc vòng lặp và bắt lỗi ngoại lệ. Đổi lại, mỗi coroutine tạo thêm áp lực cấp phát đối tượng máy trạng thái `Continuation` lên bộ nhớ Heap. Lưu biến sống qua điểm ngắt trên Heap làm tăng áp lực dọn rác so với lưu trên ngăn xếp CPU thông thường.

## Nguyên lý gốc
Thực thể hóa dòng điều khiển (Reification of Control Flow). Biến luồng thực thi của chương trình thành đối tượng dữ liệu cụ thể để có thể lưu trữ, truyền đi và phục hồi.

## Key insight
> Máy ảo không có cơ chế tạm dừng: `suspend` thực chất là compiler biến hàm thành đối tượng `Continuation` mang nhãn trạng thái.

## Ví von
Giống như chiếc kẹp đánh dấu trang sách khi bạn đang đọc dở. Bạn ghi chú số dòng rồi trả sách về kệ để người khác có thể mượn đọc tiếp. Khi rảnh rỗi, bạn lấy sách ra và mở đúng số trang ghi trên thẻ để tiếp tục câu chuyện.

## Khuôn mẫu chuẩn
```kotlin
// Mô phỏng bytecode do Kotlin Compiler sinh ra cho hàm suspend
fun fetchData(cont: Continuation<Any?>): Any? {
    val sm = cont as? FetchDataSM ?: FetchDataSM(cont)
    when (sm.label) {
        0 -> {
            sm.label = 1
            if (apiCall(sm) == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        1 -> return sm.result
    }
    return Unit
}
```

## Bẫy & ranh giới
Bẫy nguy hiểm là khai báo từ khóa `suspend` cho các tác vụ tính toán CPU nặng nề trong vòng lặp hàng triệu phần tử. Cấp phát hàng loạt đối tượng Continuation trên Heap làm máy ảo nghẽn bộ nhớ và gây giật khung hình giao diện nghiêm trọng. Chống chỉ định dùng hàm suspend khi không có nhu cầu giải phóng luồng hoặc không gọi tới thư viện bất đồng bộ con. Với tính toán dữ liệu thuần túy, hãy dùng hàm thường kết hợp [inline function](concept:kotlin-inline-function) để giữ trọn vẹn dữ liệu trên ngăn xếp.

## Câu hỏi thực chiến
**Hỏi:** Cứ gọi hàm `suspend` là hàm đó chắc chắn sẽ nhả luồng và tạm dừng đúng không?
Không phải. Nếu dữ liệu đã có sẵn trong bộ nhớ đệm, hàm suspend sẽ trả ngay kết quả mà không hề nhả luồng. Hàm chỉ thực sự nhả luồng khi tác vụ con chưa hoàn thành và trả về giá trị `COROUTINE_SUSPENDED`.

**Hỏi:** Biến cục bộ nào bên trong hàm `suspend` sẽ bị đẩy lên Heap?
Chỉ những biến còn sống qua điểm tạm dừng mới bị lưu vào Heap. Compiler phân tích vòng đời và chỉ tạo trường trong đối tượng `Continuation` cho biến cần dùng tiếp sau đó. Những biến dùng xong trước điểm ngắt vẫn được giải phóng trên ngăn xếp luồng như hàm bình thường.

**Hỏi:** Khi gọi hàm `suspend` từ mã Java, ta thấy chữ ký hàm thay đổi thế nào?
Hàm xuất hiện thêm một tham số `Continuation` ở cuối và đổi kiểu trả về thành `Object`. Java không có cú pháp coroutine nên bắt buộc phải truyền một đối tượng callback để nhận kết quả khi xong.

**Hỏi:** Thêm từ khóa `suspend` vào hàm bình thường không có điểm ngắt thì có hại gì không?
Có, nhưng không phải hại hiệu năng. Không có điểm dừng thì compiler không sinh máy trạng thái, chỉ thêm tham số `Continuation` vào chữ ký hàm. Cái hại là mọi nơi gọi bị ép nằm trong coroutine, và IDE sẽ cảnh báo `suspend` thừa.

**Hỏi:** Làm sao máy trạng thái biết cần chạy tiếp đoạn code nào sau khi thức dậy?
Nhờ vào biến `label` đóng vai trò con trỏ trạng thái. Trước mỗi lần tạm dừng, hàm tăng giá trị `label` tương ứng với vị trí đoạn mã kế tiếp. Khi hàm được gọi lại qua `resumeWith()`, khối lệnh `when (label)` sẽ nhảy ngay đến điểm dừng đó để chạy tiếp.

## Ý bắt buộc khi giải thích lại
- [ ] Trình bày được cách compiler chèn tham số Continuation và đổi kiểu trả về thành Any?.
- [ ] Giải thích được vai trò của biến label và khối lệnh when trong việc khôi phục điểm ngắt.
- [ ] Phân tích được áp lực cấp phát bộ nhớ Heap của đối tượng Continuation khi lạm dụng trong vòng lặp.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
