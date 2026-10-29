---
id: kotlin-structured-concurrency
title: Structured Concurrency
domain: kotlin
tags: [kotlin, coroutines, structured-concurrency, job, supervisor-job]
collections: [interview-fundamental]
related: [kotlin-coroutines, kotlin-coroutine-context-and-dispatchers, thread]
diagrams: [Cơ chế|job-tree.png|Khi async getNews ném lỗi, coroutineScope nhận lỗi và huỷ async getUser; huỷ viewModelScope thì tín hiệu huỷ lan xuống cả cây.]
created: 2026-10-01
---

## Bản 30 giây
Khi gọi 3 API cùng lúc mà một cái bị lỗi, các coroutine còn lại dễ chạy thừa nếu thiếu quản lý. Đồng quy có cấu trúc (*structured concurrency*) gắn vòng đời của mọi coroutine con vào một `Job` cha theo hình cây. Khi `Job` cha bị hủy hoặc một nhánh con ném lỗi, toàn bộ các nhánh liên quan đều tự dừng lại.

## Cần biết trước
- kotlin-coroutines
- kotlin-coroutine-context-and-dispatchers
- thread

## Vấn đề buộc nó ra đời
Năm 2018, Nathaniel J. Smith viết bài luận ví lệnh bắn luồng tự do giống như lệnh nhảy `goto` dễ gây rối của thập niên 1960. Một việc chạy ngầm không gắn vào đâu thường bị bỏ quên khi hàm gọi nó gặp lỗi.

Trên điện thoại, người dùng đã đóng màn hình nhưng các luồng tải mạng bên dưới vẫn cứ chạy tiếp. Ứng dụng vừa tốn pin, rò rỉ bộ nhớ, lại vừa dễ sập khi cố vẽ kết quả lên màn hình đã đóng.

Tháng 9 năm 2018, Roman Elizarov đưa mô hình này vào `kotlinx.coroutines 0.26.0`. Phiên bản này đánh dấu lỗi thời (*deprecated*) các hàm khởi chạy tự do và đưa vào `CoroutineScope` cùng `GlobalScope`.

## Nó thực sự là cái gì
Đồng quy có cấu trúc không phải một loại luồng mới hay mẹo làm CPU chạy nhanh hơn. Nó là quy tắc quản lý vòng đời bằng cách xếp các đối tượng `Job` thành một cây cha con.

Khối lệnh cha chỉ xong việc khi mọi coroutine con bên trong nó đã chạy xong.

Nếu nhánh con ném lỗi chưa bắt, nó sẽ hủy cả cha lẫn các nhánh anh em, trừ khi dùng [SupervisorJob](concept:kotlin-coroutine-context-and-dispatchers) hoặc `supervisorScope`.

## Cơ chế
Mỗi khi bạn gọi `launch` hay `async` từ một `CoroutineScope`, hàm đó tạo ra một đối tượng `Job` mới. `Job` mới này tự gắn vào danh sách con của `Job` cha để tạo thành một cây phân cấp.

Khi bạn hủy `Job` cha, tín hiệu hủy đi từ trên xuống dưới tới mọi `Job` con.

Ngược lại, khi một `Job` con gặp lỗi không bắt được, nó báo ngược lên `Job` cha để hủy hết các nhánh còn lại.

## Đánh đổi
Cách tổ chức hình cây giúp bạn không bao giờ bỏ sót một coroutine nào chạy ngầm ngoài ý muốn. Đổi lại, chỉ cần một việc phụ bị lỗi mà quên bắt lại thì cả công việc chính cũng bị hủy theo.

Ứng dụng cũng phải giữ danh sách liên kết các đối tượng `Job` trên bộ nhớ Heap trong lúc chúng đang chạy.

## Nguyên lý gốc
Một hàm chỉ được phép kết thúc khi mọi việc con mà nó đẻ ra bên trong đã dừng hẳn.

## Key insight
> Đồng quy có cấu trúc gom các coroutine rời rạc vào một cây `Job` cha con để tự chờ nhau và tự hủy cùng nhau.

## Ví von
Một người quản lý dự án giao việc cho ba nhân viên cấp dưới. Người quản lý chỉ được về khi nhân viên làm xong, và khi dự án bị hủy thì cả nhóm ngừng làm việc.

## Khuôn mẫu chuẩn
```kotlin
suspend fun fetchDashboard(): Dashboard = coroutineScope {
    val userDeferred = async { api.getUser() }
    val newsDeferred = async { api.getNews() }
    Dashboard(userDeferred.await(), newsDeferred.await())
}
```

## Bẫy & ranh giới
Nếu bạn gọi `GlobalScope.launch`, coroutine đó nằm ngoài cây `Job` của màn hình nên dễ chạy mãi và gây rò rỉ bộ nhớ. Cũng đừng truyền thẳng `Job()` mới vào trong `launch(Job())`, vì làm vậy là tự cắt đứt dây nối với `Job` cha hiện tại. Nếu bạn muốn một nhánh con lỗi mà các nhánh anh em vẫn chạy tiếp, hãy bọc bằng `supervisorScope` hoặc dùng `SupervisorJob`. Triệu chứng khi làm sai là bị [Memory leak](concept:android-memory-leak), hoặc một API phụ bị lỗi làm hủy luôn cả màn hình chính.

## Câu hỏi thực chiến
**Hỏi:** Vì sao `launch(Job())` bên trong `viewModelScope` vẫn chạy tiếp sau khi màn hình đã đóng?
Vì đối tượng `Job()` bạn mới tạo nhảy vào thay thế vị trí cha và cắt đứt liên kết với `Job` của `viewModelScope`. Khi màn hình đóng, lệnh hủy từ `viewModelScope` không còn đường nào để truyền xuống coroutine đó nữa. Vì vậy, bạn đừng tự truyền `Job()` mới vào hàm `launch`.

**Hỏi:** Khối `runBlocking` hoặc `coroutineScope` có chờ các coroutine con không gắn `await` kết thúc không?
Có. Khối cha luôn tự tạm dừng và đợi mọi coroutine con bên trong chạy xong rồi mới đi tiếp. Bạn không cần gọi `join` cho từng nhánh con, trừ khi muốn nhánh này phải xong trước nhánh kia.

**Hỏi:** Việc truyền `CoroutineScope(Job())` vào ViewModel có gây rò rỉ bộ nhớ không?
Rất dễ rò rỉ. Nếu bạn quên tự gọi `cancel()` khi [ViewModel](concept:android-viewmodel) bị đóng, các coroutine bên trong sẽ cứ thế chạy tiếp. Tốt nhất bạn hãy dùng sẵn `viewModelScope` để nó tự hủy công việc khi người dùng thoát hẳn màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách các đối tượng Job xếp thành cây cha con và quy tắc cha luôn chờ mọi con chạy xong.
- [ ] Trình bày được hai chiều truyền tín hiệu hủy: từ cha xuống con khi bị cancel, và từ con lên cha khi ném lỗi.
- [ ] Chỉ ra được bẫy truyền Job() mới vào launch làm đứt dây nối với viewModelScope và cách dùng supervisorScope.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

