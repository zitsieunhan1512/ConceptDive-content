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
Mọi thao tác bấm chạm và vẽ giao diện trên Android đều chạy trên một luồng chính duy nhất. Nếu luồng này bị chặn bởi việc đọc tệp hoặc tải mạng, toàn bộ màn hình sẽ bị đơ cứng. Hệ thống sẽ bật hộp thoại cảnh báo và buộc dừng ứng dụng nếu sự chậm trễ vượt quá ngưỡng cho phép.

## Cần biết trước
- android-handler-looper-message-queue
- thread

## Vấn đề buộc nó ra đời
Ngay từ Android 1.0 (2008), Android áp dụng kiến trúc đơn luồng cho giao diện người dùng. Mọi lệnh vẽ màn hình đều phải đi qua hàng đợi tin nhắn của luồng chính. Khi lập trình viên thực hiện tác vụ nặng trên luồng này, ứng dụng hoàn toàn không phản hồi thao tác bấm.

Để bảo vệ trải nghiệm, hệ điều hành bổ sung cơ chế giám sát và kích hoạt lỗi treo ứng dụng. Người dùng thấy một hộp thoại thông báo và có quyền đóng ứng dụng ngay lập tức. Tuy nhiên, việc phát hiện dòng mã gây tắc nghẽn ở giai đoạn phát triển vẫn rất khó khăn.

Tháng 12 năm 2010, kỹ sư Brad Fitzpatrick giới thiệu công cụ StrictMode trên Android 2.3 Gingerbread. Công cụ này lập tức cảnh báo khi phát hiện việc đọc ghi đĩa hoặc gọi mạng trên luồng chính.

## Nó thực sự là cái gì
Lỗi ngừng phản hồi KHÔNG phải là do điện thoại bị thiếu RAM hay sập luồng.

Thực chất, đó là đòn ngắt bảo vệ của hệ thống khi [hàng đợi tin nhắn](concept:android-handler-looper-message-queue) bị tắc nghẽn.

Hệ điều hành giám sát các mốc thời gian phản hồi thông qua dịch vụ hệ thống.

Nếu luồng chính không hoàn tất sự kiện trong ngưỡng quy định, hệ thống sẽ kích hoạt thông báo treo máy.

## Cơ chế
Sự kiện chạm màn hình hoặc bấm phím sẽ kích hoạt lỗi nếu không xử lý trong 5 giây.

`BroadcastReceiver` có ngưỡng 5 giây nếu app có Activity mở; ngưỡng cụ thể thay đổi theo bản Android.

`Service` của app phải chạy xong `onCreate` và `onStartCommand` trong khoảng 20 giây, theo mặc định AOSP.

Hệ thống lưu vết vào `traces.txt` ở bản cũ hoặc nhiều tệp `anr_*` ở bản mới, lấy qua `adb bugreport`.

## Đánh đổi
Việc đẩy tác vụ sang luồng phụ đòi hỏi xử lý đồng bộ và chuyển dữ liệu phức tạp hơn.

Bù lại, luồng chính luôn giữ được tốc độ vẽ cao và phản hồi tức thì với người dùng.

## Nguyên lý gốc
Nguyên lý giao diện đơn luồng: luồng trực tiếp tương tác với người dùng phải luôn được giải phóng khỏi mọi tác vụ chặn.

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
Gọi mạng trên luồng chính gây sập ngay với `NetworkOnMainThreadException`, còn đọc ghi tệp gây giật hoặc ANR. Bật công cụ StrictMode trên bản phát hành chính thức sẽ làm giảm hiệu năng thực tế của người dùng. Khi gặp lỗi treo máy, triệu chứng điển hình là nhật ký hệ thống xuất hiện thẻ ActivityManager kèm tệp nhật ký truy vết.

## Câu hỏi thực chiến
**Hỏi:** Vì sao một tác vụ đọc file chỉ tốn vài mili-giây trên Main thread vẫn có thể gây ra lỗi ANR?
Thời gian đọc tệp không cố định vì phụ thuộc vào tải của bộ nhớ flash. Khi bộ nhớ flash đang bận, thao tác này có thể kéo dài hàng giây và chặn đứng luồng chính. Do đó tác vụ vào ra đĩa không bao giờ được coi là an toàn trên luồng giao diện.

**Hỏi:** Các ngưỡng thời gian kích hoạt ANR phổ biến nhất trên Android là bao lâu?
Ngưỡng chạm là 5 giây, BroadcastReceiver là 5 giây khi có Activity ở tiền cảnh, và `startForeground` trong 5 giây. Vòng đời Service mặc định theo mã nguồn AOSP là 20 giây, tài liệu mô tả là vài giây. Vượt quá các mốc này, hệ thống sẽ kích hoạt thông báo ứng dụng ngừng phản hồi.

**Hỏi:** Làm thế nào để điều tra nguyên nhân gốc rễ khi ứng dụng bị lỗi ANR trong sản xuất?
Kiểm tra báo cáo Android Vitals trên Google Play Console hoặc trích xuất tệp traces bằng lệnh adb bugreport. Tệp vết này lưu toàn bộ trạng thái và vị trí dòng mã của luồng chính tại đúng thời điểm bị treo. Điều này giúp lập trình viên xác định chính xác dòng lệnh đang phong toả luồng.

## Ý bắt buộc khi giải thích lại
- [ ] Kiến trúc đơn luồng của giao diện và nguyên nhân gây nghẽn hàng đợi tin nhắn.
- [ ] Các ngưỡng thời gian kích hoạt ANR đối với đầu vào, BroadcastReceiver và dịch vụ.
- [ ] Cách sử dụng StrictMode để phát hiện sớm các tác vụ truy cập đĩa hoặc mạng.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

