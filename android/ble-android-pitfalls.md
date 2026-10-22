---
id: android-ble-android-pitfalls
title: BLE Android Pitfalls
domain: android
tags: [android, ble, bluetooth, concurrency, pitfalls]
collections: [interview-fundamental]
related: [android-ble-gatt-model, kotlin-coroutines]
created: 2026-09-21
diagrams: [Cơ chế|ble-command-queue-busy.png|Giao thức ATT bật cờ mDeviceBusy từ chối lệnh tiếp theo, callback Binder mở khóa và close giải phóng slot native]
---

## Bản 30 giây
Tầng Bluetooth của Android chỉ có thể xử lý một thao tác tại một thời điểm cho mỗi kết nối. Bắn nhiều lệnh cùng lúc sẽ khiến hệ thống âm thầm hủy bỏ lệnh mà không báo lỗi. Bạn bắt buộc phải xây dựng hàng đợi để thực thi tuần tự từng tác vụ một cách an toàn.

## Cần biết trước
- android-ble-gatt-model
- kotlin-coroutines

## Vấn đề buộc nó ra đời
Năm 2013, Google đưa API Bluetooth Low Energy vào Android 4.3 Jelly Bean. Tầng giao tiếp Java bọc ngoài thư viện gốc C++ qua cầu nối Binder liên tiến trình.

Thời điểm đó, lập trình viên quen với mạng bất đồng bộ đã gọi đồng thời nhiều lệnh đọc ghi trên một kết nối. Giao thức ATT lại chỉ cho phép một yêu cầu chưa có phản hồi trên mỗi kênh truyền. Tầng Java Android không có hàng đợi tự động, khiến các thao tác gọi dồn dập bị nuốt chửng âm thầm.

Các lệnh gọi tiếp theo bị loại bỏ, khiến quá trình trao đổi dữ liệu rơi vào bế tắc. Hàng loạt mã lỗi bí ẩn xuất hiện mà không hề có tài liệu giải thích rõ ràng.

## Nó thực sự là cái gì
KHÔNG PHẢI do phần cứng vô tuyến không thể truyền nhận nhiều luồng dữ liệu. Thực chất, giới hạn này bắt nguồn từ giao thức ATT: chỉ một yêu cầu được phép chờ phản hồi trên một kết nối. Android cài đặt quy tắc này bằng cờ `mDeviceBusy` trong `BluetoothGatt` của từng kết nối.

## Cơ chế
Khi một thao tác ATT đang chạy trên kết nối, cờ `mDeviceBusy` bật khiến lệnh tiếp theo trên [BluetoothGatt](concept:android-ble-gatt-model) bị từ chối ngay. Callback phản hồi mặc định chạy trên luồng Binder IPC, trừ khi truyền custom Handler vào `connectGatt` từ Android 8. Hàm `disconnect()` chỉ ngắt liên kết sóng, còn `close()` mới giải phóng slot client trong stack Bluetooth native. Mã `status 133` là lỗi `GATT_ERROR` chung do rớt sóng hoặc timeout, nhưng rò rỉ client native sẽ làm kết nối hỏng vĩnh viễn.

## Đánh đổi
Tuân thủ nghiêm ngặt giao thức ATT giúp ngăn ngừa mất mát và sai lệch trạng thái gói tin. Đổi lại, lập trình viên buộc phải tự dựng hàng đợi tuần tự bằng [Coroutines](concept:kotlin-coroutines) để xâu chuỗi mọi tác vụ. Nếu không xếp hàng, các lệnh ghi dữ liệu gửi dồn dập sẽ bị tầng Java từ chối âm thầm mà không ném lỗi.

## Nguyên lý gốc
Nguyên lý tuần tự hóa giao thức ATT: mỗi kênh truyền chỉ được phép tồn tại một yêu cầu chưa hoàn tất.

## Key insight
> Android BluetoothGatt chỉ nhận một lệnh tại một thời điểm: gọi lệnh mới trước khi callback cũ trả về sẽ bị từ chối âm thầm.

## Ví von
Như một buồng điện thoại công cộng một ống nghe. Bạn phải chờ người trước cúp máy hoàn toàn thì hệ thống mới cho phép nhấc máy quay số cuộc gọi tiếp theo.

## Khuôn mẫu chuẩn
```kotlin
// Hang doi tuan tu hoa cac lenh GATT bang Mutex
class BleQueue(private val gatt: BluetoothGatt) {
    private val mutex = Mutex()

    suspend fun write(char: BluetoothGattCharacteristic, data: ByteArray) = mutex.withLock {
        char.value = data
        gatt.writeCharacteristic(char)
        // Cho callback tren luong Binder phat tin hieu xong truoc khi mo khoa
    }
}
```

## Bẫy & ranh giới
Quên gọi `close()` sau khi ngắt kết nối làm cạn kiệt slot client native, khiến mọi lần kết nối sau đều thất bại. Từ Android 7, quét quá 5 lần trong 30 giây sẽ bị hệ thống bóp nghẽn và trả về kết quả rỗng. Từ Android 12, quét và kết nối bắt buộc xin quyền runtime `BLUETOOTH_SCAN` và `BLUETOOTH_CONNECT`. Triệu chứng hỏng hóc là hàm ghi trả về false hoặc callback nhận mã `status 133` liên tục.

## Câu hỏi thực chiến
**Hỏi:** Mã lỗi status 133 trong onConnectionStateChange là gì và xử lý thế nào?
Đó là lỗi `GATT_ERROR` chung do mất sóng, rớt gói tin hoặc cạn kiệt slot client native. Bạn bắt buộc phải gọi `gatt.close()` ngay lập tức và chờ ít nhất 500ms trước khi thử kết nối lại.

**Hỏi:** Tại sao các hàm callback của BluetoothGatt lại không chạy trên Main thread?
Vì Android nhận tín hiệu từ Bluetooth stack rồi bắn sang ứng dụng qua luồng Binder IPC. Muốn callback chạy trên luồng chỉ định, bạn có thể truyền Handler vào hàm `connectGatt` từ Android 8.0.

**Hỏi:** Tham số autoConnect trong hàm connectGatt nên đặt true hay false?
Đặt false khi người dùng chủ động kết nối để Android quét tích cực và kết nối nhanh trong vài trăm mili-giây. Đặt true cho các tác vụ kết nối lại dưới nền không giới hạn thời gian.

**Hỏi:** Quên gọi close() sau khi disconnect() gây hậu quả nghiêm trọng gì?
Nó gây rò rỉ slot client trong stack Bluetooth native. Vượt giới hạn slot sẽ khiến các lần gọi `connectGatt` sau trả về null hoặc báo lỗi.

## Ý bắt buộc khi giải thích lại
- [ ] Giới hạn một thao tác do quy tắc giao thức ATT, bảo vệ bởi cờ mDeviceBusy theo từng kết nối.
- [ ] Sự khác biệt sống còn giữa disconnect() ngắt sóng và close() giải phóng slot client native.
- [ ] Callback chạy trên Binder thread trừ khi truyền Handler vào connectGatt từ Android 8.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
