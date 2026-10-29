---
id: android-main-thread-and-anr
title: Main Thread & ANR
domain: android
tags: [android, performance, thread, anr]
collections: [interview-fundamental]
related: [android-handler-looper-message-queue, thread, kotlin-coroutine-context-and-dispatchers]
created: 2026-10-01
diagrams: [Cơ chế|anr-watchdog-thresholds.png|Hệ thống giám sát mốc thời gian: chạm 5s, Broadcast 5s, Service 20s; vượt ngưỡng kích hoạt ANR và lưu tệp traces]
---

## Bản 30 giây
Khi bạn bấm một nút trên màn hình, lệnh bấm và lệnh vẽ lại nút đều xếp hàng chạy trên một luồng chính duy nhất. Nếu bạn đọc file lớn hay gọi mạng ngay trên luồng này, các việc đứng sau sẽ kẹt cứng và màn hình đơ luôn. Quá hạn mà luồng chính chưa rảnh, Android sẽ bật bảng báo lỗi ANR cho người dùng đóng ứng dụng.

## Cần biết trước
- android-handler-looper-message-queue
- thread

## Vấn đề buộc nó ra đời
Từ bản Android 1.0 (2008), toàn bộ giao diện được thiết kế chạy trên một luồng chính duy nhất. Mọi việc vẽ màn hình hay nhận cú chạm đều phải xếp hàng qua hàng đợi của luồng này. Chỉ cần lập trình viên chạy một việc nặng trên luồng chính, màn hình sẽ đứng im không bấm được nữa.

Để người dùng không bị kẹt chết ở một ứng dụng đơ, Android đặt đồng hồ canh thời gian phản hồi. Hết giờ mà ứng dụng chưa trả lời, hệ thống hiện bảng báo không phản hồi để người dùng tắt đi. Nhưng lúc đang viết code, lập trình viên vẫn rất khó biết dòng nào vừa lỡ chặn luồng chính.

Tháng 12 năm 2010, kỹ sư Brad Fitzpatrick đưa công cụ `StrictMode` vào Android 2.3 Gingerbread. Công cụ này báo ngay trên log mỗi khi bạn đọc ghi đĩa hoặc gọi mạng trên luồng chính.

## Nó thực sự là cái gì
Lỗi ANR (*Application Not Responding*) không phải do máy hết RAM hay luồng bị sập. Đó là còi báo động của hệ thống khi [hàng đợi tin nhắn](concept:android-handler-looper-message-queue) trên luồng chính bị một việc nặng chặn đứng. Dịch vụ hệ thống luôn bấm giờ cho từng sự kiện giao diện hoặc vòng đời đang chạy. Hết hạn mà luồng chính chưa làm xong việc đó để nhận việc mới, Android sẽ bật hộp thoại ANR.

## Cơ chế
Sự kiện chạm màn hình hoặc bấm phím sẽ bị báo ANR nếu không xử lý xong trong 5 giây. Với `BroadcastReceiver`, hạn là 10 giây ở tiền cảnh (10–20 giây từ Android 14) và 60 giây ở nền, tùy bản Android. Còn `Service` phải chạy xong `onCreate` hay `onStartCommand` trong 20 giây ở tiền cảnh, hoặc 200 giây ở nền. Khi nổ ANR, hệ thống chụp lại stack trace vào `traces.txt` hoặc các tệp `anr_*`, lấy ra bằng lệnh `adb bugreport`.

## Đánh đổi
Đưa việc đọc file hay gọi mạng sang luồng phụ thì bạn phải thêm bước chuyển kết quả về luồng chính để vẽ lại. Đổi lại, luồng chính luôn rảnh tay để vẽ khung hình mượt và phản hồi ngay mỗi khi người dùng chạm vào.

## Nguyên lý gốc
Luồng đang trực tiếp vẽ màn hình và nghe tay người dùng bấm thì không được phép đứng chờ bất kỳ việc nặng nào.

## Key insight
> Lỗi ANR không xảy ra vì máy chậm, mà do luồng chính bị chiếm dụng bởi các tác vụ nặng.

## Khuôn mẫu chuẩn
```kotlin
class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            val policy = StrictMode.ThreadPolicy.Builder()
                .detectAll().penaltyLog().build()
            StrictMode.setThreadPolicy(policy)
        }
    }
}
```

## Bẫy & ranh giới
Gọi mạng trên luồng chính làm app sập ngay với `NetworkOnMainThreadException`, còn đọc file hay giải mã JSON lớn sẽ gây giật hoặc ANR. Đừng bật `StrictMode` ở bản phát hành cho người dùng vì việc theo dõi mọi lệnh đọc ghi sẽ làm máy chạy chậm đi. Khi dính ANR, bạn sẽ thấy mục `ActivityManager` báo lỗi trong logcat kèm tệp vết lưu đúng dòng code đang giữ chân luồng chính.

## Câu hỏi thực chiến
**Hỏi:** Vì sao một tác vụ đọc file chỉ tốn vài mili-giây trên Main thread vẫn có thể gây ra lỗi ANR?
Vì tốc độ đọc file phụ thuộc vào lúc đó chip nhớ flash đang rảnh hay bận. Khi máy đang cài ứng dụng khác hoặc ghi ảnh nền, lệnh đọc một file nhỏ cũng có thể phải chờ vài giây. Vì vậy mọi việc đọc ghi đĩa đều phải đưa sang luồng phụ.

**Hỏi:** Các ngưỡng thời gian kích hoạt ANR phổ biến nhất trên Android là bao lâu?
Chạm và `startForeground` có hạn 5 giây; `BroadcastReceiver` là 10 giây tiền cảnh (10–20 giây từ Android 14) và 60 giây ở nền. Với `Service`, hạn là 20 giây cho dịch vụ tiền cảnh và 200 giây cho dịch vụ nền, tùy bản Android. Quá các mốc này mà hàm chưa trả về, hệ thống sẽ báo ứng dụng ngừng phản hồi.

**Hỏi:** Làm thế nào để điều tra nguyên nhân gốc rễ khi ứng dụng bị lỗi ANR trong sản xuất?
Bạn mở mục Android Vitals trên Google Play Console hoặc chạy lệnh `adb bugreport` để lấy tệp vết `traces`. Tệp này ghi lại toàn bộ các hàm mà luồng chính đang chạy dở ngay lúc bị treo. Nhìn vào đỉnh stack trace của luồng `main`, bạn sẽ thấy ngay hàm nào đang đứng chờ khóa hay đọc đĩa.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao mọi việc vẽ và nhận chạm đi chung một hàng đợi trên luồng chính nên dễ bị kẹt.
- [ ] Nêu đúng các mốc thời gian báo ANR cho sự kiện chạm, BroadcastReceiver và Service ở tiền cảnh lẫn nền.
- [ ] Trình bày được cách bật StrictMode lúc debug và cách đọc tệp traces để tìm dòng code chặn luồng.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
