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
Màn hình Fragment chứa hai thành phần có tuổi thọ khác nhau: đối tượng điều khiển và cây giao diện hiển thị. Khi chuyển trang nhưng giữ lại trong ngăn xếp, phần giao diện bị huỷ trong khi đối tượng điều khiển vẫn sống. Lập trình viên phải dùng `viewLifecycleOwner` và giải phóng biến giao diện để tránh rò rỉ bộ nhớ.

## Cần biết trước
- android-activity-lifecycle
- android-memory-leak

## Vấn đề buộc nó ra đời
Tháng 2 năm 2011, kỹ sư Dianne Hackborn giới thiệu Fragment trên Android 3.0 Honeycomb. Mục tiêu là chia nhỏ giao diện để hiển thị linh hoạt trên cả điện thoại và máy tính bảng. Tuy nhiên, cơ chế lưu Fragment trong ngăn xếp điều hướng đã tạo ra lỗ hổng kiến trúc lớn.

Khi đưa vào ngăn xếp, cây giao diện bị huỷ nhưng thể hiện Fragment vẫn tồn tại trong RAM. Lập trình viên thời kỳ đầu chỉ có thể dùng chính Fragment làm chủ thể vòng đời khi theo dõi dữ liệu. Khi người dùng quay lại, hàm tạo giao diện chạy lại và sinh thêm người quan sát mới.

Tháng 9 năm 2018, AndroidX Fragment 1.0.0 bổ sung thuộc tính `viewLifecycleOwner`. Giải pháp này cung cấp một vòng đời riêng biệt gắn chặt với sự tồn tại thực tế của giao diện.

## Nó thực sự là cái gì
Fragment KHÔNG phải là một khung nhìn giao diện đơn lẻ.

Thực chất, nó là bộ điều phối nắm giữ hai chu kỳ sống lồng nhau: cá thể và giao diện.

Vòng đời giao diện chỉ bắt đầu tại `onCreateView` và kết thúc hoàn toàn tại `onDestroyView`.

Trong khi đó, thể hiện Fragment sống từ `onCreate` tới khi bị loại bỏ ở `onDestroy`.

## Cơ chế
Khi Fragment vào ngăn xếp lùi, hệ thống gọi `onDestroyView` để huỷ cây View.

Nếu theo dõi dữ liệu qua `viewLifecycleOwner`, hệ thống sẽ tự động gỡ bỏ bộ lắng nghe khi View bị huỷ.

Biến ViewBinding giữ tham chiếu mạnh tới cây giao diện nên phải gán null trong `onDestroyView`.

Nếu không dọn dẹp, toàn bộ cấu trúc View cồng kềnh sẽ bị rò rỉ trên bộ nhớ Heap.

## Đánh đổi
Lập trình viên phải viết thêm mã dọn dẹp biến liên kết ViewBinding trong `onDestroyView`.

Bù lại, thiết bị tiết kiệm dung lượng bộ nhớ lớn khi lưu trữ nhiều màn hình trong ngăn xếp.

## Nguyên lý gốc
Nguyên lý tách biệt tuổi thọ: thành phần tiêu tốn nhiều tài nguyên phải có phạm vi sống ngắn hơn bộ điều khiển logic.

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
Quan sát LiveData bằng `this` sẽ kích hoạt sự kiện lặp lại khi người dùng quay lại màn hình. Quên gán `_binding = null` trong `onDestroyView` sẽ giữ chặt cây View trên Heap gây rò rỉ bộ nhớ. Truy cập thuộc tính `binding` qua toán tử `_binding!!` sau `onDestroyView` sẽ ném ngoại lệ `NullPointerException`.

## Câu hỏi thực chiến
**Hỏi:** Điều gì sẽ xảy ra nếu đăng ký lắng nghe LiveData trong onViewCreated bằng this thay vì viewLifecycleOwner?
Bộ lắng nghe sẽ bị đăng ký trùng lặp nhiều lần. Khi Fragment nằm trong ngăn xếp lùi, cá thể Fragment vẫn tồn tại nhưng View cũ đã bị huỷ. Mỗi lần quay lại, một bộ lắng nghe mới được nạp thêm khiến logic cập nhật chạy lặp lại bất thường.

**Hỏi:** Tại sao phải gán biến _binding về null trong onDestroyView?
Để tránh rò rỉ toàn bộ cây giao diện trong bộ nhớ. Khi Fragment vào ngăn xếp, đối tượng Fragment vẫn sống nhưng View đã kết thúc vòng đời. Nếu giữ lại tham chiếu binding, bộ thu gom rác không thể giải phóng các View con.

**Hỏi:** Có thể dùng lifecycleScope của Fragment để thu thập Flow cập nhật giao diện không?
Không nên thu thập trực tiếp mà không gắn với vòng đời View. Nên dùng `viewLifecycleOwner.lifecycleScope` kết hợp `repeatOnLifecycle` để tự động ngắt khi View bị ẩn. Cách này ngăn việc cập nhật dữ liệu vào những View không còn tồn tại trên màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Sự lệch pha tuổi thọ giữa cá thể Fragment và cây View khi ở trong ngăn xếp lùi.
- [ ] Lý do bắt buộc dùng viewLifecycleOwner thay cho this khi quan sát dữ liệu giao diện.
- [ ] Cơ chế rò rỉ bộ nhớ nếu không giải phóng biến ViewBinding trong onDestroyView.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

