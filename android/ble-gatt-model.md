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
Bluetooth năng lượng thấp trao đổi dữ liệu qua cấu trúc cây thuộc tính lưu trên thiết bị ngoại vi. Điện thoại kết nối vào để đọc, ghi hoặc đăng ký nhận thông báo biến động từ cảm biến. Mô hình này giúp thiết bị tiết kiệm pin tối đa bằng cách chỉ truyền dữ liệu nhỏ khi có thay đổi.

## Cần biết trước

## Vấn đề buộc nó ra đời
Bluetooth cổ điển (BR/EDR) được thiết kế cho luồng truyền tải dữ liệu liên tục như tai nghe âm thanh. Nó đòi hỏi duy trì kết nối sóng vô tuyến liên tục, làm cạn sạch pin cúc áo chỉ sau vài ngày hoạt động.

Năm 2006, Nokia giới thiệu công nghệ Wibree hướng tới các thiết bị tiêu thụ năng lượng siêu thấp. Năm 2010, tổ chức Bluetooth SIG chính thức tích hợp Wibree vào chuẩn Bluetooth 4.0 dưới tên gọi BLE.

BLE cần một mô hình trao đổi dữ liệu phi tập trung, đóng gói dữ liệu thành các trường thuộc tính rời rạc. Giao thức GATT ra đời để chuẩn hóa cấu trúc dữ liệu trên nền giao thức thuộc tính ATT.

## Nó thực sự là cái gì
GATT KHÔNG PHẢI là đường ống truyền dữ liệu luồng liên tục như Bluetooth cổ điển. Thực chất, nó là cơ sở dữ liệu thuộc tính trên ngoại vi để điện thoại truy vấn hoặc nhận thông báo biến động.

## Cơ chế
Vai trò GAP phân định Central (quét và kết nối) và Peripheral (quảng bá). Trong GATT, Server lưu trữ bảng thuộc tính còn Client gửi yêu cầu đọc ghi. Dữ liệu phân tầng: Profile gom Service, Service chứa Characteristic, và Characteristic đi kèm Descriptor có mã định danh UUID. Muốn nhận dữ liệu chủ động, GATT Client phải ghi cờ kích hoạt vào Descriptor CCCD mang mã `0x2902` trên Server.

## Đánh đổi
BLE giúp cảm biến chạy hàng năm chỉ với pin cúc áo nhờ gói tin nhỏ ngắt quãng. Đổi lại, băng thông rất thấp và gói ATT MTU mặc định 23 byte chỉ chở được 20 byte dữ liệu. Truyền dữ liệu lớn hơn đòi hỏi cơ chế Long Write hoặc thương lượng `requestMtu` (tự đặt 517 từ Android 14).

## Nguyên lý gốc
Nguyên lý máy khách và máy chủ thuộc tính: chia nhỏ trạng thái thành các điểm nguyên tử thay vì duy trì luồng truyền.

## Key insight
> Bật thông báo BLE không chỉ gọi hàm trên Android: bạn bắt buộc phải ghi cờ kích hoạt vào descriptor CCCD của thiết bị.

## Ví von
GATT như bảng thông báo ở sảnh chung cư: mỗi tầng là một Service, mỗi căn hộ là một Characteristic. Bạn phải bấm chuông đăng ký nhận tin (ghi CCCD) thì chủ nhà mới chủ động báo tin mỗi khi có thư mới.

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
Chỉ gọi `setCharacteristicNotification` mà quên ghi CCCD `0x2902` là [bẫy BLE](concept:android-ble-android-pitfalls) khiến app không bao giờ nhận được dữ liệu từ ngoại vi. Write Without Response tuy nhanh nhưng dễ rơi rớt gói tin trên tầng ATT do không có phản hồi xác nhận. Chống chỉ định dùng BLE cho truyền tải âm thanh hoặc video vì giới hạn MTU và chu kỳ sóng sẽ bóp nghẹt băng thông. Triệu chứng nghẽn là dữ liệu bị cắt vụn thành 20 byte khiến tầng ứng dụng giải mã sai lệch và gây trễ nghiêm trọng.

## Câu hỏi thực chiến
**Hỏi:** Vai trò GAP khác gì vai trò GATT trong một kết nối BLE?
GAP quản lý việc quảng bá và kết nối (Central quét và kết nối, Peripheral quảng bá). GATT quản lý việc trao đổi dữ liệu sau kết nối (Server giữ cơ sở dữ liệu thuộc tính, Client gửi lệnh truy vấn). Hai cặp vai trò này hoàn toàn độc lập với nhau.

**Hỏi:** Tại sao gửi mảng 50 byte qua Characteristic lại bị cắt cụt còn 20 byte?
Vì kích thước gói ATT MTU mặc định của kết nối BLE chỉ có 23 byte. Ba byte đầu tiên dành cho tiêu đề ATT, nên phần dữ liệu ứng dụng thực tế chỉ còn vỏn vẹn 20 byte.

**Hỏi:** Làm thế nào để gửi gói tin BLE lớn hơn 20 byte mà không phải cắt mảng thủ công?
Bạn có thể dùng Long Write hoặc gọi `gatt.requestMtu(517)` để tăng kích thước gói tin. Từ Android 14, hệ điều hành tự động thương lượng mức 517 byte ngay khi có yêu cầu MTU đầu tiên.

**Hỏi:** Notify khác gì Indicate khi thiết bị ngoại vi truyền dữ liệu về điện thoại?
Notify bắn dữ liệu mà không cần điện thoại phản hồi xác nhận ở tầng ATT, giúp đạt tốc độ truyền cao hơn. Indicate bắt buộc điện thoại gửi gói tin xác nhận trước khi gửi gói tiếp theo, đảm bảo tin cậy nhưng tốc độ chậm hơn.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt rạch ròi GAP (Central/Peripheral) với GATT (Client/Server).
- [ ] Cơ chế kích hoạt Notify bắt buộc ghi cờ vào CCCD 0x2902 ngoài việc gọi hàm Android SDK.
- [ ] Giới hạn ATT MTU mặc định 23 byte (20 byte dữ liệu), cơ chế Long Write và requestMtu.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
