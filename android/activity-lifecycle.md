---
id: android-activity-lifecycle
title: Activity Lifecycle
domain: android
tags: [android, mobile, operating-system, lifecycle, memory-management]
collections: [interview-fundamental]
related: [android-memory-leak, android-viewmodel]
created: 2026-09-07
diagrams: [Cơ chế|activity-lifecycle-process-death.png|Activity che khuất lưu Bundle, LMKD tăng điểm oom_score_adj và gửi SIGKILL khi cạn RAM]
---

## Bản 30 giây
Thiết bị di động có RAM hạn chế và hệ điều hành sẵn sàng khai tử ứng dụng chạy ngầm để nhường tài nguyên. Vòng đời là giao thức thương lượng tài nguyên giữa ứng dụng và hệ điều hành. Nó báo cho ứng dụng kịp nhả tài nguyên và lưu trạng thái trước khi kernel can thiệp.

## Cần biết trước

## Vấn đề buộc nó ra đời
Tháng 10/2008, HTC Dream ra mắt với vỏn vẹn 192MB RAM. Chip sóng và hệ điều hành chiếm phần lớn, chỉ để lại chưa đầy 100MB cho ứng dụng.

Thiết bị di động không thể dùng bộ nhớ swap trên ổ đĩa vì chip nhớ flash quá chậm và nhanh hỏng. Đồng thời, cuộc gọi đến có thể ập tới bất ngờ và đòi hỏi RAM trống tức thì.

Kỹ sư Android đưa Low Memory Killer vào kernel để gửi tín hiệu `SIGKILL` tiêu diệt tiến trình ngầm. Họ tạo ra lifecycle để lưu trạng thái, tạo ảo ảnh ứng dụng vẫn chạy liên tục.

## Nó thực sự là cái gì
Activity Lifecycle KHÔNG phải chuỗi hàm tiện ích sắp xếp nạp giao diện. Thực chất, nó là giao thức thương lượng tài nguyên giữa ứng dụng và hệ điều hành. Cơ chế này cho phép kernel thu hồi RAM bất cứ lúc nào mà không làm đứt đoạn trải nghiệm.

## Cơ chế
Khi Activity bị che khuất, hệ thống gọi `onPause` rồi `onStop` và lưu trạng thái qua `onSaveInstanceState`. Hệ điều hành tăng điểm `oom_score_adj` (chỉ số ưu tiên bị diệt trong LMKD) của tiến trình lên mức cao. Khi cạn RAM, daemon LMKD gửi `SIGKILL` xóa sổ tiến trình có điểm cao nhất mà không hề gọi `onDestroy`. Khi mở lại, Android tạo tiến trình mới và nạp lại Bundle vào `onCreate` để phục hồi giao diện.

## Đánh đổi
Hệ thống giữ được độ ổn định cao trên phần cứng hạn chế. Đổi lại, lập trình viên phải tự xử lý việc lưu trữ trạng thái và đối mặt với nguy cơ [rò rỉ bộ nhớ](concept:android-memory-leak). Màn hình bị hủy rồi tạo lại sau mỗi lần xoay máy khiến luồng xử lý bất đồng bộ rất dễ đứt gãy.

## Nguyên lý gốc
Ảo ảnh tính liên tục: duy trì trải nghiệm liền mạch bằng cách hủy và dựng lại thành phần từ bản chụp trạng thái.

## Key insight
> Lifecycle là giao thức nhượng quyền để hệ điều hành giải phóng RAM bất cứ lúc nào mà người dùng không hề hay biết.

## Ví von
Khi khách rời phòng, khách sạn cất đồ đạc vào vali nhỏ rồi nhường phòng cho người khác. Khi khách quay lại, họ dọn đồ vào phòng mới y như cũ.

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
Bẫy tai hại nhất là tin rằng `onDestroy()` luôn luôn được gọi. Khi ở chế độ nền, Low Memory Killer bắn lệnh `SIGKILL` xóa sổ app ngay lập tức mà bỏ qua hoàn toàn `onDestroy()`. Lưu dữ liệu quan trọng trong `onDestroy` chắc chắn sẽ làm mất dữ liệu của người dùng. Chống chỉ định chạy tác vụ I/O dài hạn trong callback vì nếu bị SIGKILL thì công việc sẽ dở dang và hỏng dữ liệu.

## Câu hỏi thực chiến
**Hỏi:** Gọi finish() thì Activity có bị hủy ngay lập tức không?
Không. Lệnh `finish()` chỉ đánh dấu kết thúc và xếp lịch chuyển màn hình, các dòng mã phía sau lệnh gọi đó vẫn chạy tiếp. Hệ thống chỉ thực sự gọi `onDestroy` ở các chu kỳ thông điệp sau trên Main Thread.

**Hỏi:** onPause với onStop khác nhau chỗ nào trong thực tế?
`onPause` xảy ra khi màn hình mất tiêu điểm nhưng vẫn nhìn thấy, còn `onStop` chạy khi giao diện bị che khuất hoàn toàn. Hiện hộp thoại cấp quyền hệ thống dạng trong suốt là tình huống kinh điển chỉ kích hoạt `onPause`. Bạn nên tạm dừng hoạt ảnh nhẹ ở `onPause`, nhưng chỉ ngắt kết nối định vị nặng ở `onStop`.

**Hỏi:** Lưu trạng thái màn hình nên chọn onSaveInstanceState hay ViewModel, làm sao biết?
Tuỳ vào việc bạn muốn sống sót qua xoay màn hình hay qua biến cố bị hệ thống diệt tiến trình ngầm. [ViewModel](concept:android-viewmodel) giữ đối tượng bộ nhớ lớn qua việc xoay máy nhưng sẽ mất trắng khi tiến trình bị thu hồi. `onSaveInstanceState` bắt buộc dùng để lưu lại ID và cờ nhỏ vào Bundle nhằm phục hồi đúng màn hình cũ.

**Hỏi:** Vì sao Google khuyên giải phóng tài nguyên ở onStop thay vì onDestroy?
Vì `onDestroy` không có gì bảo đảm sẽ luôn được gọi. Khi ứng dụng về chế độ nền, hệ thống sẵn sàng gửi `SIGKILL` tiêu diệt tiến trình mà bỏ qua `onDestroy`. Giải phóng định vị hay máy ảnh ở `onStop` bảo đảm phần cứng được trả về ngay khi khuất tầm nhìn.

## Ý bắt buộc khi giải thích lại
- [ ] LMKD tăng điểm oom_score_adj và gửi SIGKILL xóa tiến trình nền mà không gọi onDestroy.
- [ ] Bản chất: giao thức thương lượng tài nguyên với OS, tạo ảo ảnh liên tục qua cơ chế lưu Bundle.
- [ ] Bẫy lưu dữ liệu trong onDestroy bị mất và bẫy rò rỉ bộ nhớ khi xoay máy.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

