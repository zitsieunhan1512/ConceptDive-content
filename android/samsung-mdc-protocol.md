---
id: android-samsung-mdc-protocol
title: Samsung MDC Protocol
domain: android
tags: [protocol, iot, hardware]
collections: []
related: [android-tcp-stream-framing, android-ble-android-pitfalls, kotlin-coroutine-context-and-dispatchers]
created: 2026-09-21
sources:
  - Tài liệu giao thức Samsung MDC | https://github.com/vgavro/samsung-mdc
  - Chuẩn truyền thông nối tiếp RS-232 | https://en.wikipedia.org/wiki/RS-232
  - Tổng kiểm tra Checksum — Wikipedia | https://en.wikipedia.org/wiki/Checksum
diagrams: [Cơ chế|samsung-mdc-packet-frame.png|Cấu trúc gói gửi và phản hồi qua RS-232C/TCP; mã phát sóng 0xFE không ACK và tổng kiểm tra khử dấu bằng and 0xFF]
---

## Bản 30 giây
Khi viết app điều khiển màn hình thương mại Samsung qua cáp nối tiếp hoặc mạng, bạn dùng giao thức `MDC`. Mỗi lệnh gửi đi là một mảng byte mở đầu bằng `0xAA` và chốt cuối bằng một byte `checksum`. Ứng dụng phải gửi từng lệnh một, đợi màn hình trả lời xong rồi mới được gửi lệnh kế tiếp.

## Cần biết trước
- android-tcp-stream-framing
- kotlin-coroutine-context-and-dispatchers

## Vấn đề buộc nó ra đời
Khi lắp hàng chục màn hình quảng cáo ở sân bay, kỹ thuật viên không thể cầm điều khiển hồng ngoại đi bấm từng chiếc. Kéo riêng từng sợi cáp từ máy chủ tới mỗi màn hình cũng làm hết sạch cổng cắm vật lý.

Samsung cho phép cắm cáp nối tiếp từ cổng ra của màn hình trước sang cổng vào của màn hình sau. Giao thức Multiple Display Control (`MDC`) gửi lệnh nhị phân gắn mã ID tới đúng màn hình trên một dây.

Nhờ quy ước không trả lời lệnh phát chung, bạn bật tắt cả dàn máy mà không nghẽn đường truyền.

## Nó thực sự là cái gì
`MDC` không gửi chữ đọc được bằng mắt như HTTP. Nó đóng gói lệnh thành khung byte ngắn, truyền qua cáp `RS-232C` hoặc cổng `TCP 1515` trên [TCP Stream Framing](concept:android-tcp-stream-framing).

## Cơ chế
Gói gửi đi gồm byte đầu `0xAA`, mã lệnh, ID màn hình, độ dài, dữ liệu và byte `checksum`. Lệnh hỏi trạng thái và đổi cài đặt dùng chung mã lệnh, chỉ khác độ dài dữ liệu bằng 0 hay có giá trị. Khung trả lời gồm `0xAA`, `0xFF`, ID, độ dài, chữ `A` (thành công) hoặc `N` (lỗi), mã lệnh và `checksum`. Với mã phát sóng `0xFE`, mọi màn hình làm theo nhưng không trả lời; đời máy mới có thêm `TLS`.

## Đánh đổi
Cộng dồn 8 bit làm `checksum` rất nhẹ, nhưng đổi chỗ hai byte thì tổng không đổi nên thua `CRC`. Đường truyền cũng chỉ xử lý được một lệnh tại một lúc, nên app của bạn phải tự làm [hàng đợi tuần tự](concept:android-ble-android-pitfalls). Bạn buộc phải khóa luồng gửi bằng `Mutex` hoặc [Coroutine Dispatcher](concept:kotlin-coroutine-context-and-dispatchers) đơn luồng để các lệnh không giẫm lên nhau.

## Nguyên lý gốc
Trên kênh bán song công (half-duplex), bên điều khiển phải hỏi từng câu và đợi trả lời xong (hoặc hết giờ chờ) mới gửi tiếp.

## Key insight
> Dù `Byte` có dấu biến `0xAA` thành số âm, cộng dồn rồi `and 0xFF` vẫn ra đúng `checksum` do phần dấu là bội của 256.

## Ví von
MDC giống như máy bộ đàm một chiều giữa đài chỉ huy và các tàu biển. Chỉ huy đọc lệnh cho một tàu rồi đợi tàu đó xác nhận xong mới đọc tiếp lệnh cho tàu khác.

## Khuôn mẫu chuẩn
```kotlin
fun buildMdcPacket(cmd: Byte, id: Byte, data: ByteArray = byteArrayOf()): ByteArray {
    var sum = cmd.toInt() + id.toInt() + data.size
    for (b in data) sum += b.toInt()
    val checksum = (sum and 0xFF).toByte()
    return byteArrayOf(0xAA.toByte(), cmd, id, data.size.toByte(), *data, checksum)
}
```

## Bẫy & ranh giới
Trong Java, `byte == 0xAA` luôn ra `false` vì `byte` có dấu đổi thành số âm khi sang `Int` (sign-extension); Kotlin chặn lỗi này lúc biên dịch. Muốn so sánh đúng trong Kotlin, bạn phải khử phần dấu âm bằng cách viết `(byte.toInt() and 0xFF) == 0xAA`. Nếu gửi nhiều lệnh không chờ nhau, các byte trả về sẽ trộn lẫn làm lệnh trước đọc nhầm kết quả. Giao thức này chịu trễ do xếp hàng lệnh, và dữ liệu trên 255 byte phải cắt nhỏ vì ô độ dài chỉ 1 byte.

## Câu hỏi thực chiến
**Hỏi:** Vì sao cộng các byte âm trong mảng rồi `and 0xFF` vẫn ra đúng tổng kiểm tra?
Khi `Byte` âm sang `Int` 32 bit, các byte cao lấp đầy bit 1, tức cộng thêm bội số của 256. Lúc bạn lấy 8 bit cuối bằng phép `and 0xFF`, toàn bộ phần bội số của 256 đó bị cắt bỏ sạch. Nhờ vậy, 8 bit còn lại giống hệt như khi bạn cộng các số không dấu từ đầu.

**Hỏi:** Có thể bắn đồng thời nhiều lệnh MDC trên cùng một kết nối TCP không?
Không. Màn hình trả lời lần lượt trên cùng một dòng byte, mà gói trả về không có mã riêng cho từng lần gọi. Nếu bắn hai lệnh cùng lúc, byte phản hồi nối đuôi nhau làm không rõ gói nào của lệnh nào, nên phải xếp hàng.

**Hỏi:** Khi dùng mã phát sóng `0xFE`, ứng dụng có nhận được phản hồi ACK từ màn hình không?
Không. Khi nhận lệnh có ID là `0xFE`, mọi màn hình đều làm theo lệnh nhưng được quy định là không gửi gói trả lời về. Quy ước này giúp hàng chục màn hình nối chung cáp không phát tín hiệu ngược cùng lúc gây loạn đường truyền.

## Ý bắt buộc khi giải thích lại
- [ ] Trình bày được cấu trúc gói tin MDC gồm byte đầu 0xAA, mã lệnh, ID, độ dài, dữ liệu và checksum.
- [ ] Giải thích được vì sao byte 0xAA bị đổi thành số âm khi sang Int và cách dùng `and 0xFF` để lấy lại đúng 8 bit cuối.
- [ ] Nêu được lý do phải xếp hàng gửi từng lệnh một và vì sao mã phát sóng 0xFE không trả lời ACK.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
