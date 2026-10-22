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
Khi người dùng xoay màn hình, Android lập tức hủy và tạo mới lại Activity. ViewModel lưu trữ dữ liệu ở một vùng nhớ riêng được hệ thống giữ lại qua đợt xoay. Màn hình mới chỉ cần kết nối lại với dữ liệu cũ mà không phải tải lại từ đầu.

## Cần biết trước
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Trước năm 2017, xoay màn hình là ác mộng lớn nhất của lập trình viên Android. Mỗi lần đổi cấu hình, hệ thống gọi `onDestroy()` và tạo một Activity hoàn toàn mới. Mọi biến cục bộ bị xóa sạch và luồng mạng đứt gãy dễ gây sập app.

Cơ chế `onSaveInstanceState(Bundle)` thời đó chỉ chứa được dưới 1MB qua bộ nhớ Binder. Nó bất lực trước danh sách dữ liệu lớn hoặc tiến trình bất đồng bộ đang chạy dở.

Tháng 5/2017, Google ra mắt Android Architecture Components tại Google I/O. ViewModel ra đời như một kho chứa trạng thái bền vững, sống sót an toàn qua việc tạo lại giao diện.

## Nó thực sự là cái gì
ViewModel KHÔNG phải một luồng chạy ngầm hay thành phần tự sống sót thần kỳ. Thực chất, nó là đối tượng thông thường nằm trong `ViewModelStore`, được Activity chuyển giao qua cơ chế `NonConfigurationInstance`.

## Cơ chế
Khi xoay máy, `ComponentActivity` lưu giữ lại `ViewModelStore` thông qua `onRetainNonConfigurationInstance()`. Activity mới dựng sẽ nhận lại kho này, giúp các đối tượng ViewModel trên Heap không bị Garbage Collector thu hồi. Trong Compose, hàm `viewModel()` gắn vào `ViewModelStoreOwner` của màn hình điều hướng (navigation entry) với vòng đời độc lập. Khi màn hình thực sự kết thúc qua `finish()` hoặc pop backstack, hệ thống mới gọi `ViewModelStore.clear()` để kích hoạt `onCleared()`.

## Đánh đổi
Bạn có nơi giữ dữ liệu bền vững qua xoay màn hình mà không đứt mạng. Đổi lại, ViewModel không sống sót được khi hệ điều hành tiêu diệt tiến trình ngầm (process death). ViewModel sống lâu hơn Activity nên dễ gây [rò rỉ bộ nhớ](concept:android-memory-leak) nếu giữ nhầm tham chiếu UI.

## Nguyên lý gốc
Nguyên lý tách biệt vòng đời (lifecycle decoupling): tách trạng thái nghiệp vụ khỏi các thành phần giao diện tạm thời và dễ biến động.

## Key insight
> ViewModel sống qua xoay màn hình nhờ ViewModelStore được Activity chuyển giao qua NonConfigurationInstance.

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
Lưu tham chiếu `Activity`, `View` hoặc callback UI trong ViewModel sẽ gây [rò rỉ bộ nhớ](concept:android-memory-leak) toàn bộ màn hình cũ. Khi xoay máy, Garbage Collector không thể dọn dẹp Activity cũ vì ViewModel vẫn nắm giữ nó. Để sống sót qua biến cố bị hệ thống diệt app ngầm, bạn bắt buộc phải dùng `SavedStateHandle`. Chống chỉ định lưu sự kiện một lần trong ViewModel; hãy tuân thủ nguyên lý [State vs Event](concept:architecture-state-vs-event) để tránh phát lặp thông báo.

## Câu hỏi thực chiến
**Hỏi:** ViewModel có sống sót được khi app bị hệ điều hành diệt ngầm không?
Không. ViewModel chỉ nằm trên vùng nhớ Heap thông thường nên sẽ tan biến khi hệ thống gửi `SIGKILL` xóa tiến trình. Để phục hồi dữ liệu sau khi bị diệt ngầm, bạn bắt buộc phải dùng `SavedStateHandle`.

**Hỏi:** ViewModel có nên giữ Context không, làm sao nạp tài nguyên hệ thống?
Tuyệt đối không giữ Activity Context vì chắc chắn gây [rò rỉ bộ nhớ](concept:android-memory-leak) màn hình cũ sau khi xoay máy. Nếu bắt buộc cần tài nguyên hệ thống, hãy dùng `AndroidViewModel` để chỉ giữ lại duy nhất một `Application Context` an toàn. Tốt nhất là ViewModel chỉ trả về mã ID tài nguyên để tầng giao diện tự trích xuất chuỗi hiển thị.

**Hỏi:** Màn hình nào cũng bắt buộc phải có một ViewModel riêng không?
Không. Những màn hình tĩnh chỉ hiển thị thông tin đơn giản hoặc form ngắn không cần logic phức tạp thì không nên dùng. Ép buộc tạo ViewModel cho các trường hợp vụn vặt này chỉ làm phình to mã nguồn và tăng chi phí bảo trì.

**Hỏi:** Hai Fragment trên cùng màn hình muốn dùng chung dữ liệu thì làm thế nào?
Khởi tạo ViewModel gắn vào phạm vi của Activity chứa chúng thay vì gắn vào từng Fragment riêng lẻ. Khi dùng chung một `ViewModelStoreOwner`, cả hai Fragment sẽ cùng nhận về một thể hiện duy nhất trên Heap để trao đổi thông tin.

**Hỏi:** Khi nào thì ViewModel thực sự bị hủy và dọn dẹp khỏi bộ nhớ?
Khi người dùng bấm nút Back thoát hẳn màn hình hoặc Activity chủ động gọi lệnh `finish()`. Lúc này, kho `ViewModelStore` sẽ được dọn sạch và tự động kích hoạt hàm `onCleared()` để hủy bỏ mọi công việc dở dang.

## Ý bắt buộc khi giải thích lại
- [ ] Cơ chế sống sót bản chất là ViewModelStore được chuyển qua NonConfigurationInstance.
- [ ] Phân biệt được sống sót qua xoay màn hình với tình huống bị hệ điều hành diệt tiến trình.
- [ ] Bẫy giữ Context gây rò rỉ bộ nhớ màn hình cũ và cách dùng SavedStateHandle cứu dữ liệu.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
