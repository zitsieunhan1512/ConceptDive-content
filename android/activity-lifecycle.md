---
id: android-activity-lifecycle
title: Activity Lifecycle
domain: android
tags: [android, mobile, operating-system, lifecycle, memory-management]
collections: [interview-fundamental]
related: [android-memory-leak, android-viewmodel]
created: 2026-09-07
sources:
  - The activity lifecycle — Android Developers | https://developer.android.com/guide/components/activities/activity-lifecycle
  - Low memory killer daemon — AOSP | https://source.android.com/docs/core/perf/lmkd
  - Activity API reference — Android Developers | https://developer.android.com/reference/android/app/Activity
diagrams: [Cơ chế|activity-lifecycle-process-death.png|Activity che khuất lưu Bundle, LMKD tăng điểm oom_score_adj và gửi SIGKILL khi cạn RAM]
---

## Bản 30 giây
Khi mở ứng dụng khác đè lên mà thiếu RAM, hệ điều hành sẽ tắt ứng dụng cũ để lấy bộ nhớ. Vòng đời là bộ hàm báo trước giữa ứng dụng và hệ điều hành. Nhờ đó, bạn kịp ngắt định vị và lưu dữ liệu màn hình trước khi nhân hệ điều hành thu hồi tiến trình.

## Cần biết trước

## Vấn đề buộc nó ra đời
Tháng 10/2008, HTC Dream ra mắt với vỏn vẹn 192MB RAM. Chip sóng và hệ điều hành chiếm phần lớn, chỉ để lại chưa đầy 100MB cho ứng dụng.

Thiết bị di động không thể dùng bộ nhớ swap trên ổ đĩa vì chip nhớ flash quá chậm và nhanh hỏng. Trong khi đó, cuộc gọi đến có thể ập tới bất ngờ và cần RAM trống ngay lập tức.

Kỹ sư Android đưa Low Memory Killer vào kernel để gửi tín hiệu `SIGKILL` tắt thẳng tiến trình ngầm. Họ tạo ra lifecycle để ứng dụng kịp lưu trạng thái, giúp người dùng mở lại thấy màn hình vẫn như chưa từng bị tắt.

## Nó thực sự là cái gì
Activity Lifecycle không phải chuỗi hàm tiện ích để bạn sắp xếp việc nạp giao diện. Thực ra, nó là quy ước báo hiệu giữa ứng dụng và hệ điều hành để nhường trả tài nguyên. Cơ chế này giúp nhân hệ điều hành thu hồi RAM bất cứ lúc nào mà người dùng mở lại vẫn thấy màn hình cũ.

## Cơ chế
Khi màn hình của bạn bị che khuất, hệ thống gọi `onPause` rồi `onStop` và lưu dữ liệu tạm qua `onSaveInstanceState`. Lúc này, hệ điều hành tăng điểm `oom_score_adj` (mức ưu tiên bị tắt trong bộ dọn LMKD) của tiến trình lên cao. Khi máy thiếu RAM, LMKD gửi lệnh `SIGKILL` tắt ngay tiến trình có điểm cao nhất mà không gọi `onDestroy`. Khi người dùng mở lại ứng dụng, Android tạo tiến trình mới và truyền lại `Bundle` vào `onCreate` để dựng lại màn hình.

## Đánh đổi
Hệ thống chạy ổn định ngay cả trên những máy ít RAM. Đổi lại, bạn phải tự viết code lưu trạng thái màn hình và cẩn thận tránh [rò rỉ bộ nhớ](concept:android-memory-leak). Mỗi lần người dùng xoay ngang điện thoại, Activity bị hủy rồi tạo lại nên các việc đang chạy dở rất dễ bị ngắt.

## Nguyên lý gốc
Giữ cảm giác ứng dụng chạy liên tục bằng cách cất dữ liệu vào bản chụp tạm trước khi hủy. Khi người dùng quay về, hệ thống dựng lại màn hình từ bản chụp đó.

## Key insight
> Lifecycle báo trước để hệ điều hành thu hồi RAM ứng dụng ngầm bất cứ lúc nào mà người dùng không biết.

## Ví von
Khi bạn trả phòng tạm để đi chơi xa, khách sạn cất đồ của bạn vào chiếc vali nhỏ rồi cho khách khác thuê phòng. Khi bạn quay lại, nhân viên xếp đồ từ vali ra một căn phòng mới giống hệt phòng cũ.

## Khuôn mẫu chuẩn
```kotlin
// Observer
class LocationListener(private val gpsManager: GpsManager) : DefaultLifecycleObserver {
    override fun onStart(owner: LifecycleOwner) = gpsManager.startUpdates()
    override fun onStop(owner: LifecycleOwner) = gpsManager.stopUpdates()
}

// Activity
lifecycle.addObserver(LocationListener(gpsManager))
```

## Bẫy & ranh giới
Bẫy hay gặp nhất là tin rằng `onDestroy()` lúc nào cũng chạy khi ứng dụng đóng. Khi ứng dụng nằm dưới nền mà máy hết RAM, Low Memory Killer bắn lệnh `SIGKILL` tắt tiến trình ngay mà bỏ qua `onDestroy()`. Nếu bạn đợi tới `onDestroy` mới lưu bài viết đang soạn dở, người dùng sẽ mất sạch chữ vừa gõ. Cũng đừng gọi lệnh ghi file hay mạng lâu trong các hàm vòng đời, vì bị `SIGKILL` giữa chừng sẽ làm hỏng dữ liệu.

## Câu hỏi thực chiến
**Hỏi:** Gọi finish() thì Activity có bị hủy ngay lập tức không?
Không. Lệnh `finish()` chỉ báo cho hệ thống xếp lịch đóng màn hình, nên các dòng code viết ngay bên dưới nó vẫn chạy tiếp. Hệ thống chỉ thực sự gọi `onDestroy` ở các lượt xử lý tin nhắn sau trên Main Thread.

**Hỏi:** onPause với onStop khác nhau chỗ nào trong thực tế?
`onPause` chạy khi màn hình mất tương tác nhưng vẫn nhìn thấy, còn `onStop` chạy khi màn hình bị che kín. Ví dụ khi một hộp thoại xin quyền trong suốt bật đè lên, Activity phía dưới chỉ vào `onPause` chứ chưa vào `onStop`. Bạn nên dừng hoạt ảnh nhẹ ở `onPause`, và ngắt GPS hay camera nặng ở `onStop`.

**Hỏi:** Lưu trạng thái màn hình nên chọn onSaveInstanceState hay ViewModel, làm sao biết?
Tuỳ vào việc bạn muốn giữ dữ liệu qua lúc xoay màn hình hay qua lúc hệ thống tắt ngầm tiến trình. [ViewModel](concept:android-viewmodel) giữ được danh sách lớn trong RAM khi xoay máy, nhưng sẽ mất hết nếu tiến trình bị hệ thống tắt. Vì vậy, bạn phải dùng thêm `onSaveInstanceState` để cất các ID hoặc chữ đang gõ vào `Bundle` nhằm tải lại màn hình.

**Hỏi:** Vì sao Google khuyên giải phóng tài nguyên ở onStop thay vì onDestroy?
Vì hệ thống không hứa sẽ luôn gọi `onDestroy`. Khi ứng dụng của bạn lùi về nền, hệ thống có thể gửi `SIGKILL` tắt thẳng tiến trình mà không chạy qua `onDestroy`. Ngắt định vị hay máy ảnh ngay ở `onStop` giúp trả lại phần cứng ngay khi người dùng không còn nhìn thấy màn hình.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được cách LMKD tăng điểm oom_score_adj và gửi SIGKILL tắt tiến trình nền mà không gọi onDestroy.
- [ ] Nêu được vai trò của lifecycle như cơ chế báo trước để ứng dụng lưu Bundle và nhường RAM cho hệ điều hành.
- [ ] Chỉ ra được bẫy mất dữ liệu khi lưu trong onDestroy và nguy cơ rò rỉ bộ nhớ khi xoay màn hình.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

