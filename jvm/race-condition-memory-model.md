---
id: jvm-race-condition-memory-model
title: Race Condition & Memory Model
domain: jvm
tags: [jvm, concurrency, race-condition, memory-model, volatile, synchronized]
collections: [interview-fundamental]
related: [thread, jvm-stack-vs-heap, kotlin-stateflow-vs-sharedflow]
created: 2026-10-01
diagrams: [Cơ chế|race-condition-memory-barriers.png|So sánh rào chắn volatile bảo đảm nhìn thấy, CAS phi khóa nguyên tử và synchronized chiếm monitor lock]
---

## Bản 30 giây
Khi nhiều luồng cùng thao tác trên vùng nhớ dùng chung, kết quả có thể bị sai lệch do hiện tượng tranh chấp dữ liệu. Việc tối ưu lưu thanh ghi và đảo lệnh khiến một luồng không thấy thay đổi từ luồng khác. Mô hình bộ nhớ Java thiết lập các ranh giới đồng bộ bắt buộc để dữ liệu luôn nhất quán.

## Cần biết trước
- thread
- jvm-stack-vs-heap

## Vấn đề buộc nó ra đời
Năm 2000, Bill Pugh chỉ ra mô hình bộ nhớ Java có lỗi nền tảng qua bài viết 'The Java Memory Model is Fatally Flawed'. Nhóm của ông sau đó công bố bản tuyên bố vạch trần cơ chế Double-Checked Locking bị hỏng hoàn toàn.

Một luồng có thể nhìn thấy địa chỉ đối tượng khác null nhưng các trường dữ liệu bên trong vẫn chưa kịp khởi tạo xong. Việc giữ biến trong thanh ghi và bộ đệm lưu trữ (*store buffer*) khiến luồng khác đọc phải giá trị cũ.

Năm 2004, quy chuẩn JSR-133 trong Java 5 ra đời nhằm định nghĩa lại toàn diện mô hình bộ nhớ Java. Quy chuẩn này thiết lập quan hệ xảy ra trước (*happens-before*) và bổ sung rào chắn bộ nhớ nghiêm ngặt cho từ khóa `volatile`.

## Nó thực sự là cái gì
Race condition KHÔNG PHẢI là lỗi do luồng chạy chậm hay chạy nhanh hơn bình thường. Thực chất, đây là sự vi phạm bất biến khi các luồng truy cập thiếu quan hệ xảy ra trước (*happens-before*).

Máy ảo phân định rạch ròi giữa tính nhìn thấy (*visibility*) và tính nguyên tử (*atomicity*).

Từ khóa `volatile` chỉ bảo đảm tính nhìn thấy, còn tính nguyên tử đòi hỏi khóa độc quyền hoặc lệnh phần cứng.

## Cơ chế
Rào chắn bộ nhớ (*memory barrier*) cấm trình biên dịch và CPU đảo lệnh vượt qua ranh giới khai báo `volatile`. Nhờ đó, mọi thao tác ghi trước lệnh ghi `volatile` đều hiện ra đầy đủ với luồng đọc biến đó phía sau.

Với khối `synchronized`, luồng phải chiếm giữ khóa màn hình giám sát (*monitor lock*).

Cơ chế CAS dùng chỉ thị phần cứng nguyên tử để cập nhật giá trị mà không cần khóa luồng.

## Đánh đổi
Đồng bộ hóa chặn sai lệch khi mọi truy cập đều đi qua cùng một cơ chế đồng bộ. Đổi lại, việc chèn rào chắn bộ nhớ làm mất hiệu lực tối ưu đường ống dẫn lệnh của phần cứng.

Khóa luồng gây tốn chi phí đổi ngữ cảnh và có thể dẫn tới tắc nghẽn chết (*deadlock*).

## Nguyên lý gốc
Nguyên lý nhất quán: muốn các nhân thấy dữ liệu đồng nhất, hệ thống phải đánh đổi tốc độ bằng các điểm đồng bộ.

## Key insight
> `volatile` chỉ giúp luồng khác thấy giá trị mới; phép đọc-sửa-ghi vẫn cần khoá hoặc CAS.

## Ví von
Hai thư ký cùng nhìn một chiếc bảng thông báo ở sảnh chính nhưng lại ghi chép vào sổ tay riêng ở bàn làm việc. Nếu không ra sảnh cập nhật thường xuyên, một người sẽ áp dụng quyết định cũ dù người kia đã viết bảng mới.

## Khuôn mẫu chuẩn
```kotlin
class SafeCounter {
    private val count = java.util.concurrent.atomic.AtomicInteger(0)
    fun increment(): Int = count.incrementAndGet()
    fun get(): Int = count.get()
}
val counter = SafeCounter()
val value = counter.increment()
```

## Bẫy & ranh giới
Bẫy dùng `volatile` cho thao tác tăng đếm `count++` gây mất dữ liệu vì phép tăng gồm ba bước đọc, sửa và ghi riêng biệt. Chống chỉ định dùng `synchronized` bao quanh tác vụ I/O vì sẽ làm tắc nghẽn toàn bộ các luồng khác. Tránh đồng bộ hóa trên các đối tượng khóa công khai hoặc khóa có thể thay đổi địa chỉ vùng nhớ. Triệu chứng là sai lệch số liệu khi tải nặng, rò rỉ luồng hoặc treo app do khóa chéo nhau.

## Câu hỏi thực chiến
**Hỏi:** Từ khóa `volatile` có thay thế được `synchronized` khi hai luồng cùng tăng một biến đếm không?
Không. `volatile` chỉ đảm bảo luồng khác nhìn thấy giá trị mới nhất, chứ không đảm bảo tính nguyên tử. Phép tăng gồm ba bước đọc, cộng và ghi, nên hai luồng có thể đọc trùng giá trị và ghi đè mất của nhau.

**Hỏi:** Nguyên lý quan hệ xảy ra trước (*happens-before*) trong Java Memory Model nghĩa là gì?
Đây là cam kết đảm bảo thao tác ghi của một luồng chắc chắn được nhìn thấy bởi luồng đọc tiếp theo. Ví dụ lệnh nhả khóa màn hình giám sát luôn xảy ra trước lệnh chiếm khóa đó của luồng tiếp theo.

**Hỏi:** Vì sao các lớp `AtomicInteger` hay `AtomicReference` lại chạy nhanh hơn dùng khối `synchronized`?
Bởi vì chúng sử dụng cơ chế phi khóa (*lock-free*) dựa trên chỉ thị phần cứng so sánh và hoán đổi CAS. Luồng không bị chuyển sang trạng thái ngủ và không tốn chi phí chuyển ngữ cảnh của hệ điều hành khi có tranh chấp nhẹ.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt rạch ròi giữa tính nhìn thấy (visibility) và tính nguyên tử (atomicity).
- [ ] Giải thích nguyên nhân phần cứng gây race condition: thanh ghi, store buffer và tối ưu đảo thứ tự lệnh.
- [ ] Trình bày cơ chế CAS của AtomicInteger đối lập với monitor lock của synchronized.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

