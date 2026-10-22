---
id: architecture-mvvm-vs-mvi
title: MVVM vs MVI
domain: architecture
tags: [architecture, mvvm, mvi, unidirectional-data-flow, state-management]
collections: [interview-fundamental]
related: [android-state-holders, android-viewmodel, architecture-state-vs-event]
created: 2026-09-07
diagrams: [Cơ chế|mvi-unidirectional-data-flow.png|Luồng dữ liệu một chiều khép kín: UI phát Intent lên một cửa duy nhất và nhận State bất biến cập nhật qua CAS]
---

## Bản 30 giây
Khi [ViewModel](concept:android-viewmodel) chứa nhiều biến rời rạc, màn hình rất dễ rơi vào các trạng thái hiển thị mâu thuẫn. MVI đặt lên MVVM hai ràng buộc: một State bất biến duy nhất và đúng một cửa tiếp nhận sự kiện. Nhờ vậy, mọi thay đổi giao diện đều lần về được một chỗ, giúp dễ kiểm soát dữ liệu và tái hiện lỗi.

## Cần biết trước
- android-state-holders

## Vấn đề buộc nó ra đời
Trên Android, MVI được mang vào để chữa nỗi đau phân mảnh dữ liệu của kiến trúc MVVM truyền thống. Trong MVVM, một [ViewModel](concept:android-viewmodel) thường phơi bày nhiều luồng LiveData hoặc StateFlow riêng lẻ để cập nhật độc lập cho từng phần giao diện.

Khi ứng dụng mở rộng, nhiều coroutine và hàm public sửa các biến này cùng lúc dẫn đến xung đột ngoài ý muốn. Màn hình dễ rơi vào trạng thái kỳ quặc như vừa bật cờ tải dữ liệu lại vừa báo lỗi mạng.

Lập trình viên muốn dò lỗi phải lục lọi qua hàng chục hàm sửa trạng thái rải rác trong class. Sự vắng bóng của một nguồn chân lý tập trung khiến việc tái hiện bug trở thành cơn ác mộng.

## Nó thực sự là cái gì
MVI KHÔNG phải một kiến trúc tách rời hay thay thế MVVM. Thực chất, đó là hai ràng buộc bổ sung: một State bất biến duy nhất và một hàm nhận Intent duy nhất. Ở tầng máy tính, StateFlow chỉ lưu một tham chiếu trên Heap; mỗi lần đổi trạng thái là tráo con trỏ sang object mới.

## Cơ chế
Hàm nhận Intent xử lý sự kiện qua hàm chuyển đổi thuần túy để tính toán ra bản trạng thái tiếp theo. ViewModel dùng lệnh `_state.update { it.copy(...) }` để chạy vòng lặp so sánh và hoán đổi giá trị nguyên tử. Cơ chế này đảm bảo chỉ ghi đè khi không ai xen vào giữa, tự động tính lại nếu có luồng khác vừa ghi. Ngược lại, đọc rồi gán tách rời `_state.value = _state.value.copy(...)` trên hai thread khác nhau sẽ ghi đè làm mất cập nhật của nhau.

## Đánh đổi
MVI đem lại khả năng tái hiện lỗi tuyệt vời nhưng bắt bạn trả giá bằng lượng lớn class sự kiện trung gian. Màn hình Compose đọc cả object State lớn sẽ bị recompose dư thừa nếu không tách nhỏ dữ liệu truyền xuống con. Ngoài ra, xử lý Intent tuần tự có thể khiến tác vụ mạng chậm làm các Intent phía sau phải chờ.

## Nguyên lý gốc
Nguyên lý luồng dữ liệu một chiều: trạng thái luôn chảy từ trên xuống, còn mọi sự kiện chỉ chảy ngược từ dưới lên.

## Key insight
> Object bất biến không làm phép cập nhật thành nguyên tử: đọc rồi gán từ hai thread vẫn mất cập nhật nếu thiếu compare-and-set.

## Ví von
MVI giống cuốn sổ cái đóng dấu từng giao dịch, thay vì để nhiều người cùng tẩy xóa trực tiếp trên số dư.

## Khuôn mẫu chuẩn
```kotlin
data class UiState(val isLoading: Boolean = false, val data: String = "")
sealed interface UiIntent { data object Load : UiIntent }

class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    fun onIntent(intent: UiIntent) = when (intent) {
        is UiIntent.Load -> _state.update { it.copy(isLoading = true) }
    }
}
```

## Bẫy & ranh giới
Bẫy lớn nhất là đưa sự kiện vào State mà quên xoá sau khi xử lý, khiến Toast lặp lại theo [State vs Event](concept:architecture-state-vs-event). Dùng data class chứa cờ Boolean vẫn sinh trạng thái vừa tải vừa lỗi; bạn phải dùng sealed interface để cấm tổ hợp này. Chống chỉ định ép buộc MVI cho màn hình tĩnh hoặc form một nguồn dữ liệu vì chỉ gây rác mã nguồn.

## Câu hỏi thực chiến
**Hỏi:** MVI khác MVVM thì có tác dụng thực sự gì?
Khác đúng hai chỗ: màn hình chỉ có một object State, và mọi thay đổi đi qua một cửa onIntent. Nhờ vậy log được từng dòng "Intent này sinh ra State kia", và muốn tái hiện bug thì phát lại đúng chuỗi Intent đó.

**Hỏi:** Có nhất thiết thay MVVM bằng MVI không?
Không. Nếu [ViewModel](concept:android-viewmodel) đã có đúng một `StateFlow<UiState>`, bạn có phần lớn lợi ích rồi — hướng dẫn chính thức của Android cũng chỉ đòi vậy. MVI chỉ thêm lớp Intent và hàm chuyển đổi trạng thái thuần túy.

**Hỏi:** Vì sao Compose phải dùng MVI?
Không phải. Compose chỉ đòi luồng một chiều: state chảy xuống, sự kiện chảy lên. Một UiState duy nhất hợp Compose vì @Preview hay test chỉ cần dựng một object giả, và MVVM hiện đại cũng làm được vậy.

**Hỏi:** Khác nhau có mấy đâu?
Đúng, trên Android hiện đại khác biệt rất nhỏ, nhiều cuộc cãi nhau thực ra là cãi tên gọi. Nói được điều này khi phỏng vấn là điểm cộng: nó cho thấy bạn nhìn vào ràng buộc thật chứ không chạy theo nhãn.

**Hỏi:** Màn hình này nên dùng MVVM hay MVI, làm sao biết?
Tuỳ vào số nguồn sự kiện và tổ hợp trạng thái cấm. Nhiều nguồn cùng đổ vào (người dùng gõ, socket, timer, kết quả mạng) và có tổ hợp phải cấm thì MVI trả lại công. Form hay [danh sách](concept:android-recyclerview-and-diffutil) một nguồn thì một UiState trong MVVM là đủ; app đã theo quy ước nào hãy giữ vững quy ước đó.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất: MVI là hai ràng buộc bổ sung đặt lên MVVM gồm một State duy nhất và một cửa onIntent duy nhất.
- [ ] Cơ chế an toàn đa luồng của hàm update dùng compare-and-set tránh mất mát cập nhật khi đọc-gán trên nhiều thread.
- [ ] Bẫy cờ Boolean sinh trạng thái lỗi vô lý và giải pháp triệt tiêu bằng sealed interface.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
