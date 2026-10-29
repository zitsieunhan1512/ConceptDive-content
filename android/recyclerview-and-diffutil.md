---
id: android-recyclerview-and-diffutil
title: RecyclerView & DiffUtil
domain: android
tags: [android, ui, performance, list]
collections: [interview-fundamental]
related: [android-memory-leak, compose-recomposition-and-state]
created: 2026-10-01
sources:
  - Create dynamic lists with RecyclerView | https://developer.android.com/develop/ui/views/layout/recyclerview
  - DiffUtil API reference — Android Developers | https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil
  - ListAdapter API reference — Android Developers | https://developer.android.com/reference/androidx/recyclerview/widget/ListAdapter
diagrams: [Cơ chế|recyclerview-viewholder-recycling.png|Ô cuộn tràn màn hình đưa vào bộ đệm tái chế để ô mới dùng lại View cũ, chỉ gọi onBindViewHolder nạp dữ liệu]
---

## Bản 30 giây
Khi bạn cuộn danh sách dài hàng ngàn mục, điện thoại sẽ hết sạch RAM nếu tạo riêng từng View một. `RecyclerView` chỉ tạo vừa đủ số ô hiện trên màn hình rồi lấy lại ô vừa cuộn khuất để gắn dữ liệu mới. Đi cùng nó, `DiffUtil` so sánh danh sách cũ và mới để chỉ vẽ lại đúng những ô vừa đổi.

## Cần biết trước
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2009 tại Google I/O, kỹ sư Romain Guy chỉ ra `ListView` rất dễ bị giật khi bạn cuộn danh sách. Nhiều lập trình viên tạo View mới liên tục và gọi `findViewById` ở mỗi dòng, làm bộ thu gom rác chạy dồn dập. Dù mẫu `ViewHolder` đã được khuyên dùng, `ListView` không bắt buộc viết theo và cũng không hỗ trợ hoạt ảnh khi thêm bớt mục.

Tháng 10 năm 2014, Google đưa `RecyclerView` vào Support Library 21.0.0 và bắt buộc bạn phải dùng `ViewHolder`. Tuy vậy, khi dữ liệu đổi, gọi `notifyDataSetChanged` vẫn bắt mọi ô trên màn hình phải nạp lại từ đầu. Cách này làm mất hoạt ảnh chuyển động và tốn công vẽ lại cả những ô không đổi.

Tháng 8 năm 2016, Support Library 24.2.0 bổ sung thêm `DiffUtil` dựa trên thuật toán của Eugene Myers năm 1986. Công cụ này tự tìm ra đúng mục nào thêm, bớt hay đổi chỗ để chỉ cập nhật những ô đó.

## Nó thực sự là cái gì
`RecyclerView` không tự mình ôm hết mọi việc như `ListView` ngày xưa, mà chia rõ cho ba thành phần phối hợp. `LayoutManager` tính vị trí các ô, `Adapter` gắn dữ liệu, còn `ViewHolder` giữ tham chiếu tới các View con. Sang Jetpack Compose, `LazyColumn` cũng giữ nguyên cách làm này, và bạn cần truyền `key` để định danh từng mục.

## Cơ chế
Khi bạn cuộn làm một ô trôi khuất khỏi mép màn hình, `RecyclerView` cất nó vào kho tái chế thay vì xóa bỏ. Ô mới trồi lên ở mép kia sẽ lấy lại chính View cũ đó và chỉ gọi `onBindViewHolder` để điền dữ liệu mới. Khi danh sách thay đổi, `DiffUtil` so hai danh sách cũ mới để tìm ra chuỗi thao tác thêm, xóa hoặc dời chỗ ngắn nhất. Nếu bạn dùng `ListAdapter`, phép tính `DiffUtil` sẽ tự chạy trên luồng nền rồi mới gửi kết quả về luồng chính để sửa giao diện.

## Đánh đổi
`DiffUtil` tốn thêm thời gian CPU để so sánh, tùy theo độ dài danh sách và số mục bị thay đổi. Đổi lại, bạn không phải tạo lại hàng loạt View và màn hình không bị nháy trắng mỗi lần cập nhật dữ liệu.

## Nguyên lý gốc
Chỉ tạo đủ đối tượng giao diện cho phần đang hiển thị, cộng thêm vài ô dự phòng ở hai đầu.

## Key insight
> `notifyDataSetChanged` báo rằng mọi ô đều có thể đã đổi, nên bắt buộc phải nạp lại tất cả và làm mất hoạt ảnh.

## Khuôn mẫu chuẩn
```kotlin
class UserAdapter : ListAdapter<User, UserViewHolder>(UserDiffCallback) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        UserViewHolder(ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) =
        holder.bind(getItem(position))
}

object UserDiffCallback : DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(oldItem: User, newItem: User) = oldItem.id == newItem.id
    override fun areContentsTheSame(oldItem: User, newItem: User) = oldItem == newItem
}
```

## Bẫy & ranh giới
Nếu bạn tự chạy `DiffUtil` ngay trên luồng chính với danh sách hàng ngàn mục, màn hình sẽ khựng lại và dễ dính lỗi [ANR](concept:android-main-thread-and-anr). Sang `LazyColumn` của Compose, nếu bạn quên truyền `key`, các ô sẽ giữ nhầm trạng thái cũ khi có mục chèn lên đầu danh sách. Cũng đừng dùng `RecyclerView` cho màn hình tĩnh có vài mục cố định, vì phải viết nhiều lớp phụ thừa.

## Câu hỏi thực chiến
**Hỏi:** Nếu areItemsTheSame trả về true nhưng areContentsTheSame trả về false thì RecyclerView làm gì với ViewHolder?
`RecyclerView` hiểu đây vẫn là mục đó nhưng nội dung bên trong đã đổi, nên gọi lại `onBindViewHolder`. Nếu bạn có trả về `payload`, nó dùng lại ngay `ViewHolder` đang đứng đó để sửa đúng chỗ vừa đổi. Nếu thiếu `payload`, bộ dựng hoạt ảnh có thể mượn thêm một `ViewHolder` để làm hiệu ứng mờ dần.

**Hỏi:** Tại sao gọi notifyDataSetChanged lại bị xem là giải pháp tồi tệ trong RecyclerView?
Lệnh này không chỉ ra ô nào vừa đổi, nên `RecyclerView` đành phải gọi `onBindViewHolder` lại cho mọi ô trên màn hình. Tất cả hoạt ảnh thêm, sửa, xóa đều bị hủy bỏ. Với danh sách dài hoặc ô có bố cục nặng, bạn sẽ thấy màn hình khựng lại rõ rệt.

**Hỏi:** Quên truyền tham số key trong LazyColumn của Jetpack Compose gây ra lỗi gì?
Khi không có `key`, Compose đành lấy số thứ tự vị trí của ô trong danh sách làm khóa định danh. Lúc thêm hoặc xóa mục ở đầu, các ô dưới bị lệch thứ tự nên nhận nhầm trạng thái nhau. Người dùng sẽ thấy ô đánh dấu hoặc chữ đang gõ nhảy sang dòng khác.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách ViewHolder giữ tham chiếu View con và được tái sử dụng khi cuộn để không phải gọi findViewById lại.
- [ ] Phân biệt được việc nạp lại toàn bộ của notifyDataSetChanged với việc chỉ cập nhật đúng ô thay đổi nhờ DiffUtil.
- [ ] Nêu được vai trò của tham số key trong LazyColumn để giữ đúng trạng thái khi danh sách thêm bớt phần tử.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

