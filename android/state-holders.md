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
Xoay màn hình khiến dữ liệu dễ gây sập app nếu bị gắn chặt vào vòng đời giao diện. State Holders đóng vai trò kho lưu trữ độc lập để giữ dữ liệu bền vững qua các biến cố của hệ thống. Chúng đóng gói logic hiển thị và cung cấp luồng dữ liệu an toàn cho Activity và [Fragment](concept:android-fragment-lifecycle).

## Cần biết trước
- kotlin-coroutines
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Năm 2017, Google ra mắt LiveData tại Google I/O để dập tắt lỗi observer bắn dữ liệu vào View đã chết. Trước đó, RxJava rất mạnh nhưng không nhận biết được vòng đời Android, khiến dev hay quên hủy đăng ký trong `onDestroy`.

Tuy nhiên, LiveData bị trói chặt vào Main Thread và Android SDK. Nó bất lực trong tầng domain thuần túy hoặc các dự án Kotlin Multiplatform (KMP).

Khi Kotlin Coroutines ra đời, JetBrains giới thiệu Flow, SharedFlow và StateFlow. Bộ ba này thay thế LiveData hoàn toàn mà không lệ thuộc vào bất kỳ thư viện Android nào.

## Nó thực sự là cái gì
StateFlow KHÔNG phải một công nghệ tách rời khỏi SharedFlow. Thực chất, StateFlow chính là một SharedFlow bị khóa cứng các tham số cấu hình để chuyên trách quản lý trạng thái.

```kotlin
interface StateFlow<out T> : SharedFlow<T> {
    val value: T
}
```

## Cơ chế
LiveData chỉ phát dữ liệu khi [Activity Lifecycle](concept:android-activity-lifecycle) ở mức STARTED hoặc RESUMED. Trong khi đó, SharedFlow là kênh phát đa hướng với bộ đệm và khả năng phát lại (replay) tùy biến. StateFlow khóa cứng replay bằng 1, tự lọc giá trị trùng qua `equals`, và bắt buộc có giá trị khởi tạo ban đầu.

## Đánh đổi
StateFlow giải phóng logic khỏi Android SDK và dùng được trên đa nền tảng KMP. Kết hợp `stateIn(WhileSubscribed)` và `repeatOnLifecycle` giúp tự động dừng nguồn phát ngược dòng khi giao diện ẩn. Đổi lại, `repeatOnLifecycle` dễ viết sai vị trí và LiveData vẫn đơn giản hơn cho mã Java thuần.

## Nguyên lý gốc
Nguyên lý [State vs Event](concept:architecture-state-vs-event): sự kiện từ ViewModel được quy về trạng thái, giao diện xử lý xong sẽ báo lại để xoá.

## Key insight
> [ViewModel](concept:android-viewmodel) không phải nơi gom mọi trạng thái: chỉ dữ liệu nghiệp vụ mới cần sống sót qua thay đổi cấu hình.

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
Bẫy phổ biến là đưa sự kiện vào StateFlow theo [State vs Event](concept:architecture-state-vs-event) mà quên xoá, khiến Toast bị lặp khi xoay máy. Ngoài ra, gọi `flow.collect` trực tiếp mà thiếu `repeatOnLifecycle` sẽ khiến luồng ngầm tiếp tục thu thập gây hao pin. Chống chỉ định đẩy sự kiện qua SharedFlow hoặc Channel vì không bảo đảm giao nhận khi ViewModel sống lâu hơn giao diện.

## Câu hỏi thực chiến
**Hỏi:** Có nên bỏ hết LiveData để chuyển sang StateFlow trong mọi dự án không?
Không nhất thiết. Với dự án Java cũ chạy thuần trên Android, LiveData vẫn đáp ứng tốt và ít nguy cơ mắc bẫy quản lý vòng đời luồng. Bạn chỉ nên chuyển sang StateFlow khi dự án dùng Kotlin hoàn toàn, cần làm đa nền tảng hoặc xử lý chuỗi luồng phức tạp.

**Hỏi:** Dùng StateFlow hay SharedFlow để bắn Toast thông báo, làm sao biết?
StateFlow, theo hướng dẫn chính thức của Android. Đưa thông báo vào UiState, giao diện hiển thị xong thì gọi hàm `userMessageShown()` để xoá giá trị về `null`. Dùng SharedFlow hoặc Channel không bảo đảm giao sự kiện khi giao diện đang được tạo lại lúc xoay máy.

**Hỏi:** Gom hết trạng thái màn hình vào một data class UiState duy nhất có hại gì không?
Có thể, nếu bạn truyền nguyên object UiState xuống mọi Composable con. Đổi một cờ nhỏ tạo ra object mới khiến mọi hàm con nhận object đó đều phải recompose. Truyền cho mỗi con đúng trường dữ liệu nó cần giúp Compose bỏ qua các thành phần không bị thay đổi.

**Hỏi:** Tham số WhileSubscribed(5000) trong stateIn có tác dụng thực sự gì?
Nó giữ luồng dữ liệu sống thêm đúng năm giây sau khi màn hình ngắt kết nối lắng nghe. Khoảng trễ này ngăn chặn việc hủy và nạp lại dữ liệu mạng tốn kém trong khoảnh khắc người dùng xoay máy.

**Hỏi:** Thu thập StateFlow trong Activity tại sao phải bọc bằng repeatOnLifecycle?
Để bảo đảm luồng thu thập dữ liệu tự động dừng lại khi màn hình khuất khỏi tầm nhìn. Nếu chỉ gọi `lifecycleScope.launch`, coroutine vẫn tiếp tục nhận dữ liệu ngầm gây lãng phí tài nguyên CPU và hao pin. `repeatOnLifecycle` sẽ tự khởi động lại việc lắng nghe ngay khi màn hình đạt trạng thái `STARTED`.

## Ý bắt buộc khi giải thích lại
- [ ] StateFlow thực chất là SharedFlow có replay bằng 1, lọc trùng qua equals và có giá trị khởi tạo.
- [ ] Quy sự kiện một lần về State và cơ chế giao diện báo lại để xoá sau khi xử lý.
- [ ] Bẫy quên xoá sự kiện trong State làm Toast lặp khi xoay máy; SharedFlow và Channel không bảo đảm giao nhận.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

