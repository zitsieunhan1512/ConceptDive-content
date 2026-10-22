---
id: android-recyclerview-and-diffutil
title: RecyclerView & DiffUtil
domain: android
tags: [android, ui, performance, list]
collections: [interview-fundamental]
related: [android-memory-leak, compose-recomposition-and-state]
created: 2026-10-01
diagrams: [Cơ chế|recyclerview-viewholder-recycling.png|Ô cuộn tràn màn hình đưa vào bộ đệm tái chế để ô mới dùng lại View cũ, chỉ gọi onBindViewHolder nạp dữ liệu]
---

## Bản 30 giây
Khi hiển thị danh sách dài hàng ngàn mục, điện thoại sẽ cạn kiệt bộ nhớ nếu tạo từng ô xem riêng biệt. RecyclerView giải quyết vấn đề này bằng cách chỉ tạo đủ số ô vừa màn hình và tái sử dụng chúng khi cuộn. Đi kèm với đó, DiffUtil tính toán sự khác biệt giữa hai danh sách để chỉ vẽ lại đúng những ô bị đổi.

## Cần biết trước
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2009 tại Google I/O, kỹ sư Romain Guy cảnh báo về hiệu năng kém của ListView khi người dùng cuộn danh sách. Lập trình viên liên tục tạo mới View và duyệt cây giao diện bằng `findViewById`. Việc này khiến bộ thu gom rác chạy dồn dập và gây giật khung hình. Dù mẫu ViewHolder thủ công được đề xuất, ListView vẫn không bắt buộc áp dụng và thiếu khả năng hỗ trợ ảnh động.

Tháng 10 năm 2014, Google phát hành RecyclerView trong Support Library 21.0.0 để bắt buộc dùng ViewHolder ở tầng giao diện lập trình. Tuy nhiên, lệnh `notifyDataSetChanged` ép buộc gán lại dữ liệu cho mọi ô đang hiển thị. Điều này vô hiệu hóa các hiệu ứng chuyển động và gây lãng phí công vẽ.

Tháng 8 năm 2016, thư viện Support Library 24.2.0 bổ sung DiffUtil dựa trên thuật toán Eugene Myers năm 1986. Công cụ này tính toán chính xác phần tử thêm, bớt hoặc di chuyển để cập nhật giao diện mượt mà.

## Nó thực sự là cái gì
RecyclerView KHÔNG phải là một khung nhìn đơn lẻ tự quản lý mọi việc như ListView cũ.

Thực chất, nó tách rời ba việc: tái sử dụng View, sắp xếp bố cục và phát hiện dữ liệu đổi.

LayoutManager tính vị trí ô, Adapter nạp dữ liệu, còn ViewHolder giữ tham chiếu trực tiếp tới View con.

Trong Jetpack Compose, thành phần `LazyColumn` cũng kế thừa triết lý này và bạn nên truyền `key` để định danh.

## Cơ chế
Khi một mục cuộn tràn khỏi màn hình, nó được đưa vào bộ nhớ đệm tái chế thay vì bị huỷ bỏ.

Ô tiếp theo xuất hiện sẽ dùng lại View cũ và chỉ nạp dữ liệu mới qua hàm gán.

DiffUtil so sánh hai danh sách để tìm ra các thao tác chèn, xoá hoặc dời vị trí ngắn nhất.

Thư viện ListAdapter tự động đẩy phép tính DiffUtil sang luồng nền và gửi kết quả cập nhật về luồng chính.

## Đánh đổi
Chi phí tính toán của DiffUtil phụ thuộc vào cả kích thước danh sách lẫn số lượng phần tử thay đổi. Bù lại, ứng dụng tiết kiệm được chi phí nạp lại bố cục và loại bỏ hiện tượng nháy trắng danh sách.

## Nguyên lý gốc
Nguyên lý tái sử dụng tài nguyên: chỉ cấp phát lượng đối tượng đủ cho vùng hiển thị thực tế cộng một khoảng đệm nhỏ.

## Key insight
> `notifyDataSetChanged` báo mọi ô có thể đã đổi, nên phải bind lại tất cả và mất hoạt ảnh.

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
Tính toán DiffUtil trên luồng chính với danh sách lớn sẽ gây giật khung hình và dẫn đến [ANR](concept:android-main-thread-and-anr). Trong `LazyColumn` của Compose, quên truyền tham số `key` sẽ khiến giao diện giữ nhầm trạng thái cũ khi danh sách đảo lộn. Không cần dùng RecyclerView cho các danh sách tĩnh chỉ có vài phần tử cố định vì cấu trúc lớp quá cồng kềnh.

## Câu hỏi thực chiến
**Hỏi:** Nếu areItemsTheSame trả về true nhưng areContentsTheSame trả về false thì RecyclerView làm gì với ViewHolder?
RecyclerView ghi nhận ô đổi nội dung và gọi lại hàm `onBindViewHolder`. Nếu có payload, ViewHolder cũ được bind tại chỗ; nếu không có, animator mặc định có thể tạo thêm ViewHolder để mờ chéo. Cơ chế này giúp tối ưu hoá việc tái sử dụng giao diện.

**Hỏi:** Tại sao gọi notifyDataSetChanged lại bị xem là giải pháp tồi tệ trong RecyclerView?
Lệnh này không cho biết ô nào đổi, nên RecyclerView phải bind lại mọi ô đang hiển thị. Mọi hoạt ảnh thêm, sửa, xoá đều bị bỏ qua. Danh sách dài hoặc ô phức tạp sẽ thấy giật rõ khi cập nhật.

**Hỏi:** Quên truyền tham số key trong LazyColumn của Jetpack Compose gây ra lỗi gì?
Hệ thống sẽ lấy chỉ số vị trí làm khoá nhận diện mặc định. Khi danh sách bị thay đổi ở đầu, các ô giao diện sẽ giữ sai trạng thái nội tại. Người dùng sẽ thấy dữ liệu hiển thị lộn xộn.

## Ý bắt buộc khi giải thích lại
- [ ] Cơ chế tái sử dụng của ViewHolder để tránh việc duyệt cây giao diện liên tục.
- [ ] Tác hại của notifyDataSetChanged so với các tín hiệu cập nhật cục bộ từ DiffUtil.
- [ ] Vai trò của tham số key trong LazyColumn để bảo toàn trạng thái phần tử.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

