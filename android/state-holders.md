---
id: android-state-holders
title: State Holders
domain: android
tags: [android, architecture, state, flow, livedata]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-viewmodel, architecture-state-vs-event]
created: 2026-09-07
diagrams: [Cơ chế|state-holder-repeat-on-lifecycle.png|repeatOnLifecycle cổng chặn thu thập StateFlow theo vòng đời STARTED và STOPPED, kết hợp WhileSubscribed 5 giây giữ nguồn phát]
---

## Bản 30 giây
Khi người dùng xoay màn hình, dữ liệu gắn chặt vào giao diện sẽ bị mất hoặc làm sập ứng dụng. `State Holder` đứng tách riêng để giữ dữ liệu an toàn qua những lần màn hình bị hủy và tạo lại. Nó gom logic chuẩn bị dữ liệu hiển thị và phát luồng trạng thái xuống cho Activity hoặc [Fragment](concept:android-fragment-lifecycle) vẽ lên màn hình.

## Cần biết trước
- kotlin-coroutines
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Năm 2017 tại Google I/O, Google ra mắt `LiveData` để chặn lỗi bắn dữ liệu vào View đã bị hủy. Trước đó, nhiều dự án dùng RxJava hay quên hủy đăng ký trong `onDestroy`, gây rò rỉ bộ nhớ và sập app.

Tuy nhiên, `LiveData` lại bị cột chặt vào luồng chính Main Thread và thư viện Android SDK. Bạn không thể mang nó xuống tầng domain thuần Kotlin hay dùng chung cho dự án đa nền tảng Kotlin Multiplatform (`KMP`).

Khi Kotlin Coroutines phổ biến, JetBrains đưa ra bộ ba `Flow`, `SharedFlow` và `StateFlow`. Chúng thay thế trọn vẹn `LiveData` mà không phụ thuộc vào bất kỳ thư viện Android nào.

## Nó thực sự là cái gì
`StateFlow` không phải là một công cụ tách biệt hoàn toàn với `SharedFlow`. Thật ra, `StateFlow` chính là một `SharedFlow` đã khóa sẵn các tham số cấu hình để chuyên giữ trạng thái hiện tại của màn hình.

```kotlin
interface StateFlow<out T> : SharedFlow<T> {
    val value: T
}
```

## Cơ chế
`LiveData` tự biết nhìn [Activity Lifecycle](concept:android-activity-lifecycle) và chỉ phát dữ liệu khi màn hình đang ở trạng thái `STARTED` hoặc `RESUMED`. Ngược lại, `SharedFlow` là kênh phát cho nhiều nơi cùng nghe với bộ đệm và số lượng giá trị phát lại (`replay`) tùy bạn chọn. Còn `StateFlow` là `SharedFlow` đặc biệt với `replay = 1` và có giá trị đầu. Nó tự bỏ qua nếu giá trị mới trùng giá trị cũ qua `equals`.

## Đánh đổi
Dùng `StateFlow` giúp bạn tách sạch code khỏi Android SDK để viết unit test nhanh và dùng chung trên `KMP`. Khi ghép `stateIn(WhileSubscribed)` với `repeatOnLifecycle`, luồng dữ liệu phía trên cũng tự tạm nghỉ lúc người dùng ẩn app xuống nền. Đổi lại, bạn phải bọc `repeatOnLifecycle` ở giao diện, còn `LiveData` vẫn gọn hơn cho dự án Java thuần.

## Nguyên lý gốc
Theo nguyên lý [State vs Event](concept:architecture-state-vs-event), thông báo từ ViewModel nên là một trường trong trạng thái màn hình. Khi giao diện hiện xong thì báo lại để xóa đi.

## Key insight
> [ViewModel](concept:android-viewmodel) không phải nơi chứa mọi biến của màn hình: chỉ trạng thái nghiệp vụ cần sống qua lúc xoay máy mới đặt ở đó.

## Ví von
State là bảng hiện nhiệt độ phòng, ai bước vào trễ đều thấy cùng nhiệt độ đó. Event là tiếng chuông cửa reo, chỉ ai có mặt lúc chuông kêu mới nghe thấy.

## Khuôn mẫu chuẩn
```kotlin
// ViewModel
fun userMessageShown() = _uiState.update { it.copy(userMessage = null) }

// Activity
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            render(state)
            state.userMessage?.let { showMessage(it); viewModel.userMessageShown() }
        }
    }
}
```

## Bẫy & ranh giới
Nếu đưa thông báo vào `StateFlow` theo [State vs Event](concept:architecture-state-vs-event) mà quên xóa sau khi hiện, Toast sẽ bật lại khi xoay máy. Gọi `flow.collect` trong `lifecycleScope.launch` mà thiếu `repeatOnLifecycle` khiến luồng ngầm chạy tiếp khi ẩn app, gây tốn CPU và pin. Cũng đừng bắn thông báo một lần qua `SharedFlow` hay `Channel`, vì sự kiện dễ mất lúc xoay màn hình.

## Câu hỏi thực chiến
**Hỏi:** Có nên bỏ hết LiveData để chuyển sang StateFlow trong mọi dự án không?
Không cần thiết. Với dự án Java cũ chỉ chạy trên Android, `LiveData` vẫn làm tốt và ít sợ quên bọc vòng đời. Bạn nên dùng `StateFlow` khi dự án viết bằng Kotlin, cần biến đổi luồng dữ liệu bằng Coroutines hoặc chia sẻ code sang `KMP`.

**Hỏi:** Dùng StateFlow hay SharedFlow để bắn Toast thông báo, làm sao biết?
Hãy dùng `StateFlow`, theo đúng hướng dẫn của đội ngũ Android. Bạn thêm trường thông báo vào `UiState`, khi màn hình hiện Toast xong thì gọi ngay `userMessageShown()` để đặt nó về `null`. Nếu dùng `SharedFlow` hay `Channel`, thông báo bắn ra lúc đang xoay máy có thể bị mất luôn.

**Hỏi:** Gom hết trạng thái màn hình vào một data class UiState duy nhất có hại gì không?
Có thể gây vẽ lại thừa nếu bạn truyền nguyên cục `UiState` đó xuống cho mọi hàm Composable con. Khi một trường nhỏ đổi, cả đối tượng `UiState` được tạo mới khiến mọi hàm con nhận nó đều phải chạy lại. Bạn chỉ nên tách và truyền cho mỗi hàm con đúng trường dữ liệu mà nó thực sự đọc.

**Hỏi:** Tham số WhileSubscribed(5000) trong stateIn có tác dụng thực sự gì?
Nó giữ cho luồng dữ liệu phía trên sống thêm 5 giây sau khi người nghe cuối cùng ngừng thu thập. Lúc xoay máy, Activity mới nối lại chỉ mất vài trăm mili giây nên luồng mạng hay DB không bị ngắt. Còn nếu người dùng bấm Home thoát ra ngoài quá 5 giây, luồng phía trên mới thực sự dừng để tiết kiệm tài nguyên.

**Hỏi:** Thu thập StateFlow trong Activity tại sao phải bọc bằng repeatOnLifecycle?
Vì `StateFlow` là thư viện Kotlin thuần nên tự nó không biết lúc nào màn hình Android bị che khuất. Chỉ gọi `collect` trong `lifecycleScope.launch` làm coroutine vẫn thức nhận dữ liệu khi app xuống nền, gây tốn CPU và pin. Bọc `repeatOnLifecycle(Lifecycle.State.STARTED)` sẽ tự hủy thu thập khi màn hình `STOPPED` và tự thu thập lại khi màn hình lên `STARTED`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được StateFlow thực chất là SharedFlow có replay = 1, có giá trị khởi tạo và lọc trùng bằng equals.
- [ ] Trình bày được cách biến sự kiện thông báo thành một trường trong UiState rồi gọi hàm xóa về null sau khi đã hiển thị.
- [ ] Nêu được bẫy quên xóa sự kiện làm Toast hiện lặp khi xoay máy và bẫy thiếu repeatOnLifecycle làm coroutine chạy ngầm tốn pin.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

