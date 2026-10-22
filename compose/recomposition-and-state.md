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
Khi dữ liệu thay đổi, Compose đối chiếu bảng nhớ để tính toán và chỉ chạy lại đúng phạm vi hàm có dữ liệu mới. Đối tượng `State` phát tín hiệu cập nhật, còn `remember` giữ dữ liệu qua các lần gọi lại. Cơ chế này giúp màn hình tự động làm mới cục bộ mà không cần can thiệp thủ công.

## Cần biết trước
- android-viewmodel
- architecture-state-vs-event
- kotlin-stateflow-vs-sharedflow

## Vấn đề buộc nó ra đời
Năm 2008, hệ thống View của Android ra đời dựa trên mô hình thay đổi trạng thái trực tiếp. Lập trình viên phải gọi các lệnh như `setText` hay `setVisibility` từ nhiều nơi trong vòng đời Activity.

Khi ứng dụng phình to, dữ liệu trong ViewModel và giao diện thực tế rất dễ lệch pha nhau (*State drift*). Hai trạng thái cạnh tranh nhau gây ra hàng loạt lỗi sập ứng dụng và sai lệch hiển thị.

Năm 2021, Google phát hành Jetpack Compose để chuyển đổi hoàn toàn sang mô hình giao diện khai báo (*Declarative UI*). Giao diện giờ đây trở thành một hàm thuần túy biến đổi từ dữ liệu đầu vào thành hình ảnh hiển thị.

## Nó thực sự là cái gì
`Recomposition` KHÔNG phải là việc xóa sạch màn hình để vẽ lại toàn bộ các điểm ảnh từ đầu. Thực chất, đây là quá trình duyệt lại mảng bộ đệm khe hở (*Gap Buffer*) lưu trữ cây giao diện (*Slot Table*).

Trình chạy chỉ thực thi lại những phạm vi hàm (*RecomposeScope*) đã đọc giá trị của đối tượng `State` vừa thay đổi.

## Cơ chế
Trình biên dịch chèn tham số `$composer` để điều hướng con trỏ đọc ghi trên các nhóm khe nhớ (*Groups*). Khi chạy, hàm `remember` đọc và ghi giá trị vào các khe nhớ tuần tự trên bảng khe nhớ.

Nếu tham số đầu vào ổn định (*Stable*) và không đổi, hệ thống sẽ bỏ qua việc chạy lại hàm (*Smart Recomposition*). Khi dùng `derivedStateOf`, hệ thống chỉ kích hoạt vẽ lại nếu kết quả tính toán cuối cùng thực sự thay đổi giá trị.

## Đánh đổi
Mô hình khai báo giúp loại bỏ hoàn toàn lỗi lệch pha dữ liệu giữa logic nghiệp vụ và giao diện. Đổi lại, bạn phải trả giá bằng chi phí theo dõi trạng thái và bộ nhớ cho bảng khe nhớ.

Nếu tạo đối tượng không ổn định, hàm sẽ bị gọi lại liên tục và gây tụt giảm khung hình nghiêm trọng.

## Nguyên lý gốc
Nguyên lý giao diện khai báo: giao diện người dùng luôn là một hàm ánh xạ trực tiếp từ trạng thái hiện tại. Mọi thay đổi trên màn hình chỉ được phép bắt nguồn từ việc cập nhật một nguồn chân lý duy nhất.

## Key insight
> Recomposition không phải vẽ lại điểm ảnh trên màn hình, mà là tính toán lại cây giao diện dựa trên bảng khe nhớ.

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
Ghi đè State trực tiếp trong hàm Composable sẽ kích hoạt vòng lặp vẽ lại (*Recomposition loop*). Đọc State quá rộng gây lãng phí phạm vi vẽ lại và tụt giảm tốc độ khung hình. Chống chỉ định khởi tạo đối tượng nặng hoặc gọi tác vụ bất đồng bộ thay vì dùng [hiệu ứng lề](concept:compose-side-effects). Triệu chứng gồm đóng băng ứng dụng (*ANR*), giật lag do áp lực dọn rác (*GC pause*) hoặc rò rỉ bộ nhớ.

## Câu hỏi thực chiến
**Hỏi:** Có thể dùng remember có truyền key thay thế hoàn toàn cho derivedStateOf được không?
Không nên. Truyền State làm key khiến hàm Composable đọc trực tiếp và bị kích hoạt vẽ lại liên tục dù kết quả điều kiện chưa đổi. Ngược lại, `derivedStateOf` chỉ phát tín hiệu vẽ lại khi giá trị tính toán cuối cùng thực sự thay đổi.

**Hỏi:** Bỏ quên remember khi khai báo mutableStateOf có làm sập ứng dụng không?
Không làm sập app ngay, nhưng làm mất trắng dữ liệu mỗi khi màn hình vẽ lại. Trạng thái bị tạo mới liên tục trên Heap và đưa về giá trị mặc định ban đầu. Mọi thao tác nhập liệu của người dùng sẽ bị xóa sạch bất thình lình.

**Hỏi:** Có phải compiler Compose luôn tự động bỏ qua việc vẽ lại hàm Composable nếu dữ liệu không đổi?
Tuỳ vào tính ổn định của các tham số truyền vào hàm. Nếu toàn bộ tham số đều bất biến hoặc được đánh dấu `@Stable`, compiler sẽ cho phép bỏ qua hàm. Nếu có tham số không ổn định như List thông thường, hàm sẽ luôn bị thực thi lại.

## Ý bắt buộc khi giải thích lại
- [ ] Recomposition chỉ duyệt lại Slot Table và chạy RecomposeScope đã đọc State thay đổi.
- [ ] Compiler Compose inject tham số $composer để quản lý các khe nhớ tuần tự.
- [ ] Phân biệt remember(key) và derivedStateOf để tránh kích hoạt tính toán dư thừa.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

