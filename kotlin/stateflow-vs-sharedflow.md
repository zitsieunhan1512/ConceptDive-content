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
`StateFlow` và `SharedFlow` là hai luồng phát dữ liệu nóng (*Hot Stream*) trong thư viện Coroutines của Kotlin. `StateFlow` luôn giữ giá trị hiện tại để vẽ màn hình và chứa luôn các sự kiện trong `UiState`. Còn `SharedFlow` phát sự kiện ở tầng dữ liệu cho nhiều nơi cùng nghe mà không bắt buộc lưu lại giá trị cũ.

## Cần biết trước
- kotlin-coroutines
- architecture-state-vs-event

## Vấn đề buộc nó ra đời
Trước đây, lập trình viên Android dùng `LiveData` để đưa dữ liệu từ ViewModel lên màn hình. Nhưng `LiveData` dính chặt vào hệ điều hành Android nên không đem xuống tầng dữ liệu dùng chung đa nền tảng được.

Còn nếu dùng luồng lạnh thông thường, cứ thêm một nơi thu thập là hàm lại chạy lại từ đầu. Nếu hai chỗ trên màn hình cùng nghe một luồng lạnh gọi API, ứng dụng sẽ bắn hai request giống hệt nhau.

Năm 2020, JetBrains đưa `SharedFlow` và `StateFlow` vào `kotlinx.coroutines` 1.4.0. Hai luồng nóng này viết thuần bằng Kotlin, chạy được trên mọi nền tảng KMP và chia sẻ chung kết quả cho nhiều nơi nghe.

## Nó thực sự là cái gì
`StateFlow` không phải một cấu trúc hoàn toàn khác với `SharedFlow`. Thực ra, `StateFlow` chính là một `SharedFlow` được cài sẵn bộ đệm phát lại bằng 1 và bắt buộc có giá trị ban đầu.

Cả hai đều là luồng nóng (*Hot Stream*), vẫn giữ và nhận dữ liệu mới ngay cả khi chưa có ai lắng nghe. Nhiều nơi có thể cùng nghe một luồng mà không làm code bên trong chạy lại từ đầu như [luồng lạnh](concept:kotlin-flow-under-the-hood).

## Cơ chế
`SharedFlow` giữ một hàng đợi các giá trị gần nhất (*replay cache*) để gửi lại cho những nơi vào nghe sau.

`StateFlow` luôn giữ đúng 1 giá trị mới nhất và lấy hàm `equals` ra so sánh mỗi khi bạn gán giá trị mới. Nếu giá trị mới bằng hệt giá trị đang có, `StateFlow` bỏ qua không phát lại nữa (*conflation*). Còn ở `SharedFlow`, khi hàng đợi đầy, hàm `emit` mặc định sẽ tạm dừng chờ trừ khi bạn cấu hình bỏ bớt.

## Đánh đổi
Nhờ viết thuần bằng Kotlin, bạn dùng được `StateFlow` và `SharedFlow` từ tầng dữ liệu tới ViewModel mà không phụ thuộc vào Android SDK. Đổi lại, chúng không tự dừng khi màn hình ẩn đi như `LiveData`, nên bạn phải bọc bằng `repeatOnLifecycle`.

Nếu thu thập luồng nóng mà quên gắn với vòng đời màn hình, ứng dụng vẫn chạy ngầm gây tốn pin và mạng.

## Nguyên lý gốc
Bên phát chỉ đẩy dữ liệu vào đường ống chung, không cần biết có bao nhiêu bên đang lắng nghe ở đầu kia (*Publish-Subscribe*).

## Key insight
> `StateFlow` không phải luồng bắn sự kiện, mà là ô nhớ giữ trạng thái hiện tại và tự bỏ qua giá trị cũ.

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
Nếu bạn thu thập thẳng trong `lifecycleScope.launch` mà thiếu `repeatOnLifecycle`, luồng vẫn tiếp tục nhận dữ liệu khi người dùng đã ẩn ứng dụng. Hậu quả là máy hao pin và ứng dụng có thể sập vì cố cập nhật lên View đã bị hủy. Đừng nhét sự kiện như Toast vào một `StateFlow` riêng mà quên xóa, vì xoay màn hình sẽ hiện lại Toast đó. Cũng đừng dùng `SharedFlow` có `replay = 0` để điều hướng từ ViewModel, vì bắn lúc màn hình chưa kịp nghe là mất luôn.

## Câu hỏi thực chiến
**Hỏi:** Tại sao không nên dùng lifecycleScope.launch trực tiếp để thu thập StateFlow trong Fragment?
Vì khi người dùng bấm Home đưa Fragment về trạng thái `STOPPED`, coroutine đó vẫn không dừng lại. Nó cứ chạy ngầm và cố vẽ lại giao diện đang bị che khuất. Bạn hãy bọc trong `repeatOnLifecycle(Lifecycle.State.STARTED)` để tự hủy khi ẩn màn hình và nghe lại khi mở lên.

**Hỏi:** StateFlow xử lý thế nào khi ViewModel liên tiếp phát hai giá trị giống hệt nhau?
Nó bỏ qua lần gán thứ hai và không báo gì cho giao diện. Trước khi phát, `StateFlow` luôn gọi `equals` để kiểm tra xem giá trị mới có khác giá trị đang giữ hay không. Nếu bạn cần phát cả những lần trùng nhau, hãy dùng `SharedFlow`.

**Hỏi:** Có nên dùng SharedFlow với replay bằng 0 để làm sự kiện điều hướng màn hình không?
Không nên, vì sự kiện đó rất dễ bị rơi mất. Nếu ViewModel bắn sự kiện lúc màn hình đang xoay hoặc chưa kịp thu thập, lệnh chuyển màn hình sẽ mất. Bạn nên đặt một trường cờ hoặc danh sách sự kiện ngay trong `UiState` của `StateFlow`, xử lý xong thì báo ViewModel xóa đi.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được StateFlow là dạng riêng của SharedFlow luôn có giá trị ban đầu và giữ lại 1 giá trị mới nhất.
- [ ] Nêu được cách StateFlow dùng equals để bỏ qua giá trị trùng (conflation) và cách SharedFlow xử lý khi đầy bộ đệm.
- [ ] Giải thích được vì sao phải dùng repeatOnLifecycle khi thu thập trên Android và vì sao sự kiện UI nên đưa vào UiState.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

