---
id: architecture-state-vs-event
title: State vs Event
domain: architecture
tags: [architecture, state-machine, event-driven, distributed-systems]
collections: []
related: [android-state-holders, android-viewmodel, architecture-mvvm-vs-mvi, kotlin-stateflow-vs-sharedflow]
created: 2026-09-09
sources:
  - UI events — Android App Architecture | https://developer.android.com/topic/architecture/ui-layer/events
  - Two Generals Problem — Wikipedia | https://en.wikipedia.org/wiki/Two_Generals%27_Problem
  - Idempotence in computer science — Wikipedia | https://en.wikipedia.org/wiki/Idempotence
diagrams: [Cơ chế|state-vs-event-lifecycle.png|State có tính lũy đẳng đọc lại vô hại, Event chỉ kích hoạt tác vụ phụ một lần và phải xóa null sau khi dùng]
---

## Bản 30 giây
Khi nhầm giữa sự kiện một lần và trạng thái đứng yên, ứng dụng dễ hiện lặp thông báo hoặc trừ tiền hai lần. `State` là tình trạng hiện tại, màn hình đọc lại bao nhiêu lần kết quả vẫn giữ nguyên. Còn `Event` là việc vừa xảy ra, chỉ được chạy đúng một lần rồi phải khép lại.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1975, Akkoyunlu, Ekanadham và Huber chứng minh bài toán Hai vị tướng không có lời giải tuyệt đối. Năm 1978, Jim Gray đặt tên cho bài toán này trong tài liệu "Notes on Data Base Operating Systems". Khi gửi tin qua đường truyền có thể rớt mạng, bên gửi không bao giờ chắc chắn tin đã tới nơi. Vì vậy bạn chỉ có hai cách: gửi lại và chấp nhận tin bị trùng, hoặc không gửi lại và chấp nhận mất tin.

Đến thập niên 2000, các hệ thống truyền tin qua hàng đợi (message queue) và kiến trúc hướng sự kiện (EDA) bùng nổ. Khi mạng chập chờn, cơ chế gửi lại (at-least-once) làm máy chủ nhận trùng lệnh thanh toán hoặc gửi lặp email.

Trên ứng dụng di động, lập trình viên cũng hay cất các thông báo một lần vào chung biến trạng thái màn hình. Khi xoay ngang máy mà quên xóa thông báo, màn hình vẽ lại sẽ đọc biến cũ và bật thông báo lần nữa.

## Nó thực sự là cái gì
`State` và `Event` không phải là hai cái tên tùy thích đặt cho cùng một biến dữ liệu. `State` là dữ liệu tĩnh có tính lũy đẳng (idempotent - đọc lại vẫn như cũ), còn `Event` là tín hiệu đổi trạng thái. Trong [MVVM vs MVI](concept:architecture-mvvm-vs-mvi), MVI nối chúng thành vòng một chiều: `Event` đi lên để sinh `State` mới đi xuống.

## Cơ chế
Khi nhận một `Event`, hệ thống kiểm tra điều kiện rồi tính ra bản `State` mới. Trong hệ thống lưu nhật ký sự kiện (Event Sourcing) hay Redux, mọi `Event` đều ghi nối vào sổ log để dựng lại `State`. Tuy nhiên, các `side effect` như gọi API hay bật thông báo chỉ được chạy lúc `Event` mới xuất hiện lần đầu. Khi đọc lại log để khôi phục `State`, hệ thống chỉ tính lại dữ liệu chứ tuyệt đối không chạy lại `side effect`.

## Đánh đổi
Phân biệt rõ hai thứ giúp dẹp hẳn lỗi hiện lặp thông báo khi xoay máy và dễ dựng lại kịch bản lỗi. Đổi lại, bạn phải viết thêm bước xóa sự kiện sau khi đã hiển thị hoặc tách luồng xử lý riêng. Nếu màn hình chỉ hiện dữ liệu tĩnh đơn giản mà bày vẽ hệ thống `Event` đồ sộ thì code sẽ dài dòng thừa thãi.

## Nguyên lý gốc
Tính lũy đẳng (Idempotence): làm một thao tác năm lần vẫn ra kết quả y hệt như làm một lần. Trong HTTP, `GET` và `PUT` mang tính `State` vì gọi lại vẫn ra kết quả đó. Còn `POST` là `Event` vì mỗi lần gọi lại tạo thêm một bản ghi mới.

## Key insight
> `Event` không phải là `State` dùng một lần: `State` đọc lại bao nhiêu lần vẫn an toàn, còn `Event` làm phát sinh `side effect`.

## Ví von
`State` giống mực nước hiện trên thước đo của bồn chứa: ai đi ngang nhìn vào lúc nào cũng thấy đúng mực nước đó. Còn `Event` giống giọt nước vừa rơi xuống mặt bồn: nó chỉ làm mặt nước gợn sóng đúng một khoảnh khắc lúc chạm vào.

## Bẫy & ranh giới
Bẫy quen thuộc nhất trên Android là nhét câu thông báo lỗi vào `StateFlow` nhưng hiện xong lại quên gán nó về `null`. Khi xoay máy, giao diện dựng lại sẽ đọc trúng giá trị cũ và bật lại thông báo đó. Ở phía máy chủ, nếu hàm nhận tin không kiểm tra mã trùng (thiếu tính lũy đẳng), khách sẽ bị trừ tiền hai lần. Cũng đừng bắn sự kiện qua `Channel` hay [SharedFlow](concept:kotlin-stateflow-vs-sharedflow) mà không gắn vào `State`, vì màn hình tạm ẩn sẽ làm rơi sự kiện.

## Câu hỏi thực chiến
**Hỏi:** Nhét Event vào State rồi gán lại giá trị null sau khi dùng có ổn không?
Rất ổn, đây chính là cách tài liệu kiến trúc chính thức của Android khuyên bạn dùng. `ViewModel` đặt biến `userMessage` vào trong `UiState`, màn hình hiện thông báo xong thì gọi ngay một hàm để xóa trường đó về `null`. Nếu xoay máy ngay trước lúc hàm xóa chạy thì thông báo có thể hiện lại, nhưng bạn không bao giờ mất thông báo.

**Hỏi:** Lệnh điều hướng sang màn hình mới nên xử lý dưới dạng State hay Event, làm sao biết?
Bạn xem việc chuyển màn hình đó có cần `ViewModel` kiểm tra điều kiện hay lưu dữ liệu trước không. Nếu bấm nút là sang màn hình mới ngay thì giao diện tự gọi lệnh chuyển trang luôn, không cần đi vòng qua `ViewModel`. Còn nếu phải đợi lưu dữ liệu xong, `ViewModel` hãy bật một cờ trong `UiState` để màn hình nhìn vào đó chuyển trang.

**Hỏi:** Phân biệt State và Event có tác dụng thực sự gì trong ứng dụng thực tế?
Nó giúp bạn trị dứt điểm lỗi bật lặp hộp thoại hoặc hiện `Toast` hai lần mỗi khi xoay màn hình điện thoại. Ở máy chủ, khi chạy lại lịch sử sự kiện để khôi phục dữ liệu, hệ thống không gửi nhầm email cũ cho khách.

**Hỏi:** Tính lũy đẳng (idempotent) cứu nguy thế nào khi backend nhận trùng Event?
Nó đảm bảo dù máy chủ nhận cùng một tin nhắn nhiều lần thì kết quả vẫn như nhận một lần. Khi mạng yếu làm máy tự gửi lại lệnh thanh toán, máy chủ nhìn mã giao dịch duy nhất sẽ bỏ qua lần trùng. Nhờ đó, khách hàng không bị trừ tiền hai lần.

**Hỏi:** Thao tác nhấn một nút trên màn hình là State hay Event?
Cú chạm ngón tay vào nút là `Event`, còn việc nút đang sáng hay bị khóa mờ là `State`. Cú chạm gửi `Event` cho hệ thống xử lý, rồi hệ thống trả về `State` mới để vẽ lại nút.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được: State có tính lũy đẳng (đọc lại nhiều lần không hại), còn Event kích hoạt side effect đúng một lần.
- [ ] Nêu được ví dụ ngoài giao diện: tính lũy đẳng của HTTP GET/PUT so với POST, và việc phát lại log không chạy lại side effect trong Event Sourcing.
- [ ] Chỉ ra được bẫy quên xóa sự kiện về null trong StateFlow làm lặp thông báo khi xoay màn hình, và bẫy trừ tiền hai lần khi thiếu kiểm tra trùng.

## Tôi đã gặp nó ở đâu

