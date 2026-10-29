---
id: architecture-mvvm-vs-mvi
title: MVVM vs MVI
domain: architecture
tags: [architecture, mvvm, mvi, unidirectional-data-flow, state-management]
collections: [interview-fundamental]
related: [android-state-holders, android-viewmodel, architecture-state-vs-event]
created: 2026-09-07
sources:
  - UI layer — Android App Architecture | https://developer.android.com/topic/architecture/ui-layer
  - MutableStateFlow.update API reference — Kotlin | https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/update.html
  - Model–view–viewmodel — Wikipedia | https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
diagrams: [Cơ chế|mvi-unidirectional-data-flow.png|Luồng dữ liệu một chiều khép kín: UI phát Intent lên một cửa duy nhất và nhận State bất biến cập nhật qua CAS]
---

## Bản 30 giây
Khi [ViewModel](concept:android-viewmodel) giữ nhiều biến trạng thái rời rạc, màn hình rất dễ hiện các trạng thái chỏi nhau. MVI đặt thêm hai quy tắc: gom hết vào một `State` bất biến và nhận sự kiện qua một hàm duy nhất. Nhờ vậy, mọi thay đổi trên màn hình đều quy về đúng một chỗ, giúp bạn dễ tìm lỗi và dựng lại kịch bản lỗi.

## Cần biết trước
- android-state-holders

## Vấn đề buộc nó ra đời
Trên Android, MVI được đưa vào để trị cảnh dữ liệu nằm rải rác của lối viết MVVM cũ. Trong MVVM cũ, một [ViewModel](concept:android-viewmodel) thường mở ra năm bảy biến `LiveData` hoặc `StateFlow` riêng lẻ cho từng góc màn hình.

Khi màn hình lớn dần, nhiều coroutine và hàm cùng sửa các biến này một lúc nên rất dễ lệch nhau. Màn hình có thể rơi vào cảnh vừa quay vòng báo đang tải, vừa hiện chữ báo lỗi mất mạng.

Lúc muốn tìm xem ai vừa sửa biến, bạn phải lần qua hàng chục hàm nằm rải rác khắp `ViewModel`. Không có một chỗ giữ trạng thái chung khiến việc tái hiện lại lỗi trên máy dev cực kỳ vất vả.

## Nó thực sự là cái gì
MVI không phải là một kiến trúc mới tách rời hay đập bỏ MVVM. Nó là MVVM cộng thêm hai quy tắc: gom màn hình vào một `State` bất biến và nhận thao tác qua một hàm `Intent`. Ở tầng bộ nhớ, `StateFlow` chỉ giữ một con trỏ tới đối tượng trên Heap. Mỗi lần đổi trạng thái là nó trỏ sang một đối tượng `State` mới.

## Cơ chế
Khi bạn bấm nút, giao diện gửi một `Intent` vào hàm nhận duy nhất để tính ra đối tượng `State` mới từ `State` cũ. `ViewModel` gọi `_state.update { it.copy(...) }` để chạy vòng lặp so sánh và đổi con trỏ nguyên tử (CAS - compare-and-set). Cách này chỉ ghi khi chưa có luồng khác chen ngang, và tự tính lại nếu có luồng vừa cập nhật trước. Ngược lại, nếu bạn viết `_state.value = _state.value.copy(...)` trên hai luồng chạy song song, luồng chạy sau sẽ đè mất dữ liệu của luồng chạy trước.

## Đánh đổi
MVI giúp nhìn rõ từng bước đổi trạng thái để tái hiện lỗi, nhưng phải viết thêm nhiều lớp `Intent`. Nếu hàm Compose ngoài cùng đọc cả đối tượng `State` lớn, nhiều phần sẽ chạy lại thừa khi một trường đổi. Ngoài ra, nếu xếp hàng xử lý từng `Intent` nối đuôi nhau, lệnh mạng chậm sẽ làm các cú bấm sau phải đợi.

## Nguyên lý gốc
Luồng dữ liệu một chiều: trạng thái chỉ chảy từ `ViewModel` xuống màn hình. Ngược lại, thao tác của người dùng chỉ đi từ màn hình ngược lên.

## Key insight
> `State` bất biến không tự chống mất dữ liệu đa luồng: hai luồng cùng gán vẫn đè nhau nếu thiếu `update` (compare-and-set).

## Ví von
MVI giống cuốn sổ cái ngân hàng ghi nối tiếp từng dòng giao dịch. Nó không để nhiều nhân viên cùng lấy tẩy xóa thẳng lên con số tổng.

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
Bẫy hay gặp là nhét thông báo vào `State` mà quên xóa, làm `Toast` bật lại khi xoay máy theo [State vs Event](concept:architecture-state-vs-event). Bẫy thứ hai là dùng `data class` chứa nhiều biến `Boolean` rời làm vừa `isLoading = true` vừa `isError = true`. Bạn hãy đổi sang `sealed interface` để khóa các nhánh loại trừ nhau. Đừng ép dùng MVI cho màn hình tĩnh hay form đơn giản vì chỉ đẻ thêm hàng đống lớp `Intent` thừa.

## Câu hỏi thực chiến
**Hỏi:** MVI khác MVVM thì có tác dụng thực sự gì?
Khác đúng hai chỗ: cả màn hình chỉ có một đối tượng `State`, và mọi thao tác bấm đều đi qua một cửa `onIntent`. Nhờ vậy bạn in log được từng dòng "gửi `Intent` này thì ra `State` kia". Khi có lỗi, bạn chỉ cần phát lại đúng chuỗi `Intent` đó là tái hiện được ngay.

**Hỏi:** Có nhất thiết thay MVVM bằng MVI không?
Không cần. Nếu [ViewModel](concept:android-viewmodel) đã gom về một `StateFlow<UiState>`, bạn đã có lợi ích lớn nhất và tài liệu Android cũng chỉ khuyên tới đó. MVI chỉ đi thêm một bước là gom hết các hàm public thành một lớp `Intent` đi qua một hàm xử lý.

**Hỏi:** Vì sao Compose phải dùng MVI?
Không bắt buộc. Compose chỉ cần luồng dữ liệu một chiều: `State` đi xuống, sự kiện bấm đi lên. Một `UiState` chung hợp với Compose vì viết `@Preview` hay test chỉ cần tạo một đối tượng giả, giống hệt MVVM hiện đại.

**Hỏi:** Khác nhau có mấy đâu?
Đúng vậy, trên Android hiện nay hai cách viết này khác nhau rất ít, chủ yếu chỉ khác cách đặt tên. Nói thẳng điểm này khi phỏng vấn là điểm cộng lớn, vì nó cho thấy bạn hiểu cơ chế thật chứ không học vẹt.

**Hỏi:** Màn hình này nên dùng MVVM hay MVI, làm sao biết?
Tuỳ vào số nguồn bắn sự kiện vào màn hình và các trạng thái không được phép xuất hiện cùng lúc. Nếu màn hình nhận tin cùng lúc từ người bấm, socket và mạng, lại cần khóa các trạng thái xung đột thì nên dùng MVI. Còn với form hay [danh sách](concept:android-recyclerview-and-diffutil) một nguồn thì một `UiState` trong MVVM là vừa đẹp, cứ giữ thống nhất theo chuẩn dự án.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được MVI chỉ là MVVM cộng thêm hai quy tắc: một đối tượng State duy nhất và một hàm nhận onIntent duy nhất.
- [ ] Trình bày được vì sao phải dùng hàm update (compare-and-set) thay vì đọc rồi gán trực tiếp khi nhiều luồng cùng sửa State.
- [ ] Chỉ ra được bẫy dùng nhiều cờ Boolean rời trong data class gây trạng thái mâu thuẫn và cách khóa lại bằng sealed interface.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
