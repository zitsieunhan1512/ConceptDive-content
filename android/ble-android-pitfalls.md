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
Khi giao tiếp Bluetooth với thiết bị ngoại vi, Android chỉ xử lý được đúng một lệnh tại một thời điểm trên mỗi kết nối. Nếu gọi hai lệnh đọc ghi liên tiếp ngay lập tức, hệ thống sẽ âm thầm bỏ qua lệnh thứ hai. Vì vậy, bạn bắt buộc phải xếp hàng các lệnh để chạy lần lượt từng lệnh một.

## Cần biết trước
- android-ble-gatt-model
- kotlin-coroutines

## Vấn đề buộc nó ra đời
Năm 2013, Google đưa API Bluetooth Low Energy vào Android 4.3 Jelly Bean. Tầng Java bọc ngoài thư viện C++ bên dưới và nói chuyện qua cầu nối Binder liên tiến trình.

Quen với cách gọi mạng bất đồng bộ, lập trình viên thường bắn cùng lúc nhiều lệnh đọc ghi trên một kết nối. Nhưng giao thức ATT chỉ cho phép một yêu cầu chờ phản hồi trên mỗi kênh truyền. Tầng Java của Android lại không tự xếp hàng, nên các lệnh gọi dồn dập bị bỏ qua âm thầm.

Hậu quả là các lệnh tiếp theo biến mất, khiến việc truyền dữ liệu bị treo giữa chừng. Hàng loạt mã lỗi khó hiểu như `status 133` xuất hiện mà tài liệu không giải thích rõ.

## Nó thực sự là cái gì
Giới hạn này không phải do con chip sóng không truyền nhận được nhiều luồng dữ liệu. Nó bắt nguồn từ quy định của giao thức ATT: trên một kết nối chỉ được có một yêu cầu đang chờ phản hồi. Android kiểm soát việc này bằng biến cờ `mDeviceBusy` bên trong mỗi đối tượng `BluetoothGatt`.

## Cơ chế
Khi một lệnh ATT đang chạy, biến cờ `mDeviceBusy` bật lên làm lệnh tiếp theo gọi vào [BluetoothGatt](concept:android-ble-gatt-model) trả về `false` ngay. Hàm callback báo kết quả mặc định chạy trên luồng Binder IPC, trừ khi bạn truyền `Handler` riêng vào `connectGatt` từ Android 8. Hàm `disconnect()` chỉ ngắt sóng, còn bạn phải gọi `close()` mới trả lại được chỗ đăng ký (slot client) trong tầng Bluetooth native. Mã `status 133` là lỗi `GATT_ERROR` chung do rớt sóng hoặc hết giờ, nhưng quên gọi `close()` sẽ làm kẹt hẳn các lần kết nối sau.

## Đánh đổi
Chặn gọi chồng lệnh giúp các gói tin truyền qua lại không bị lẫn lộn hay sai thứ tự. Đổi lại, bạn phải tự làm hàng đợi bằng [Coroutines](concept:kotlin-coroutines) để chạy từng lệnh một và chờ callback báo xong. Nếu bạn gọi liền hai lệnh ghi mà không chờ nhau, tầng Java sẽ lặng lẽ bỏ lệnh sau mà không ném ngoại lệ nào.

## Nguyên lý gốc
Trên mỗi kênh giao thức ATT, bạn phải chờ yêu cầu trước trả kết quả xong rồi mới được gửi yêu cầu tiếp theo.

## Key insight
> Android BluetoothGatt chỉ nhận một lệnh tại một thời điểm: gọi lệnh mới trước khi callback cũ trả về sẽ bị từ chối âm thầm.

## Ví von
Giống như buồng điện thoại công cộng chỉ có một ống nghe. Bạn phải chờ người trước nói chuyện và gác máy xong xuôi thì mới nhấc ống nghe bấm số gọi tiếp được.

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
Quên gọi `close()` sau khi ngắt kết nối sẽ làm hết sạch chỗ đăng ký client native, khiến mọi lần kết nối sau đều hỏng. Từ Android 7, nếu bạn bật tắt quét quá 5 lần trong 30 giây, hệ thống sẽ chặn và trả về danh sách rỗng. Từ Android 12, ứng dụng bắt buộc phải xin hai quyền `BLUETOOTH_SCAN` và `BLUETOOTH_CONNECT` lúc đang chạy. Triệu chứng khi dính bẫy là hàm ghi trả về `false`, hoặc callback trả về mã lỗi `status 133` liên tục.

## Câu hỏi thực chiến
**Hỏi:** Mã lỗi status 133 trong onConnectionStateChange là gì và xử lý thế nào?
Đó là mã lỗi chung `GATT_ERROR`, thường nổ ra khi thiết bị mất sóng, quá thời gian chờ hoặc hết chỗ đăng ký client native. Khi gặp mã này, bạn phải gọi `gatt.close()` ngay để dọn sạch kết nối cũ rồi chờ ít nhất 500ms mới kết nối lại.

**Hỏi:** Tại sao các hàm callback của BluetoothGatt lại không chạy trên Main thread?
Tiến trình Bluetooth hệ thống gửi kết quả qua cầu nối Binder IPC nên callback chạy ngay trên luồng Binder. Từ Android 8.0, nếu muốn callback chạy trên luồng mình chọn, bạn có thể truyền thêm một `Handler` vào hàm `connectGatt`.

**Hỏi:** Tham số autoConnect trong hàm connectGatt nên đặt true hay false?
Khi người dùng bấm kết nối và đứng gần thiết bị, hãy đặt `false` để Android quét nhanh trong vài trăm mili-giây. Chỉ đặt `true` khi muốn ứng dụng chờ dưới nền và tự nối lại mỗi khi thiết bị vào vùng phủ sóng.

**Hỏi:** Quên gọi close() sau khi disconnect() gây hậu quả nghiêm trọng gì?
Hàm `disconnect()` chỉ ngắt sóng chứ không trả lại chỗ đăng ký client trong tầng Bluetooth native của máy. Sau vài lần quên gọi `close()`, bảng đăng ký bị đầy khiến các lần gọi `connectGatt` tiếp theo trả về `null` hoặc báo lỗi `status 133`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao giao thức ATT chỉ nhận một lệnh mỗi lúc và cách biến cờ mDeviceBusy chặn lệnh thứ hai.
- [ ] Phân biệt được disconnect() chỉ ngắt sóng với close() trả lại slot client trong tầng Bluetooth native.
- [ ] Nêu được việc callback chạy trên luồng Binder IPC và cách truyền Handler riêng vào connectGatt từ Android 8.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
