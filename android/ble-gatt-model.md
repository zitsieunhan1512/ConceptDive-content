---
id: android-ble-gatt-model
title: BLE GATT Model
domain: android
tags: [android, ble, bluetooth, iot, networking]
collections: [interview-fundamental]
related: [android-ble-android-pitfalls, kotlin-coroutines]
created: 2026-09-21
diagrams: [Cơ chế|ble-gatt-hierarchy.png|Cấu trúc phân tầng Profile, Service, Characteristic và ghi cờ kích hoạt vào Descriptor CCCD 0x2902 để nhận thông báo]
---

## Bản 30 giây
Khi bạn kết nối điện thoại với vòng tay đo nhịp tim, BLE không mở sẵn một đường truyền chảy liên tục. Thiết bị ngoại vi lưu dữ liệu thành cây mục nhỏ, và điện thoại chỉ đọc, ghi hoặc chờ báo tin khi số đo đổi. Cách gửi từng mẩu tin ngắn lúc có thay đổi giúp viên pin cúc áo chạy được rất lâu.

## Cần biết trước

## Vấn đề buộc nó ra đời
Bluetooth cổ điển (BR/EDR) được làm cho các luồng dữ liệu chảy liên tục như tai nghe nhạc. Nó bắt hai bên giữ sóng liên tục nên làm cạn viên pin cúc áo chỉ sau vài ngày.

Năm 2006, Nokia giới thiệu công nghệ Wibree dành cho những thiết bị cần tiêu thụ thật ít điện. Năm 2010, tổ chức Bluetooth SIG đưa Wibree vào chuẩn Bluetooth 4.0 dưới tên gọi BLE.

BLE cần cách trao đổi dữ liệu gọn nhẹ, cất từng mẩu thông tin vào các mục riêng lẻ. Giao thức GATT ra đời để sắp xếp các mục dữ liệu đó trên nền giao thức thuộc tính ATT.

## Nó thực sự là cái gì
GATT không phải là một đường ống chảy liên tục như Bluetooth cổ điển. Nó giống bảng dữ liệu nhỏ nằm ngay trên thiết bị ngoại vi để điện thoại hỏi hoặc chờ báo tin khi có số mới.

## Cơ chế
Tầng GAP chia việc kết nối: `Central` đi quét và nối vào, còn `Peripheral` phát tín hiệu quảng bá. Sang tầng GATT, `Server` giữ bảng dữ liệu còn `Client` gửi lệnh đọc hoặc ghi. Bảng này xếp thành 3 tầng mang mã `UUID`: `Profile` gom các `Service`, `Service` chứa `Characteristic`, và mỗi `Characteristic` gắn thêm `Descriptor`. Muốn thiết bị tự báo tin mới, `Client` phải ghi cờ bật thông báo vào `Descriptor` CCCD mã `0x2902` trên `Server`.

## Đánh đổi
Nhờ chỉ gửi những gói tin ngắn lúc cần, BLE giúp cảm biến chạy hàng năm bằng một viên pin cúc áo. Đổi lại, tốc độ truyền rất thấp và gói tin `ATT MTU` mặc định 23 byte chỉ chở được 20 byte dữ liệu. Muốn gửi gói dài hơn, bạn phải dùng Long Write hoặc gọi `requestMtu` để xin gói lớn hơn (Android 14 tự xin mức 517 byte).

## Nguyên lý gốc
Thay vì giữ dòng chảy liên tục, thiết bị tách trạng thái thành từng ô nhỏ, ô nào đổi mới gửi.

## Key insight
> Bật thông báo BLE không chỉ gọi hàm trên Android: bạn bắt buộc phải ghi cờ kích hoạt vào descriptor CCCD của thiết bị.

## Ví von
GATT giống bảng hộp thư ở sảnh chung cư: mỗi tầng là một `Service`, mỗi căn hộ là một `Characteristic`. Bạn phải gạt công tắc đăng ký ở cửa hộp thư (ghi CCCD) thì chủ nhà mới báo cho bạn mỗi khi có thư mới.

## Khuôn mẫu chuẩn
```kotlin
// Bat Notify tren Characteristic va ghi Descriptor CCCD 0x2902
gatt.setCharacteristicNotification(characteristic, true)
val cccdUuid = UUID.fromString("00002902-0000-1000-8000-00805f9b34fb")
val descriptor = characteristic.getDescriptor(cccdUuid)
descriptor?.let {
    it.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
    gatt.writeDescriptor(it)
}
```

## Bẫy & ranh giới
Nếu bạn chỉ gọi `setCharacteristicNotification` mà quên ghi CCCD `0x2902`, bạn sẽ dính [bẫy BLE](concept:android-ble-android-pitfalls) làm app không nhận được gói tin nào. Kiểu Write Without Response vẫn được tầng liên kết xác nhận gói, nhưng bỏ phản hồi ATT nên ghi dồn nhanh sẽ tràn bộ đệm. Cũng đừng tự truyền âm thanh hay video qua đặc tính GATT, vì âm thanh trên BLE phải đi qua kênh riêng của LE Audio. Khi gửi quá cỡ mà chưa xin tăng `MTU`, gói tin sẽ bị cắt còn 20 byte hoặc lệnh ghi trả lỗi bận.

## Câu hỏi thực chiến
**Hỏi:** Vai trò GAP khác gì vai trò GATT trong một kết nối BLE?
GAP lo chuyện tìm nhau và bắt tay: `Central` đi quét và kết nối, còn `Peripheral` phát quảng bá. GATT lo chuyện đọc ghi dữ liệu sau khi đã nối xong: `Server` giữ bảng dữ liệu, còn `Client` gửi lệnh hỏi. Hai cặp vai trò này tách biệt nhau.

**Hỏi:** Tại sao gửi mảng 50 byte qua Characteristic lại bị cắt cụt còn 20 byte?
Vì gói tin `ATT MTU` mặc định của một kết nối BLE chỉ dài 23 byte. Mất 3 byte đầu cho phần tiêu đề ATT, nên chỗ chứa dữ liệu của bạn chỉ còn đúng 20 byte.

**Hỏi:** Làm thế nào để gửi gói tin BLE lớn hơn 20 byte mà không phải cắt mảng thủ công?
Bạn có thể dùng cơ chế Long Write hoặc gọi `gatt.requestMtu(517)` để xin nới rộng gói tin. Từ Android 14, hệ điều hành tự xin mức 517 byte ngay ở yêu cầu `MTU` đầu tiên.

**Hỏi:** Notify khác gì Indicate khi thiết bị ngoại vi truyền dữ liệu về điện thoại?
Với `Notify`, thiết bị cứ gửi dữ liệu mà không chờ điện thoại xác nhận ở tầng ATT nên gửi được nhanh hơn. Với `Indicate`, điện thoại phải trả lời đã nhận xong gói trước thì thiết bị mới gửi tiếp gói sau, chậm hơn nhưng chắc chắn.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được vai trò kết nối của GAP (Central/Peripheral) với vai trò dữ liệu của GATT (Client/Server).
- [ ] Giải thích được vì sao bật Notify phải vừa gọi hàm Android SDK vừa ghi cờ vào Descriptor CCCD 0x2902.
- [ ] Nêu được giới hạn gói ATT MTU mặc định 23 byte (chở 20 byte dữ liệu) và cách tăng bằng requestMtu.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
