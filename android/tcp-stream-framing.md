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
Giao thức mạng TCP chỉ truyền tải một dòng byte liên tục và không giữ lại ranh giới giữa các lần gửi.
Một thao tác đọc có thể nhận thiếu hoặc thừa byte của một gói tin logic.
Ứng dụng phải tự định nghĩa cấu trúc khung gói và đọc lặp cho đến khi đủ dữ liệu.

## Cần biết trước
- jvm-stack-vs-heap

## Vấn đề buộc nó ra đời
Năm 1981, chuẩn RFC 793 chính thức định nghĩa TCP là dịch vụ dòng byte tin cậy, không có ranh giới thông điệp. Thiết kế này giúp mạng linh hoạt ghép nhiều mảnh nhỏ thành một gói lớn hoặc chia nhỏ dữ liệu theo kích thước kênh truyền.

Nhiều lập trình viên quen với UDP thường lầm tưởng một lệnh gửi sẽ tương ứng đúng một lệnh nhận. Khi mạng chập chờn hoặc gói tin lớn, một nửa gói đến trước và một nửa đến sau.

Ứng dụng xử lý dữ liệu dở dang sẽ sập hoặc đọc lẫn byte thừa của gói trước, làm hỏng phiên truyền tải.

## Nó thực sự là cái gì
TCP KHÔNG PHẢI là giao thức vận chuyển các gói tin rời rạc. Thực chất, nó là một dòng byte liên tục (stream of octets), không bảo toàn ranh giới giữa các lần gửi dữ liệu.

## Cơ chế
Hàm đọc socket chỉ lấy số byte sẵn có trong bộ đệm và có thể trả về ít byte hơn yêu cầu.
Ứng dụng phải lặp cho đến khi đủ dữ liệu hoặc dùng phương thức chuyên dụng như `DataInputStream.readFully` để gom đủ khung gói.
Tầng ứng dụng phân tách gói bằng cách dùng tiền tố độ dài hoặc ký tự phân cách như trong [Samsung MDC Protocol](concept:android-samsung-mdc-protocol).
Chương trình đọc xong phần đầu cố định để xác định độ dài thân gói rồi mới đọc tiếp đúng số byte phần thân đó.

## Đánh đổi
Kỹ thuật đóng khung thủ công bảo toàn tính toàn vẹn của dữ liệu logic trên kênh truyền dòng chảy.
Đổi lại, bên nhận phải tốn bộ đệm trên [Heap](concept:jvm-stack-vs-heap) và tốn CPU để dò tìm dấu phân cách hoặc độ dài.
Nếu không thiết lập giới hạn tối đa cho trường độ dài, dữ liệu mạng sai lệch sẽ gây cạn kiệt bộ nhớ ứng dụng.

## Nguyên lý gốc
Tầng vận chuyển chỉ truyền dòng byte tin cậy, còn ranh giới thông điệp do tầng ứng dụng hoàn toàn chịu trách nhiệm. Mọi dòng dữ liệu phi cấu trúc đều đòi hỏi một giao thức đóng khung để khôi phục lại ranh giới thông điệp ban đầu.

## Key insight
> TCP bảo đảm thứ tự byte chứ KHÔNG giữ ranh giới gói, nên một lần đọc không tương đương một thông điệp hoàn chỉnh.

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
Đọc thiếu byte xảy ra ngay trong vận hành bình thường khiến ứng dụng xử lý dữ liệu dở dang và tính sai cấu trúc.
Byte thừa chưa xử lý làm lệch pha vĩnh viễn mọi thông điệp đến sau, và cấp phát vượt ngưỡng gây tràn bộ nhớ Heap.
Kỹ thuật này trả giá bằng bộ đệm Heap, và không cần dùng khi tầng dưới đã bảo toàn ranh giới như UDP.

## Câu hỏi thực chiến
**Hỏi:** Vì sao code socket chỉ gọi `read` một lần vẫn chạy tốt trên máy dev nhưng lại sập ở môi trường thật?
Do mạng nội bộ có độ trễ thấp và gói tin nhỏ, dữ liệu thường nằm trọn trong một phân đoạn TCP. Trên mạng thật với độ trễ cao và nhiễu sóng, gói tin dễ bị xé nhỏ thành nhiều đợt truyền dẫn khác nhau. Code gọi `read` một lần sẽ nhận thiếu dữ liệu và sập ngay lập tức.

**Hỏi:** Dấu hiệu điển hình của việc phân tách khung gói tin bị lệch pha là gì?
Là hiện tượng parser liên tục báo lỗi dữ liệu rác hoặc không tìm thấy byte mở đầu của gói tin. Khi một gói bị đọc thiếu hoặc thừa byte, con trỏ đọc sẽ nằm lơ lửng ở giữa thân dữ liệu của gói sau. Mọi nỗ lực phân tích các thông điệp tiếp theo đều thất bại do lệch ranh giới.

**Hỏi:** Vì sao bắt buộc phải kiểm tra giới hạn kích thước trước khi cấp phát bộ đệm từ trường độ dài?
Có, nhằm ngăn chặn nguy cơ cạn kiệt bộ nhớ dẫn đến sập ứng dụng. Kẻ tấn công hoặc đường truyền nhiễu có thể gửi trường độ dài mang giá trị giả mạo lên tới hàng gigabyte. Cấp phát mảng byte trực tiếp mà không kiểm tra sẽ gây cạn kiệt Heap ngay tức khắc.

## Ý bắt buộc khi giải thích lại
- [ ] Trình bày được bản chất dòng byte phi cấu trúc của TCP và lý do không bảo toàn ranh giới thông điệp.
- [ ] Giải thích được nguyên nhân gây hiện tượng đọc thiếu byte hoặc lệch pha khung gói tin trên mạng thật.
- [ ] Phân tích được rủi ro cạn kiệt bộ nhớ Heap khi cấp phát mảng trực tiếp từ trường độ dài nhận được.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
