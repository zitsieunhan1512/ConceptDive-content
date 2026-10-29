---
id: android-viewmodel
title: ViewModel
domain: android
tags: [android, architecture, lifecycle, state-management]
collections: [interview-fundamental]
related: [android-state-holders, android-activity-lifecycle, android-memory-leak, architecture-state-vs-event]
created: 2026-09-07
diagrams: [Cơ chế|viewmodel-store-retention.png|ComponentActivity chuyển giao ViewModelStore qua onRetainNonConfigurationInstance để Activity mới nhận lại trên Heap khi xoay máy]
---

## Bản 30 giây
Khi người dùng xoay ngang điện thoại, Android hủy ngay Activity cũ rồi tạo một Activity mới tinh. `ViewModel` cất dữ liệu của màn hình ở một vùng nhớ riêng được hệ thống giữ lại qua lần xoay đó. Nhờ vậy, Activity mới tạo xong chỉ việc lấy lại dữ liệu đang có mà không phải gọi mạng tải lại từ đầu.

## Cần biết trước
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Trước năm 2017, xoay màn hình là nỗi ám ảnh quen thuộc của người làm Android. Mỗi lần đổi cấu hình máy, hệ thống gọi `onDestroy()` trên Activity cũ rồi dựng một Activity mới. Biến trong Activity mất sạch, còn cuộc gọi mạng đang chạy thì gãy ngang hoặc trả kết quả về màn hình cũ.

Cách cứu dữ liệu qua `onSaveInstanceState(Bundle)` thời đó chỉ gửi được dưới 1MB qua bộ đệm `Binder`. Nó không thể chứa nổi danh sách dài hay giữ lại cuộc gọi API đang chạy dở dang.

Tháng 5/2017 tại Google I/O, Google ra mắt bộ thư viện Android Architecture Components cùng với `ViewModel`. Lớp này đứng tách khỏi Activity để giữ dữ liệu và các tác vụ đang chạy đi qua những lần màn hình tạo lại.

## Nó thực sự là cái gì
`ViewModel` không phải là một luồng chạy ngầm hay có phép màu tự thoát khỏi bộ thu gom rác. Nó chỉ là đối tượng nằm trong `ViewModelStore`, được Activity cũ trao cho Activity mới qua `NonConfigurationInstance`.

## Cơ chế
Khi bạn xoay máy, `ComponentActivity` giữ lại kho `ViewModelStore` qua hàm `onRetainNonConfigurationInstance()`. Activity mới nhận lại đúng kho này, nên các `ViewModel` trên Heap vẫn còn tham chiếu và không bị thu gom rác. Trong Jetpack Compose, hàm `viewModel()` gắn vào `ViewModelStoreOwner` của từng đích điều hướng (navigation entry) với vòng đời riêng. Chỉ khi bạn bấm Back thoát hẳn màn hình hoặc gọi `finish()`, hệ thống mới gọi `ViewModelStore.clear()` để chạy hàm `onCleared()` và hủy `viewModelScope`.

## Đánh đổi
Bạn có một chỗ giữ dữ liệu và coroutine sống nguyên vẹn qua lúc xoay màn hình mà không bị gọi lại API. Đổi lại, `ViewModel` vẫn mất sạch khi hệ điều hành tắt tiến trình chạy ngầm (`Process Death`). Do sống lâu hơn Activity, `ViewModel` sẽ gây [rò rỉ bộ nhớ](concept:android-memory-leak) nếu giữ tham chiếu tới View hay Activity.

## Nguyên lý gốc
Tách vòng đời của dữ liệu nghiệp vụ ra khỏi vòng đời ngắn ngủi của các thành phần vẽ giao diện (lifecycle decoupling).

## Key insight
> `ViewModel` sống qua lúc xoay màn hình vì kho `ViewModelStore` chứa nó được Activity cũ trao tận tay cho Activity mới qua `NonConfigurationInstance`.

## Khuôn mẫu chuẩn
```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<String>("Init")
    val uiState: StateFlow<String> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = repository.fetchName()
        }
    }
}
```

## Bẫy & ranh giới
Nếu giữ `Activity`, `View` hay lambda trỏ tới giao diện trong `ViewModel`, màn hình cũ sẽ bị [rò rỉ bộ nhớ](concept:android-memory-leak). Lúc xoay máy, bộ thu gom rác không sao xóa được Activity cũ vì `ViewModel` vẫn đang nắm giữ nó trên Heap. Ngoài ra, để giữ dữ liệu khi hệ thống tắt app chạy ngầm, một mình `ViewModel` là không đủ mà bạn phải dùng thêm `SavedStateHandle`. Cũng đừng giữ sự kiện một lần bằng biến lẻ trong `ViewModel`, hãy theo [State vs Event](concept:architecture-state-vs-event) để tránh lặp khi xoay máy.

## Câu hỏi thực chiến
**Hỏi:** ViewModel có sống sót được khi app bị hệ điều hành diệt ngầm không?
Không. `ViewModel` chỉ là đối tượng nằm trên RAM của ứng dụng nên sẽ mất sạch khi hệ thống bắn tín hiệu `SIGKILL` tắt tiến trình. Muốn giữ lại dữ liệu qua tình huống này, bạn bắt buộc phải kết hợp với `SavedStateHandle`.

**Hỏi:** ViewModel có nên giữ Context không, làm sao nạp tài nguyên hệ thống?
Tuyệt đối không giữ `Activity Context`, vì xoay máy một lần là `ViewModel` sẽ giữ chặt lấy Activity cũ gây [rò rỉ bộ nhớ](concept:android-memory-leak). Nếu thật sự cần `Context` của hệ thống, bạn có thể kế thừa `AndroidViewModel` để chỉ giữ `Application Context` sống cùng toàn ứng dụng. Cách gọn nhất là `ViewModel` chỉ phát ra mã số tài nguyên như `R.string.error`, còn tầng giao diện tự đổi mã đó thành chữ.

**Hỏi:** Màn hình nào cũng bắt buộc phải có một ViewModel riêng không?
Không. Với những màn hình tĩnh chỉ hiện vài dòng giới thiệu hoặc hộp thoại đơn giản không gọi mạng, tạo thêm `ViewModel` là thừa thãi. Lúc đó bạn chỉ cần giữ trạng thái ngay trên giao diện cho đỡ phải tạo thêm lớp và viết code nối dây.

**Hỏi:** Hai Fragment trên cùng màn hình muốn dùng chung dữ liệu thì làm thế nào?
Bạn hãy lấy `ViewModel` theo vòng đời của Activity đang chứa hai Fragment đó thay vì lấy riêng theo từng Fragment. Nhờ dùng chung `ViewModelStoreOwner` là Activity, hai Fragment sẽ nhận cùng một `ViewModel` trên Heap để đọc ghi chung.

**Hỏi:** Khi nào thì ViewModel thực sự bị hủy và dọn dẹp khỏi bộ nhớ?
Khi người dùng bấm nút Back rời hẳn khỏi màn hình đó hoặc khi code của bạn tự gọi `finish()` đóng Activity. Lúc ấy hệ thống không giữ kho lại nữa mà gọi `ViewModelStore.clear()`, kích hoạt hàm `onCleared()` để hủy mọi coroutine trong `viewModelScope`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được ViewModel sống qua xoay máy là nhờ kho ViewModelStore được chuyển giao qua NonConfigurationInstance.
- [ ] Phân biệt được xoay màn hình (ViewModel còn trên RAM) với lúc hệ thống tắt tiến trình ngầm (ViewModel mất sạch, cần SavedStateHandle).
- [ ] Nêu được vì sao giữ Activity Context trong ViewModel lại gây rò rỉ bộ nhớ màn hình cũ.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
