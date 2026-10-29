---
id: android-fragment-lifecycle
title: Fragment Lifecycle & View Lifecycle
domain: android
tags: [android, ui, lifecycle, memory-leak]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-memory-leak, android-viewmodel]
created: 2026-10-01
diagrams: [Cơ chế|fragment-vs-view-lifecycle.png|Khi vào backstack cây View bị huỷ ở onDestroyView trong khi Fragment vẫn sống, bắt buộc dùng viewLifecycleOwner và gán null binding]
---

## Bản 30 giây
Một `Fragment` có hai vòng đời riêng: vòng đời của đối tượng `Fragment` và vòng đời của cây `View` bên trong. Khi bạn bấm sang trang mới và đẩy `Fragment` cũ vào ngăn xếp lùi, cây `View` bị hủy nhưng đối tượng `Fragment` vẫn sống. Vì vậy, bạn phải dùng `viewLifecycleOwner` và gán biến `binding` về `null` để tránh rò rỉ bộ nhớ.

## Cần biết trước
- android-activity-lifecycle
- android-memory-leak

## Vấn đề buộc nó ra đời
Tháng 2 năm 2011, kỹ sư Dianne Hackborn giới thiệu `Fragment` trên Android 3.0 Honeycomb. Mục tiêu là chia nhỏ màn hình để ghép linh hoạt trên cả điện thoại và máy tính bảng. Nhưng cách Android cất `Fragment` vào ngăn xếp điều hướng đã đẻ ra một rắc rối lớn.

Khi bị đẩy vào ngăn xếp lùi, cây `View` bị hủy để nhường RAM nhưng đối tượng `Fragment` vẫn nằm lại trong bộ nhớ. Thời kỳ đầu, bạn chỉ có thể truyền chính `Fragment` làm chủ vòng đời khi đăng ký lắng nghe dữ liệu. Khi người dùng bấm nút Back quay lại, hàm tạo `View` chạy lại và gắn chồng thêm một bộ lắng nghe mới nữa.

Tháng 9 năm 2018, AndroidX Fragment 1.0.0 bổ sung thuộc tính `viewLifecycleOwner`. Nó tách riêng một vòng đời chỉ sống từ lúc tạo `View` tới lúc hủy `View`.

## Nó thực sự là cái gì
`Fragment` không phải là một khung giao diện đơn thuần. Thực ra, nó quản lý hai vòng đời lồng nhau: vòng đời đối tượng `Fragment` và vòng đời cây `View`. Vòng đời của cây `View` chỉ bắt đầu ở `onCreateView` và kết thúc ngay tại `onDestroyView`. Trong khi đó, đối tượng `Fragment` sống từ `onCreate` tới tận lúc bị gỡ hẳn ở `onDestroy`.

## Cơ chế
Khi `Fragment` lùi vào ngăn xếp, hệ thống gọi `onDestroyView` để dọn cây `View` khỏi RAM. Nếu bạn đăng ký lắng nghe dữ liệu qua `viewLifecycleOwner`, hệ thống sẽ tự gỡ bộ lắng nghe ngay khi `View` bị hủy. Riêng biến `ViewBinding` vẫn giữ tham chiếu tới cây `View` cũ nên bạn phải tự gán nó bằng `null` trong `onDestroyView`. Nếu quên bước này, cả cây `View` cồng kềnh sẽ bị giữ lại trên bộ nhớ Heap.

## Đánh đổi
Bạn phải tự tay viết thêm vài dòng gán biến `ViewBinding` về `null` bên trong hàm `onDestroyView`. Đổi lại, điện thoại tiết kiệm được rất nhiều RAM khi người dùng mở chồng nhiều màn hình trong ngăn xếp.

## Nguyên lý gốc
Phần giao diện tốn nhiều bộ nhớ phải được hủy sớm hơn đối tượng giữ trạng thái điều khiển nó.

## Key insight
> Vòng đời của Fragment luôn dài hơn vòng đời của View khi nằm trong ngăn xếp điều hướng.

## Khuôn mẫu chuẩn
```kotlin
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    private var _binding: FragmentProfileBinding? = null
    private val binding get() = _binding!!
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        _binding = FragmentProfileBinding.bind(view)
        viewModel.user.observe(viewLifecycleOwner) { binding.name.text = it.name }
    }
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

## Bẫy & ranh giới
Truyền `this` thay vì `viewLifecycleOwner` khi lắng nghe `LiveData` sẽ làm hàm cập nhật chạy lặp nhiều lần khi người dùng bấm quay lại. Quên gán `_binding = null` trong `onDestroyView` sẽ giữ chặt cây `View` cũ trên Heap và gây [rò rỉ bộ nhớ](concept:android-memory-leak). Còn nếu bạn cố đọc biến `binding` qua `_binding!!` sau khi `onDestroyView` đã chạy, ứng dụng sẽ sập vì lỗi `NullPointerException`.

## Câu hỏi thực chiến
**Hỏi:** Điều gì sẽ xảy ra nếu đăng ký lắng nghe LiveData trong onViewCreated bằng this thay vì viewLifecycleOwner?
Bộ lắng nghe sẽ bị gắn chồng lên nhau sau mỗi lần chuyển trang. Khi `Fragment` lùi vào ngăn xếp, đối tượng `Fragment` chưa chết nên bộ lắng nghe gắn với `this` không được gỡ. Khi bấm Back quay lại, `onViewCreated` chạy lại và gắn thêm bộ lắng nghe thứ hai, khiến sự kiện chạy hai lần.

**Hỏi:** Tại sao phải gán biến _binding về null trong onDestroyView?
Để bộ dọn rác thu hồi được cây giao diện vừa bị hủy. Khi `Fragment` nằm trong ngăn xếp lùi, đối tượng `Fragment` vẫn sống trên RAM dù các nút bấm và chữ nghĩa đã biến mất. Nếu biến `_binding` trên `Fragment` vẫn chỉ vào cây `View` cũ, bộ dọn rác sẽ không dọn được bất kỳ `View` con nào.

**Hỏi:** Có thể dùng lifecycleScope của Fragment để thu thập Flow cập nhật giao diện không?
Bạn không nên gọi trực tiếp trên `lifecycleScope` của `Fragment` vì nó vẫn chạy khi `View` đã bị hủy. Hãy gọi `viewLifecycleOwner.lifecycleScope` kèm `repeatOnLifecycle` để việc thu thập tự dừng khi màn hình khuất đi. Cách này giúp bạn không bao giờ gán nhầm dữ liệu vào những `View` đã bị dọn khỏi màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao đối tượng Fragment sống lâu hơn cây View khi nằm trong ngăn xếp lùi.
- [ ] Nêu được lý do phải dùng viewLifecycleOwner thay cho this khi lắng nghe dữ liệu trong onViewCreated.
- [ ] Trình bày được cách biến ViewBinding làm rò rỉ cây View nếu không gán về null ở onDestroyView.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

