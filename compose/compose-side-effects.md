---
id: compose-side-effects
title: Compose: Side Effects
domain: compose
tags: [compose, side-effects, launched-effect, coroutines, lifecycle]
collections: [interview-fundamental]
related: [compose-recomposition-and-state, kotlin-coroutines, android-activity-lifecycle]
created: 2026-09-15
sources:
  - Side-effects in Compose — Android Developers | https://developer.android.com/develop/ui/compose/side-effects
  - Lifecycle of composables — Android Developers | https://developer.android.com/develop/ui/compose/lifecycle
diagrams: [Cơ chế|compose-lifecycle-side-effects.png|Ba điểm neo vòng đời: LaunchedEffect theo cây, DisposableEffect dọn dẹp và rememberCoroutineScope cho sự kiện]
---

## Bản 30 giây
Nếu đặt lệnh gọi API thẳng vào thân hàm Composable của màn hình hồ sơ, mỗi lần vẽ lại là gọi mạng thêm một lần. Compose cho bạn các khối side effect để gắn những việc ngoài giao diện vào vòng đời của màn hình. Khi người dùng rời màn hình, các coroutine và bộ lắng nghe bên trong cũng tự động hủy theo.

## Cần biết trước
- compose-recomposition-and-state
- kotlin-coroutines
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Trong Compose, hàm giao diện có thể chạy lại nhiều lần do [Recomposition](concept:compose-recomposition-and-state), chạy bất cứ lúc nào hoặc bị bỏ qua. Vì vậy, lệnh gọi mạng hay đăng ký bộ lắng nghe đặt ngay trong thân hàm sẽ chạy lặp lại không kiểm soát được.

Compose 1.0 (2021) ra mắt kèm sẵn `LaunchedEffect`, `DisposableEffect` và `SideEffect` để đặt những việc đó vào chỗ có vòng đời rõ ràng. Nhờ gắn với cây giao diện, các việc này chỉ chạy đúng lúc cần và tự dọn dẹp khi màn hình đóng lại.

## Nó thực sự là cái gì
Side effect không phải là chỗ để bạn nhét đại các đoạn code chạy ngầm. Chúng là những điểm neo vòng đời (*Lifecycle anchors*) được ghi trong sổ `Slot Table` của Compose. Nhờ vậy, Compose biết khi nào thành phần xuất hiện để bắt đầu việc, khi nào biến mất để hủy việc.

## Cơ chế
Khi thành phần bước vào màn hình, `LaunchedEffect` mở một coroutine và tự hủy nó rồi chạy lại mỗi khi tham số `key` đổi. Nếu muốn mở coroutine lúc người dùng bấm nút, bạn lấy phạm vi từ `rememberCoroutineScope` để gọi bên trong hàm xử lý nút bấm. Còn `DisposableEffect` dành cho thứ cần gỡ bỏ như bộ lắng nghe, với khối `onDispose` bắt buộc chạy khi đổi `key` hoặc rời màn hình.

## Đánh đổi
Đặt việc gọi mạng và lắng nghe vào đúng khối side effect giúp bạn không quên hủy chúng khi đóng màn hình. Đổi lại, bạn phải chọn đúng tham số `key` cho từng khối. Thiếu `key` thì màn hình giữ mãi dữ liệu cũ, còn `key` đổi liên tục thì tác vụ cứ bị hủy rồi chạy lại.

## Nguyên lý gốc
Hàm vẽ giao diện chỉ nên làm việc vẽ. Mọi việc chạm ra bên ngoài phải gắn vào vòng đời của phần giao diện đó.

## Key insight
> Side effect không phải nơi chạy ngầm tự do, mà là điểm neo gắn công việc bên ngoài vào vòng đời của phần giao diện.

## Khuôn mẫu chuẩn
```kotlin
@Composable
fun UserProfileScreen(userId: String, viewModel: ProfileViewModel) {
    val scope = rememberCoroutineScope()
    LaunchedEffect(userId) { viewModel.loadProfile(userId) }
    DisposableEffect(userId) {
        onDispose { viewModel.clearCache() }
    }
    Button(onClick = { scope.launch { viewModel.saveProfile() } }) {
        Text("Lưu hồ sơ")
    }
}
```

## Bẫy & ranh giới
Nếu bạn truyền cố định `LaunchedEffect(Unit)` cho chỗ cần tải lại theo `userId`, đổi người dùng xong màn hình vẫn hiện hồ sơ cũ. Quên gỡ bộ lắng nghe trong `onDispose` sẽ giữ chặt đối tượng trên bộ nhớ Heap gây rò rỉ. Đừng dùng `LaunchedEffect` để xử lý lúc người dùng bấm nút mà hãy dùng `rememberCoroutineScope`. Khi truyền nhầm `key` đổi liên tục, màn hình sẽ nháy hoài, gọi API lặp lại hoặc dừng giữa chừng với `JobCancellationException`.

## Câu hỏi thực chiến
**Hỏi:** Có thể gọi trực tiếp LaunchedEffect bên trong callback khi người dùng bấm nút không?
Không được. `LaunchedEffect` là một hàm `@Composable` nên chỉ gọi được ngay trong thân hàm vẽ giao diện, không gọi được bên trong `onClick`. Muốn chạy coroutine khi bấm nút, bạn lấy `scope` từ `rememberCoroutineScope` rồi gọi `scope.launch` trong `onClick`.

**Hỏi:** Dùng LaunchedEffect(Unit) có đảm bảo tác vụ chỉ chạy đúng một lần trong đời ứng dụng không?
Không. Nó chỉ chạy một lần trong suốt thời gian thành phần đó còn nằm trên màn hình. Nếu người dùng chuyển sang màn hình khác rồi quay lại, thành phần được tạo mới nên `LaunchedEffect(Unit)` sẽ chạy lại từ đầu.

**Hỏi:** Có bắt buộc phải cung cấp khối onDispose bên trong DisposableEffect không?
Bắt buộc. Nếu bạn quên viết `onDispose` ở cuối `DisposableEffect`, trình biên dịch sẽ báo lỗi ngay lúc gõ code. Khối này giúp bạn chắc chắn có chỗ gỡ bỏ bộ lắng nghe hay `BroadcastReceiver` khi thành phần rời khỏi màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách LaunchedEffect tự hủy coroutine cũ và chạy coroutine mới mỗi khi key đổi.
- [ ] Phân biệt được việc dùng LaunchedEffect khi vào màn hình với dùng rememberCoroutineScope trong callback bấm nút.
- [ ] Nêu được vai trò bắt buộc của khối onDispose trong DisposableEffect để gỡ bộ lắng nghe khi rời màn hình.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
