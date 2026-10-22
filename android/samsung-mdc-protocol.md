---
id: android-samsung-mdc-protocol
title: Samsung MDC Protocol
domain: android
tags: [protocol, iot, hardware]
collections: []
related: [android-tcp-stream-framing, android-ble-android-pitfalls, kotlin-coroutine-context-and-dispatchers]
created: 2026-09-21
diagrams: [Cơ chế|samsung-mdc-packet-frame.png|Cấu trúc gói gửi và phản hồi qua RS-232C/TCP; mã phát sóng 0xFE không ACK và tổng kiểm tra khử dấu bằng and 0xFF]
---

## Bản 30 giây
Giao thức điều khiển màn hình thương mại qua cáp nối tiếp hoặc mạng.
Mỗi lệnh là chuỗi byte bắt đầu bằng `0xAA` và kết thúc bằng tổng kiểm tra.
Ứng dụng phải gửi từng lệnh tuần tự và chờ phản hồi trước khi gửi tiếp.

## Cần biết trước
- android-tcp-stream-framing
- kotlin-coroutine-context-and-dispatchers

## Vấn đề buộc nó ra đời
Khi lắp hàng loạt màn hình thương mại, kỹ thuật viên không thể dùng điều khiển hồng ngoại thao tác từng chiếc. Kéo riêng từng sợi cáp nối tiếp từ máy chủ đến hàng chục màn hình cũng làm cạn kiệt cổng vật lý.

Samsung cho phép nối chuỗi cáp nối tiếp, từ cổng ra của máy trước vào cổng vào máy sau. Giao thức Multiple Display Control (MDC) ra đời để gửi lệnh nhị phân có định danh tới từng màn hình trên cùng một đường truyền.

Người vận hành có thể phát sóng lệnh đồng loạt mà không sợ nghẽn mạch do các máy phản hồi cùng lúc.

## Nó thực sự là cái gì
MDC KHÔNG PHẢI là giao thức dựa trên văn bản như HTTP. Thực chất, nó là định dạng khung nhị phân điều khiển phần cứng trên cổng RS-232C hoặc cổng TCP 1515 của [dòng byte TCP](concept:android-tcp-stream-framing).

## Cơ chế
Gói gửi gồm byte đầu `0xAA`, mã lệnh, mã màn hình, độ dài dữ liệu, dữ liệu và một byte tổng kiểm tra (checksum).
Lệnh đọc và lệnh đặt dùng chung mã lệnh, phân biệt bằng độ dài dữ liệu bằng không hay mang giá trị cài đặt.
Màn hình phản hồi bằng `0xAA`, `0xFF`, mã màn hình, độ dài, ký tự `A` hoặc `N`, mã lệnh gốc và byte tổng kiểm tra.
Mã phát sóng `0xFE` gửi tới toàn màn hình mà không nhận phản hồi, còn dòng mới hỗ trợ bảo mật TLS.

## Đánh đổi
Tổng kiểm tra 8 bit tiết kiệm phần cứng nhưng bỏ sót lỗi hoán vị hai byte dữ liệu so với CRC.
Kênh truyền chỉ xử lý một lệnh tại một thời điểm, khiến ứng dụng phải tự xây [hàng đợi tuần tự](concept:android-ble-android-pitfalls).
Lập trình viên phải giới hạn bằng [bộ điều phối](concept:kotlin-coroutine-context-and-dispatchers) hoặc khóa đồng bộ, làm tăng độ phức tạp quản lý luồng.

## Nguyên lý gốc
Giao thức giao dịch bán song công (half-duplex transaction): một bên hỏi và một bên đáp theo chu kỳ đóng gói tuần tự. Mỗi yêu cầu phải hoàn tất hoặc hết thời gian chờ trước khi phiên giao tiếp tiếp nhận thông điệp mới.

## Key insight
> Byte Java có dấu khiến `0xAA` âm, nhưng cộng dồn rồi `and 0xFF` vẫn đúng vì phần dấu chỉ thêm bội số 256.

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
Trong Java, so sánh `byte == 0xAA` luôn sai do mở rộng dấu âm; còn Kotlin chặn lỗi này ngay khi biên dịch.
Để so sánh đúng trong Kotlin, lập trình viên bắt buộc phải chuyển đổi bằng `(byte.toInt() and 0xFF) == 0xAA`.
Bắn dồn dập nhiều lệnh sẽ xáo trộn gói phản hồi, khiến ứng dụng nhận nhầm kết quả giữa các tác vụ.
Giao thức trả giá bằng độ trễ hàng đợi; trường Length một byte buộc dữ liệu lớn phải chia nhiều gói 255 byte.

## Câu hỏi thực chiến
**Hỏi:** Vì sao cộng các byte âm trong mảng rồi `and 0xFF` vẫn ra đúng tổng kiểm tra?
Do bit dấu mở rộng chỉ cộng thêm các bội số của 256 vào số nguyên 32 bit. Khi lấy 8 bit thấp qua phép `and 0xFF`, toàn bộ các bội số 256 này đều bị triệt tiêu hoàn toàn. Kết quả cuối cùng luôn đồng dư với tổng các số nguyên không dấu tương ứng.

**Hỏi:** Có thể bắn đồng thời nhiều lệnh MDC trên cùng một kết nối TCP không?
Không, vì giao thức này vận hành theo cơ chế yêu cầu và phản hồi tuần tự trên một dòng truyền dữ liệu. Bắn lệnh dồn dập sẽ làm byte phản hồi bị lẫn lộn giữa các lệnh khác nhau. Bạn bắt buộc phải tuần tự hóa qua hàng đợi hoặc khóa đồng bộ.

**Hỏi:** Khi dùng mã phát sóng `0xFE`, ứng dụng có nhận được phản hồi ACK từ màn hình không?
Không, các màn hình trong chuỗi nối tiếp được quy định không phản hồi lệnh phát sóng. Quy ước này nhằm ngăn chặn thảm họa xung đột tín hiệu khi hàng chục màn hình cùng phát dữ liệu ngược về máy chủ. Ứng dụng phải chấp nhận cơ chế bắn lệnh mà không có xác nhận trạng thái tức thì.

## Ý bắt buộc khi giải thích lại
- [ ] Trình bày được cấu trúc gói tin MDC gồm byte đầu 0xAA, mã lệnh, ID, độ dài và checksum.
- [ ] Giải thích được nguyên nhân sign-extension trong Java và cách dùng `and 0xFF` để khử phần dấu.
- [ ] Phân tích được lý do phải tuần tự hóa lệnh và vì sao mã phát sóng 0xFE không có ACK.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
