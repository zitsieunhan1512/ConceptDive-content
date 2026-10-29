---
id: android-tcp-stream-framing
title: TCP Stream Framing
domain: android
tags: [network, protocol, socket]
collections: [interview-fundamental]
related: [android-samsung-mdc-protocol, jvm-stack-vs-heap]
created: 2026-09-21
diagrams: [Cơ chế|tcp-stream-framing-accumulator.png|Dòng byte TCP không giữ ranh giới, ứng dụng đọc 2 byte độ dài rồi dùng readFully gom đủ thân gói]
---

## Bản 30 giây
Khi gửi hai gói tin qua socket `TCP`, mạng chỉ truyền một dòng byte nối liền và không giữ ranh giới. Một lần gọi hàm đọc ở bên nhận có thể lấy thiếu byte hoặc dính luôn cả mấy byte của gói kế tiếp. Do đó, ứng dụng phải tự quy ước khung gói và đọc lặp tới khi gom đủ số byte của tin nhắn.

## Cần biết trước
- jvm-stack-vs-heap

## Vấn đề buộc nó ra đời
Năm 1981, chuẩn RFC 793 định nghĩa `TCP` là dịch vụ truyền dòng byte tin cậy nhưng không giữ ranh giới giữa các lần gửi. Thiết kế này giúp tầng mạng tự do gom nhiều mẫu tin nhỏ hoặc cắt nhỏ dữ liệu theo đường truyền.

Nhiều bạn quen làm việc với `UDP` hay tưởng rằng bên kia gửi một lần thì bên này gọi `read` một lần là nhận đủ. Khi ra mạng thật bị trễ hoặc gửi gói lớn, nửa đầu gói tin tới trước còn nửa sau tới trễ hơn một nhịp.

Nếu ứng dụng đem mảng byte dở dang đi giải mã, chương trình sẽ báo lỗi hoặc đọc lẹm sang gói sau.

## Nó thực sự là cái gì
`TCP` không phải là người đưa thư giao từng phong bì rời rạc cho bạn. Nó chỉ là dòng chảy liên tục các byte (stream of octets) và không đánh dấu chỗ hết một lần gửi.

## Cơ chế
Mỗi lần gọi `read` trên socket, hệ thống chỉ lấy byte có sẵn trong bộ đệm nên dễ trả về thiếu. Bạn phải tự viết vòng lặp đọc gom dần hoặc gọi thẳng `DataInputStream.readFully` để bắt nó đợi tới khi gom đủ số byte. Để biết độ dài gói tin, ứng dụng gắn vài byte độ dài ở đầu hoặc dùng byte đánh dấu như [Samsung MDC Protocol](concept:android-samsung-mdc-protocol). Chương trình đọc phần đầu cố định để biết độ dài thân gói, rồi gọi `readFully` lấy tiếp đủ phần thân.

## Đánh đổi
Tự đóng khung gói tin giúp bạn ghép lại đúng từng thông điệp hoàn chỉnh dù mạng bên dưới cắt ghép thế nào. Đổi lại, bên nhận phải cấp phát mảng đệm trên [Heap](concept:jvm-stack-vs-heap) và tốn thêm nhịp CPU để đọc độ dài hoặc dò byte đánh dấu. Nếu quên chặn giới hạn cho ô độ dài, một gói tin lỗi báo độ dài khổng lồ sẽ làm tràn bộ nhớ.

## Nguyên lý gốc
Tầng `TCP` chỉ cam kết giao byte đúng thứ tự, còn việc cắt dòng byte thành bản tin thuộc về tầng ứng dụng.

## Key insight
> `TCP` giữ đúng thứ tự từng byte nhưng không giữ ranh giới gói, nên một lần gọi `read` không bằng một lần gọi `write`.

## Ví von
TCP giống như băng chuyền chuyển than liên tục vào lò sấy. Người nhận phải dùng thùng chứa có kích thước định sẵn để xúc than theo từng mẻ chứ than không tự rơi thành từng bao.

## Khuôn mẫu chuẩn
```kotlin
fun readLengthPrefixedFrame(input: DataInputStream, maxPayloadSize: Int = 4096): ByteArray {
    val length = input.readUnsignedShort()
    require(length <= maxPayloadSize) { "Payload $length bytes vượt quá giới hạn" }
    val payload = ByteArray(length)
    input.readFully(payload)
    return payload
}
```

## Bẫy & ranh giới
Hàm `read` trả thiếu byte xảy ra thường xuyên trên mạng thật, khiến code đọc một lần lấy dữ liệu cụt và lỗi. Ngược lại, nếu không giữ phần byte dư cho lần đọc kế, mọi gói tin phía sau sẽ bị đọc lệch vị trí. Cách đóng khung này tốn bộ nhớ đệm trên Heap, và không cần dùng với giao thức giữ sẵn ranh giới như `UDP`.

## Câu hỏi thực chiến
**Hỏi:** Vì sao code socket chỉ gọi `read` một lần vẫn chạy tốt trên máy dev nhưng lại sập ở môi trường thật?
Trên mạng nội bộ, đường truyền nhanh và gói tin nhỏ nên cả thông điệp thường gọn trong một gói `TCP`. Ra mạng di động chập chờn hoặc gửi dữ liệu lớn, tầng mạng cắt thông điệp thành nhiều mảnh gửi lệch nhịp. Lúc đó hàm `read` gọi một lần chỉ lấy được mảnh đầu tiên, làm code giải mã bị sập ngay.

**Hỏi:** Dấu hiệu điển hình của việc phân tách khung gói tin bị lệch pha là gì?
Bộ giải mã sẽ liên tục báo lỗi dữ liệu rác, sai `checksum` hoặc mất byte mở đầu ở các gói sau. Chỉ cần gói trước đọc thiếu hoặc lố vài byte, vị trí đọc kế tiếp sẽ rơi vào giữa thân gói sau. Từ giây phút đó, toàn bộ dòng byte trên kết nối bị lệch nhịp cho tới khi đóng socket mở lại.

**Hỏi:** Vì sao bắt buộc phải kiểm tra giới hạn kích thước trước khi cấp phát bộ đệm từ trường độ dài?
Để tránh việc ứng dụng bị sập vì tràn bộ nhớ `OutOfMemoryError` chỉ bởi một gói tin hỏng hoặc kẻ gian cố tình gửi bậy. Khi lệch pha hoặc nhận hai byte độ dài cực lớn, lệnh `ByteArray(length)` sẽ đòi cấp phát ngay ngần ấy bộ nhớ. Kiểm tra ngưỡng tối đa bằng `require` giúp bạn ngắt kết nối sớm trước khi bộ nhớ Heap bị vắt kiệt.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được bản chất dòng byte liên tục của TCP và vì sao nó không giữ ranh giới giữa các lần gửi.
- [ ] Trình bày được vì sao gọi read một lần dễ lấy thiếu byte hoặc làm lệch vị trí đọc của toàn bộ các gói tin phía sau.
- [ ] Nêu được nguy cơ tràn bộ nhớ Heap nếu tạo mảng ByteArray thẳng từ số đo độ dài mà không kiểm tra giới hạn tối đa.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
