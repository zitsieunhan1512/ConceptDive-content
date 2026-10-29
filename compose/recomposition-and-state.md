---
id: compose-recomposition-and-state
title: Compose: Recomposition & State
domain: compose
tags: [compose, recomposition, state, remember, derived-state]
collections: [interview-fundamental]
related: [compose-side-effects, android-viewmodel, kotlin-stateflow-vs-sharedflow]
created: 2026-09-15
diagrams: [Cơ chế|recomposition-slot-table-gap-buffer.png|Duyệt bảng khe nhớ Slot Table qua composer, chỉ chạy lại RecomposeScope cục bộ và bỏ qua nếu Stable]
---

## Bản 30 giây
Khi bạn bấm nút tăng số đếm, Compose không vẽ lại cả màn hình mà chỉ gọi lại đúng hàm đọc biến `State` đó. Hàm `remember` giúp bạn giữ lại dữ liệu trong sổ nhớ `Slot Table` qua những lần hàm chạy lại. Nhờ vậy, chỉ phần giao diện có dữ liệu mới được cập nhật, còn các chỗ khác đứng yên.

## Cần biết trước
- android-viewmodel
- architecture-state-vs-event
- kotlin-stateflow-vs-sharedflow

## Vấn đề buộc nó ra đời
Năm 2008, hệ thống View của Android ra đời. Muốn sửa giao diện, bạn tự gọi `setText` hay `setVisibility` từ nhiều nơi trong Activity.

Khi màn hình phức tạp, chỉ cần quên gọi `setVisibility` ở một nhánh `if` là màn hình hiện sai so với ViewModel (*State drift*). Nhiều chỗ cùng sửa một View thì rất khó lần ra chỗ nào vừa làm hỏng giao diện.

Năm 2021, Google phát hành Jetpack Compose với cách viết giao diện khai báo (*Declarative UI*). Bạn chỉ mô tả màn hình trông thế nào với dữ liệu hiện tại, còn Compose tự lo cập nhật.

## Nó thực sự là cái gì
`Recomposition` không xoá màn hình rồi vẽ lại từ đầu. Lần chạy đầu, Compose ghi mọi thứ hàm của bạn tạo ra vào một cuốn sổ theo đúng thứ tự gọi, gọi là `Slot Table`. Khi một `State` đổi, Compose chỉ gọi lại những phạm vi hàm (`RecomposeScope`) đã đọc `State` đó, rồi so kết quả mới với sổ cũ. Chỗ nào khác với sổ cũ thì mới sửa trên màn hình.

## Cơ chế
Khi biên dịch, Compose thêm tham số `$composer` vào mọi hàm `@Composable` để đọc và ghi từng mục trong sổ `Slot Table`. Sổ chừa sẵn một khoảng trống (*Gap Buffer*) nên thêm hay bớt mục ở giữa không phải dời cả mảng. Gặp `remember`, `$composer` cất giá trị vào đúng ô để lần chạy sau lấy lại. Hàm con có tham số không đổi thì được bỏ qua, còn `derivedStateOf` chỉ báo chạy lại khi kết quả điều kiện đổi.

## Đánh đổi
Viết giao diện như hàm đọc dữ liệu giúp bạn tránh cảnh màn hình hiện một đằng, ViewModel lưu một nẻo. Đổi lại, ứng dụng tốn thêm bộ nhớ để giữ cuốn sổ `Slot Table` và tốn công theo dõi xem hàm nào đọc `State` nào.

Nếu mỗi lần recompose bạn tạo đối tượng mới ngay trong thân hàm, hàm con nhận nó sẽ không được bỏ qua.

## Nguyên lý gốc
Muốn đổi một chữ hay ẩn một nút, bạn chỉ sửa dữ liệu gốc, không tự tay tìm View để sửa.

## Key insight
> Recomposition không vẽ lại điểm ảnh trên màn hình, mà chỉ gọi lại các hàm đọc `State` mới để cập nhật `Slot Table`.

## Khuôn mẫu chuẩn
```kotlin
@Composable
fun TodoListScreen(listState: LazyListState) {
    val showScrollToTop by remember(listState) {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }
    if (showScrollToTop) {
        ScrollToTopButton(onClick = { /* cuộn lên đầu */ })
    }
}
```

## Bẫy & ranh giới
Nếu bạn gán giá trị mới cho `State` ngay trong thân hàm Composable, hàm sẽ tự gọi lại chính nó mãi không dừng (*Recomposition loop*). Khi mở Layout Inspector, bạn sẽ thấy số lần recompose tăng vọt liên tục và khung hình bị giật. Cũng đừng đọc `State` đổi liên tục ở hàm cha bọc cả màn hình, vì cả màn hình sẽ chạy lại theo. Gọi API thì đặt vào [side effect](concept:compose-side-effects), không gọi thẳng trong thân hàm.

## Câu hỏi thực chiến
**Hỏi:** Có thể dùng remember có truyền key thay thế hoàn toàn cho derivedStateOf được không?
Không nên khi dữ liệu đầu vào thay đổi từng chút một như vị trí cuộn danh sách. Nếu truyền vị trí cuộn làm key cho `remember`, hàm chạy lại mỗi lần vị trí đổi, dù nút cuộn lên đầu chưa cần đổi. Còn `derivedStateOf` chỉ báo cho hàm chạy lại đúng lúc bạn cuộn qua mốc mục thứ 5.

**Hỏi:** Bỏ quên remember khi khai báo mutableStateOf có làm sập ứng dụng không?
Ứng dụng không sập, nhưng cứ mỗi lần hàm chạy lại là biến của bạn bị tạo mới và quay về giá trị ban đầu. Ví dụ bạn đang gõ dở chữ vào ô tìm kiếm mà màn hình recompose, chữ vừa gõ sẽ biến mất ngay.

**Hỏi:** Có phải compiler Compose luôn tự động bỏ qua việc vẽ lại hàm Composable nếu dữ liệu không đổi?
Không hẳn, và từ Kotlin 2.0.20 luật đã đổi vì chế độ *strong skipping* bật mặc định. Tham số không ổn định như `List` được bỏ qua nếu là đúng đối tượng cũ (so `===`), tham số ổn định thì so `equals`. Hàm con hay chạy lại thừa nhất khi bạn tạo đối tượng mới mỗi lần recompose, ví dụ gọi `listOf(...)` trong thân hàm cha.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách Compose ghi cây giao diện vào sổ Slot Table và chỉ gọi lại RecomposeScope đã đọc State vừa đổi.
- [ ] Nêu được vai trò của tham số $composer, khoảng trống Gap Buffer và cách remember giữ giá trị qua các lần chạy.
- [ ] Phân biệt được cơ chế bỏ qua hàm bằng === và equals trong strong skipping, cũng như cách dùng derivedStateOf.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
