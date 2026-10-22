---
id: architecture-state-vs-event
title: State vs Event
domain: architecture
tags: [architecture, state-machine, event-driven, distributed-systems]
collections: []
related: [android-state-holders, android-viewmodel, architecture-mvvm-vs-mvi, kotlin-stateflow-vs-sharedflow]
created: 2026-09-09
diagrams: [Cơ chế|state-vs-event-lifecycle.png|State có tính lũy đẳng đọc lại vô hại, Event chỉ kích hoạt tác vụ phụ một lần và phải xóa null sau khi dùng]
---

## Bản 30 giây
Trộn lẫn giữa sự kiện tạm thời và trạng thái bền vững là nguồn gốc của lỗi lặp thông báo và sai lệch dữ liệu. Trạng thái phản ánh thực tại hiện thời, có thể đọc lại vô số lần mà không đổi kết quả. Sự kiện đại diện cho biến cố vừa xảy ra, chỉ được xử lý đúng một lần duy nhất.

## Cần biết trước

## Vấn đề buộc nó ra đời
Năm 1975, Akkoyunlu, Ekanadham và Huber chứng minh bài toán Hai vị tướng không có lời giải. Năm 1978, Jim Gray đặt tên cho nó trong tài liệu "Notes on Data Base Operating Systems". Qua kênh truyền có thể mất tin, bên gửi không bao giờ chắc tin đã tới. Vì vậy chỉ còn hai lựa chọn: gửi lại và chấp nhận trùng lặp, hoặc không gửi lại và chấp nhận mất.

Thập niên 2000, kiến trúc hướng sự kiện (EDA) và message queue bùng nổ. Khi mạng chập chờn, cơ chế gửi lại (at-least-once) khiến các consumer nhận trùng sự kiện thanh toán hoặc gửi thư nhiều lần.

Trên giao diện người dùng, lập trình viên thường lưu các hành động tạm thời vào luồng dữ liệu hiển thị. Khi xoay màn hình mà quên xoá, hệ thống đọc lại dữ liệu cũ và vô tình kích hoạt lại hành động đó.

## Nó thực sự là cái gì
State và Event KHÔNG phải hai cách gọi tên tùy ý cho cùng một biến dữ liệu. Thực chất, State là trạng thái tĩnh có tính lũy đẳng (idempotent), còn Event là thông điệp kích hoạt chuyển đổi trạng thái. Trong kiến trúc [MVVM vs MVI](concept:architecture-mvvm-vs-mvi), MVI chuẩn hóa sự phân định này thành luồng dữ liệu một chiều khép kín.

## Cơ chế
Hệ thống xử lý Event bằng cách kiểm tra điều kiện rồi áp dụng thay đổi vào State hiện tại. Trong mô hình Event Sourcing và Redux, toàn bộ lịch sử Event được ghi thành log bất biến để tái tạo State khi cần. Tuy nhiên, các tác vụ phụ (side-effect) như gọi API hay bắn thông báo chỉ được kích hoạt khi Event phát sinh lần đầu. Khi khôi phục State từ log, hệ thống chỉ chạy logic biến đổi dữ liệu thuần túy và cấm chạy lại side-effect.

## Đánh đổi
Tách bạch hai khái niệm giúp hệ thống miễn nhiễm với lỗi lệch pha dữ liệu và tái hiện bug dễ dàng. Đổi lại, bạn phải duy trì hai đường ống truyền dữ liệu riêng biệt và xử lý logic phức tạp hơn. Nếu áp dụng cực đoan cho các màn hình đơn giản, việc tạo thêm lớp Event sẽ làm tăng mã bản mẫu không cần thiết.

## Nguyên lý gốc
Tính lũy đẳng (Idempotence): thực hiện một thao tác nhiều lần vẫn cho ra kết quả giống hệt làm một lần. HTTP GET và PUT là State vì có tính lũy đẳng, còn POST là Event vì mỗi lần gọi tạo ra biến cố mới.

## Key insight
> Event không phải là State dùng một lần: State luôn lũy đẳng khi đọc lại, còn Event chỉ kích hoạt tác vụ phụ.

## Ví von
State là mực nước hiển thị trên đồng hồ đo, ai nhìn vào thời điểm nào cũng thấy mực nước đó. Event là giọt nước nhỏ vào bồn, chỉ kích hoạt gợn sóng ngay lúc giọt nước chạm mặt nước.

## Bẫy & ranh giới
Bẫy kinh điển là đưa sự kiện một lần vào `StateFlow` mà không xoá sau khi giao diện đã xử lý. Khi xoay màn hình, giao diện mới dựng nhận lại trạng thái cũ và kích hoạt lại thông báo ngoài ý muốn. Ở backend, consumer nhận thông điệp thiếu tính lũy đẳng sẽ gây lỗi trừ tiền hai lần khi mạng chập chờn. Chống chỉ định đẩy sự kiện qua Channel hay [SharedFlow](concept:kotlin-stateflow-vs-sharedflow) khi ViewModel sống lâu hơn giao diện vì không bảo đảm giao nhận.

## Câu hỏi thực chiến
**Hỏi:** Nhét Event vào State rồi gán lại giá trị null sau khi dùng có ổn không?
Ổn, đây chính là cách hướng dẫn chính thức của Android khuyên dùng. ViewModel đặt `userMessage` vào UiState, giao diện hiện xong sẽ gọi hàm để xoá trường đó về `null`. Xoay máy trước khi kịp xoá thì thông báo có thể hiện lại, nhưng không bao giờ bị rơi rớt sự kiện.

**Hỏi:** Lệnh điều hướng sang màn hình mới nên xử lý dưới dạng State hay Event, làm sao biết?
Tuỳ vào việc bạn có cần ViewModel xác thực logic nghiệp vụ trước khi chuyển trang hay không. Chuyển trang thuần tuý thì UI tự gọi điều hướng mà không cần thông qua ViewModel. Khi cần kiểm tra điều kiện, ViewModel phơi cờ trạng thái trong UiState để giao diện quan sát và tự điều hướng.

**Hỏi:** Phân biệt State và Event có tác dụng thực sự gì trong ứng dụng thực tế?
Nó dập tắt triệt để lỗi nhảy lặp hộp thoại hoặc gửi thông báo hai lần khi người dùng xoay điện thoại. Ở hệ thống lớn, phát lại lịch sử sự kiện để dựng dữ liệu sẽ không kích hoạt gửi thư hay trừ tiền.

**Hỏi:** Tính lũy đẳng (idempotent) cứu nguy thế nào khi backend nhận trùng Event?
Nó bảo đảm xử lý một thông điệp nhiều lần vẫn cho ra cùng một kết quả như xử lý lần đầu. Khi mạng chập chờn, yêu cầu gửi lại sẽ không trừ tiền hai lần nhờ kiểm tra mã giao dịch duy nhất.

**Hỏi:** Thao tác nhấn một nút trên màn hình là State hay Event?
Nhát bấm của ngón tay là Event, còn việc nút bấm đang bị vô hiệu hóa hay sáng đèn là State. Event nhấn nút gửi đi tín hiệu xử lý, sau đó hệ thống mới tính toán và cập nhật State mới lên giao diện.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: State có tính lũy đẳng (đọc lại vô hại), Event chỉ kích hoạt tác vụ phụ đúng một lần.
- [ ] Ứng dụng ngoài giao diện: tính lũy đẳng của HTTP GET/PUT và cơ chế replay log không side-effect trong Event Sourcing.
- [ ] Bẫy quên xoá sự kiện trong StateFlow làm lặp thông báo và bẫy trừ tiền trùng ở consumer message queue.

## Tôi đã gặp nó ở đâu

