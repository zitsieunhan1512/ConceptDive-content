---
id: compose-side-effects
title: Compose: Side Effects
domain: compose
tags: [compose, side-effects, launched-effect, coroutines, lifecycle]
collections: [interview-fundamental]
related: [compose-recomposition-and-state, kotlin-coroutines, android-activity-lifecycle]
created: 2026-09-15
diagrams: [Cơ chế|compose-lifecycle-side-effects.png|Ba điểm neo vòng đời: LaunchedEffect theo cây, DisposableEffect dọn dẹp và rememberCoroutineScope cho sự kiện]
---

## Bản 30 giây
Đặt tác vụ bất đồng bộ vào thân hàm giao diện sẽ khiến chúng bị kích hoạt lặp lại mỗi khi vẽ lại. Compose cung cấp các điểm neo vòng đời để gắn coroutine chặt chẽ vào cây hiển thị. Các tác vụ ngầm sẽ tự động bị hủy bỏ ngay khi thành phần giao diện biến mất để tránh rò rỉ bộ nhớ.

## Cần biết trước
- compose-recomposition-and-state
- kotlin-coroutines
- android-activity-lifecycle

## Vấn đề buộc nó ra đời
Trong Compose, các hàm giao diện có thể bị gọi lại nhiều lần do [Recomposition](concept:compose-recomposition-and-state) hoặc bị hủy giữa chừng. Nếu đặt lệnh gọi mạng hay lắng nghe sự kiện vào thân hàm, chúng sẽ bị kích hoạt lặp lại liên tục.

Điều này dẫn tới thảm họa gọi API vô tận, hao pin, rò rỉ bộ nhớ và làm sập ứng dụng. Trước năm 2021, nhiều dự án Compose thử nghiệm gặp lỗi nghiêm trọng do không kiểm soát được tác vụ bất đồng bộ.

Google thiết kế bộ công cụ quản lý hiệu ứng phụ để gắn chặt vòng đời của Coroutine vào cây giao diện Composition. Các tác vụ giờ đây được tự động dọn dẹp mỗi khi thành phần giao diện biến mất.

## Nó thực sự là cái gì
Hiệu ứng phụ KHÔNG phải là nơi để lập trình viên tùy tiện nhét các đoạn mã chạy ngầm. Thực chất, chúng là các điểm neo vòng đời (*Lifecycle anchors*) được quản lý bởi bảng khe nhớ của Compose.

Chúng điều khiển việc khởi tạo, hủy bỏ và dọn dẹp các tác vụ bất đồng bộ tương ứng với cây giao diện.

## Cơ chế
`LaunchedEffect` khởi chạy một coroutine độc lập ngay khi thành phần giao diện đi vào cây Composition. Mỗi khi khóa định danh (*key*) thay đổi, coroutine cũ sẽ bị hủy ngay lập tức và một coroutine mới được tạo ra.

`rememberCoroutineScope` cung cấp phạm vi coroutine để khởi chạy tác vụ từ các hàm phản hồi sự kiện. `DisposableEffect` kích hoạt khối dọn dẹp `onDispose` khi đổi khóa phụ thuộc hoặc khi thành phần rời khỏi cây hiển thị.

## Đánh đổi
Sử dụng đúng hiệu ứng phụ giúp ứng dụng giải phóng triệt để tài nguyên mạng và bộ nhớ. Đổi lại, việc quản lý khóa phụ thuộc đòi hỏi sự am hiểu sâu sắc về chu kỳ Recomposition.

Truyền sai khóa sẽ khiến tác vụ bị hủy và chạy lại liên tục, gây chớp nháy màn hình và nghẽn mạng.

## Nguyên lý gốc
Nguyên lý hiệu ứng phụ: giữ cho hàm hiển thị thuần túy và đẩy mọi thay đổi trạng thái ra ngoài. Vòng đời của tác vụ bất đồng bộ bắt buộc phải phụ thuộc chặt chẽ vào vòng đời của đối tượng sở hữu nó.

## Key insight
> Hiệu ứng lề không phải nơi chạy ngầm tự do, mà là chốt chặn neo tác vụ vào vòng đời hiển thị.

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
Dùng `LaunchedEffect(Unit)` cho tác vụ cần theo dõi tham số thay đổi sẽ khiến màn hình không bao giờ cập nhật dữ liệu mới. Bỏ quên lệnh dọn dẹp trong `onDispose` sẽ neo giữ các listener vĩnh viễn trên Heap, gây rò rỉ bộ nhớ nghiêm trọng. Chống chỉ định dùng `LaunchedEffect` để xử lý sự kiện click của người dùng thay vì dùng `rememberCoroutineScope`. Triệu chứng lỗi gồm màn hình nháy liên tục, gọi API lặp vô hạn hoặc crash với lỗi `JobCancellationException`.

## Câu hỏi thực chiến
**Hỏi:** Có thể gọi trực tiếp LaunchedEffect bên trong callback khi người dùng bấm nút không?
Không được. `LaunchedEffect` là hàm Composable chỉ chạy trong cây hiển thị, không thể gọi trong callback thông thường. Bạn bắt buộc phải dùng `rememberCoroutineScope` để khởi chạy coroutine từ sự kiện người dùng.

**Hỏi:** Dùng LaunchedEffect(Unit) có đảm bảo tác vụ chỉ chạy đúng một lần trong đời ứng dụng không?
Không. Nó chỉ chạy một lần trong suốt vòng đời thành phần giao diện đó nằm trên màn hình. Nếu người dùng điều hướng rời đi rồi quay lại, thành phần sẽ tái tạo và coroutine chạy lại từ đầu.

**Hỏi:** Có bắt buộc phải cung cấp khối onDispose bên trong DisposableEffect không?
Bắt buộc. Trình biên dịch sẽ báo lỗi cú pháp ngay lập tức nếu bạn thiếu khối `onDispose`. Khối này đảm bảo các tài nguyên như Listener hay BroadcastReceiver được dọn dẹp sạch sẽ khi rời cây giao diện.

## Ý bắt buộc khi giải thích lại
- [ ] LaunchedEffect tự động hủy coroutine cũ và chạy coroutine mới khi key thay đổi.
- [ ] rememberCoroutineScope chỉ dùng cho callback sự kiện do người dùng chủ động kích hoạt.
- [ ] DisposableEffect bắt buộc có onDispose để giải phóng listener ngoài cây Composition.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

