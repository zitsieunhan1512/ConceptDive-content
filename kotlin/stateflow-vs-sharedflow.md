---
id: kotlin-stateflow-vs-sharedflow
title: StateFlow vs SharedFlow
domain: kotlin
tags: [kotlin, coroutines, flow, reactive, android]
collections: [interview-fundamental]
related: [architecture-state-vs-event, android-viewmodel, kotlin-coroutines]
diagrams: [Cơ chế|stateflow-conflation-buffer.png|Conflation nuốt giá trị trùng trong StateFlow và cấu hình Replay trong SharedFlow]
created: 2026-09-15
---

## Bản 30 giây
`StateFlow` và `SharedFlow` là hai luồng phát dữ liệu nóng (*Hot Stream*) trong thư viện Coroutines của Kotlin. `StateFlow` luôn giữ giá trị hiện tại để phát cho người nghe mới, thích hợp để hiển thị trạng thái giao diện. `SharedFlow` phát sự kiện cho nhiều bên cùng nghe mà không bắt buộc lưu trạng thái, phù hợp cho luồng thông báo hoặc tin nhắn.

## Cần biết trước
- kotlin-coroutines
- architecture-state-vs-event

## Vấn đề buộc nó ra đời
Trước đây, lập trình viên Android dùng `LiveData` để đẩy dữ liệu từ ViewModel sang giao diện. `LiveData` gắn chặt với vòng đời của hệ điều hành Android và khó tích hợp trong tầng dữ liệu đa nền tảng.

Năm 2020, JetBrains bổ sung `SharedFlow` và `StateFlow` vào thư viện `kotlinx.coroutines` 1.4.0. Giải pháp này thay thế RxJava và LiveData bằng cơ chế luồng thuần Kotlin, hoạt động đồng nhất trên đa nền tảng KMP.

Khi sử dụng luồng dữ liệu lạnh thông thường, mỗi bên thu thập lại kích hoạt lại công việc từ đầu. Hệ thống cần các luồng nóng chia sẻ kết quả để tránh gọi mạng hoặc truy vấn cơ sở dữ liệu nhiều lần lãng phí.

## Nó thực sự là cái gì
`StateFlow` KHÔNG phải là một cấu trúc hoàn toàn tách biệt. Thực chất, `StateFlow` là hiện thực chuyên biệt của `SharedFlow` với bộ đệm phát lại bằng một và luôn có giá trị đầu.

Cả hai đều là luồng nóng (*Hot Stream*), phát dữ liệu độc lập ngay cả khi không có người thu thập nào đang lắng nghe. Chúng chia sẻ cùng một luồng phát cho nhiều bên thu thập thay vì chạy lại logic từ đầu như [luồng lạnh](concept:kotlin-flow-under-the-hood).

## Cơ chế
`SharedFlow` duy trì một hàng đợi đệm các giá trị gần nhất (*replay cache*) để gửi cho các bên đăng ký mới.

`StateFlow` giữ bộ đệm phát lại bằng một và tự động so sánh bằng phương thức `equals` trước khi phát giá trị mới. Nếu giá trị mới giống hệt giá trị cũ, `StateFlow` sẽ nuốt giá trị đó (*conflation*) và không báo cho các bên thu thập. Khi hàng đợi của `SharedFlow` đầy, hàm `emit` mặc định sẽ treo luồng phát (*suspend*) trừ khi chọn cấu hình thả rơi.

## Đánh đổi
`StateFlow` và `SharedFlow` mang lại tính bất đồng bộ mạnh mẽ và hoạt động trên mọi nền tảng không lệ thuộc vào Android SDK. Đổi lại, chúng không tự động ngắt theo vòng đời màn hình như `LiveData` mà phải dựa vào công cụ hỗ trợ như `repeatOnLifecycle`.

Thu thập luồng nóng sai cách sẽ âm thầm làm rò rỉ tài nguyên mạng và pin ngay cả khi màn hình đã ẩn.

## Nguyên lý gốc
Mẫu hình xuất bản - đăng ký (*Publish-Subscribe*): bên phát dữ liệu và các bên nhận dữ liệu hoàn toàn tách rời nhau. Bên phát chỉ quan tâm đến việc đẩy tin vào đường ống chung mà không cần biết ai đang lắng nghe.

## Key insight
> StateFlow không phải luồng phát sự kiện, mà là bộ nhớ đệm trạng thái luôn nuốt các giá trị trùng lặp liên tiếp.

## Khuôn mẫu chuẩn
```kotlin
class ChatViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<ChatUiState>(ChatUiState.Idle)
    val uiState: StateFlow<ChatUiState> = _uiState.asStateFlow()

    private val _incomingMessages = MutableSharedFlow<Message>(extraBufferCapacity = 64)
    val incomingMessages: SharedFlow<Message> = _incomingMessages.asSharedFlow()

    fun onReceive(msg: Message) = _incomingMessages.tryEmit(msg)
}
```

## Bẫy & ranh giới
Thu thập luồng trong `lifecycleScope.launch` mà không có `repeatOnLifecycle` sẽ khiến luồng tiếp tục chạy ngầm khi màn hình ở Background. Triệu chứng là ứng dụng tiêu hao pin bất thường, rò rỉ bộ nhớ hoặc gây lỗi crash vì cập nhật View đã bị hủy. Chống chỉ định dùng `StateFlow` cho sự kiện một lần như hiển thị Toast vì giá trị bị phát lại khi xoay màn hình. Cũng không nên lạm dụng `SharedFlow` với `replay = 0` cho sự kiện quan trọng vì sự kiện sẽ mất trắng nếu chưa có ai lắng nghe.

## Câu hỏi thực chiến
**Hỏi:** Tại sao không nên dùng lifecycleScope.launch trực tiếp để thu thập StateFlow trong Fragment?
Vì luồng thu thập sẽ không dừng lại khi Fragment đi vào trạng thái dừng (*STOPPED*). Nó tiếp tục tiêu tốn CPU và cập nhật giao diện khi màn hình không hiển thị. Bạn nên dùng `repeatOnLifecycle(Lifecycle.State.STARTED)` để tự động hủy và khởi động lại luồng.

**Hỏi:** StateFlow xử lý thế nào khi ViewModel liên tiếp phát hai giá trị giống hệt nhau?
Nó tự động bỏ qua giá trị thứ hai và không phát cho giao diện. `StateFlow` tích hợp sẵn toán tử so sánh `equals` để chỉ phát khi trạng thái thực sự thay đổi. Muốn phát mọi giá trị lặp lại, bạn phải chuyển sang dùng `SharedFlow`.

**Hỏi:** Có nên dùng SharedFlow với replay bằng 0 để làm sự kiện điều hướng màn hình không?
Không nên vì rất dễ làm mất sự kiện quan trọng. Nếu ViewModel phát sự kiện trước khi màn hình kịp đăng ký lắng nghe, sự kiện đó sẽ biến mất hoàn toàn. Thay vào đó, bạn nên gom sự kiện vào UiState của `StateFlow` để giao diện tiêu thụ theo trạng thái.

## Ý bắt buộc khi giải thích lại
- [ ] StateFlow là hiện thực chuyên biệt của SharedFlow có replay bằng 1 và luôn có giá trị khởi tạo.
- [ ] Conflation trong StateFlow: tự động bỏ qua giá trị mới nếu trùng khớp giá trị cũ qua equals.
- [ ] Quy tắc vòng đời Android: thu thập bằng repeatOnLifecycle để tránh rò rỉ tài nguyên ở Background.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

