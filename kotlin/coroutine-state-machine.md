---
id: kotlin-coroutine-state-machine
title: Continuation & Compiler State Machine
domain: kotlin
tags: [kotlin, coroutines, compiler, bytecode]
collections: []
related: [kotlin-coroutines, android-activity-lifecycle, kotlin-inline-function]
sources:
  - Kotlin Specification: Asynchronous programming with coroutines | https://kotlinlang.org/spec/asynchronous-programming-with-coroutines.html
  - Kotlin Coroutines KEEP design proposal | https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md
  - Continuation-passing style — Wikipedia | https://en.wikipedia.org/wiki/Continuation-passing_style
diagrams: [Cơ chế|coroutine-state-machine.png|Trình biên dịch sinh switch-case theo biến label, nhả luồng khi gặp COROUTINE_SUSPENDED và phục hồi qua resumeWith]
created: 2026-09-08
---

## Bản 30 giây
Khi bạn gọi API trên Android, hàm tạm dừng chờ mạng mà không giam cứng luồng giao diện. Trình biên dịch cắt hàm tại mỗi điểm dừng thành một máy trạng thái đánh số thứ tự bằng biến `label`. Biến cục bộ được cất vào đối tượng `Continuation` trên Heap để khi mạng trả về thì lấy ra chạy tiếp.

## Cần biết trước
- kotlin-coroutines

## Vấn đề buộc nó ra đời
Năm 1975, Gerald Jay Sussman và Guy L. Steele dùng kiểu viết Continuation-Passing Style (CPS) trong ngôn ngữ Scheme. Mỗi hàm nhận thêm một tham số là phần việc còn lại và gọi nó khi chạy xong.

Đến thập niên 2010, code Android và JavaScript gọi mạng bằng callback lồng nhau nên mỗi bước lại lùi sâu thêm một tầng. Khối `try-catch` bọc ngoài không bắt được lỗi ném trong callback chạy sau, còn vòng lặp bất đồng bộ phải viết thành đệ quy. Callback ẩn danh giữ tham chiếu tới Activity đã đóng còn gây rò rỉ bộ nhớ.

Roman Elizarov dẫn dắt thiết kế Coroutines dựa trên CPS, đưa vào Kotlin 1.1 (2017) bản thử nghiệm và ổn định ở Kotlin 1.3 (2018). Bạn viết code tuần tự từ trên xuống, còn trình biên dịch tự sinh máy trạng thái quản lý việc dừng và chạy tiếp.

## Nó thực sự là cái gì
Nhiều người nghĩ hàm `suspend` có phép đóng băng luồng, nhưng thật ra trình biên dịch viết lại hàm của bạn. Nó gom thân hàm vào khối `when (label)` bên trong một lớp con của `ContinuationImpl`. Mỗi biến cục bộ cần dùng qua điểm dừng được cất thành trường của đối tượng trên Heap thay vì nằm ở ngăn xếp. Mỗi điểm gọi `suspend` được đánh một số `label` để hàm biết nhảy vào nhánh nào khi chạy lại.

## Cơ chế
Khi biên dịch, hàm của bạn nhận thêm tham số `completion: Continuation<Any?>` ở cuối và đổi kiểu trả về sang `Any?`. Lúc mới gọi, hàm kiểm tra tham số này để dùng lại đối tượng máy trạng thái cũ hoặc tạo bản mới trên Heap. Trước mỗi điểm dừng, hàm tăng số `label` lên rồi trả về hằng số `COROUTINE_SUSPENDED` để nhả luồng ngay. Khi có kết quả mạng, hệ thống gọi `continuation.resumeWith()` để chạy lại hàm và nhảy tới đúng nhánh `label` kế tiếp.

## Đánh đổi
Khi gọi 3 API nối tiếp nhau, bạn vẫn dùng vòng lặp và `try-catch` tự nhiên như code đồng bộ. Đổi lại, mỗi lần gọi hàm có điểm dừng sẽ cấp thêm đối tượng `Continuation` trên bộ nhớ Heap. Các biến giữ qua điểm dừng cũng phải nằm trên Heap nên làm bộ dọn rác tốn thêm công thu hồi.

## Nguyên lý gốc
Khi hàm đang chạy giữa chừng, ta biến thông tin đang chạy tới đâu thành một đối tượng dữ liệu (Reification of Control Flow). Nhờ nằm trong đối tượng, vị trí dừng và biến dở dang có thể cất vào bộ nhớ rồi lấy ra chạy tiếp.

## Key insight
> `suspend` của Kotlin không nhờ máy ảo tạm dừng, mà compiler tự biến hàm thành đối tượng `Continuation` nhớ nhãn `label`.

## Ví von
Hãy tưởng tượng bạn đang đọc sách tới dòng thứ 5 thì phải trả sách về kệ cho người khác mượn. Bạn kẹp một mảnh giấy ghi số trang và dòng đang đọc dở vào cuốn sách đó. Lúc rảnh, bạn lấy sách ra, nhìn mảnh giấy để mở đúng dòng thứ 5 và đọc tiếp.

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
Mỗi lần gọi hàm `suspend` có điểm dừng là thêm một đối tượng `Continuation` trên Heap, gọi hàng triệu lần thì tốn công dọn rác. `suspend` cũng không tự đưa việc sang luồng khác: vòng lặp tính toán nặng chạy trên Main vẫn chặn giao diện, phải bọc `withContext(Dispatchers.Default)`. Với tính toán thuần trong vòng lặp lớn, hãy dùng hàm thường hoặc [inline function](concept:kotlin-inline-function) để biến nằm trên ngăn xếp. Triệu chứng là giao diện giật hoặc ANR dù code đã là coroutine, và stack trace khi lỗi thiếu các hàm gọi trước điểm dừng.

## Câu hỏi thực chiến
**Hỏi:** Cứ gọi hàm `suspend` là hàm đó chắc chắn sẽ nhả luồng và tạm dừng đúng không?
Không phải. Khi bạn gọi hàm lấy ảnh đã có sẵn trong bộ nhớ đệm, hàm trả kết quả ngay mà không nhả luồng. Hàm chỉ thực sự tạm dừng và nhả luồng khi việc bên trong chưa xong và trả về `COROUTINE_SUSPENDED`.

**Hỏi:** Biến cục bộ nào bên trong hàm `suspend` sẽ bị đẩy lên Heap?
Chỉ những biến bạn tạo trước điểm dừng và còn đọc lại sau điểm dừng đó mới được cất lên Heap. Trình biên dịch dò từng biến và chỉ tạo trường trong đối tượng `Continuation` cho các biến cần giữ qua lúc nghỉ. Biến nào dùng xong trước điểm dừng thì vẫn nằm trên ngăn xếp luồng như hàm thường.

**Hỏi:** Khi gọi hàm `suspend` từ mã Java, ta thấy chữ ký hàm thay đổi thế nào?
Bạn sẽ thấy hàm có thêm một tham số `Continuation` ở cuối cùng và kiểu trả về đổi thành `Object`. Vì Java không có từ khóa `suspend`, bạn phải tự truyền một đối tượng nhận kết quả như kiểu viết callback.

**Hỏi:** Thêm từ khóa `suspend` vào hàm bình thường không có điểm ngắt thì có hại gì không?
Không hại về tốc độ chạy. Thân hàm không có điểm dừng thì trình biên dịch không tạo máy trạng thái, chỉ thêm tham số `Continuation` vào cuối. Cái bất tiện là mọi chỗ gọi hàm này đều bị bắt phải nằm trong coroutine, và IDE sẽ báo chữ `suspend` bị thừa.

**Hỏi:** Làm sao máy trạng thái biết cần chạy tiếp đoạn code nào sau khi thức dậy?
Nó dựa vào số nguyên lưu trong biến `label` của đối tượng `Continuation`. Ngay trước khi gọi API và tạm nghỉ, hàm gán `label` bằng số thứ tự của đoạn code ngay sau đó. Khi mạng trả kết quả và gọi `resumeWith()`, khối `when (label)` đọc số này để nhảy thẳng tới đoạn code kế tiếp.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách trình biên dịch thêm tham số Continuation vào cuối hàm và đổi kiểu trả về thành Any?.
- [ ] Trình bày được cách biến label và khối when giúp hàm nhảy tới đúng chỗ vừa dừng sau khi gọi resumeWith.
- [ ] Nêu được chi phí tạo đối tượng Continuation trên Heap khi gọi hàm có điểm dừng trong vòng lặp lớn.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
