---
id: android-gc-roots
title: GC Roots & Chuỗi Tham chiếu Android
domain: android
tags: [android, memory, garbage-collector, performance]
collections: []
related: [android-memory-leak, android-viewmodel, android-activity-lifecycle]
created: 2026-09-08
diagrams: [Cơ chế|gc-roots-reachability-graph.png|Đồ thị từ Thread stack, biến static và JNI; tham chiếu vòng cô lập bị thu hồi, chuỗi neo giữ gây rò rỉ Activity]
---

## Bản 30 giây
Bộ dọn rác cần biết đối tượng nào còn dùng và đối tượng nào đã bỏ đi để dọn sạch bộ nhớ. Hệ thống giải quyết việc này bằng cách xuất phát từ các điểm neo gốc luôn sống để lần theo chuỗi kết nối. Bất kỳ vùng nhớ nào mất kết nối tới các điểm neo gốc này đều sẽ bị dọn dẹp triệt để.

## Cần biết trước
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 1960, John McCarthy phát minh thuật toán thu gom rác Đánh dấu và Quét (Mark-and-Sweep) cho ngôn ngữ Lisp.

Thuật toán đếm tham chiếu bất lực trước tham chiếu vòng: hai đối tượng trỏ chéo nhau sẽ không bao giờ được giải phóng. Lỗi này làm các hệ thống nhúng nhanh chóng cạn kiệt RAM và sập tiến trình không thể cứu vãn.

Năm 2008, hệ điều hành Android đưa máy ảo Dalvik rồi sau đó là ART vào thực tế để quản lý bộ nhớ tự động. GC Roots giải quyết dứt điểm tham chiếu vòng tròn bằng cách chỉ coi những gì truy vết được từ gốc là còn sống.

## Nó thực sự là cái gì
Thu gom rác trong Android KHÔNG PHẢI là việc đi đếm số lượng mũi tên trỏ vào một đối tượng. Thực chất nó là thuật toán duyệt đồ thị định hướng bắt đầu từ một nhóm đối tượng đặc quyền gọi là GC Roots. GC Roots bao gồm: ngăn xếp của luồng đang chạy, các biến tĩnh (static), và tham chiếu toàn cục từ mã máy JNI. Nếu đối tượng không có đường dẫn tham chiếu mạnh nối về ít nhất một GC Root, nó được coi là rác.

## Cơ chế
Bộ dọn rác bắt đầu duyệt từ các GC Roots để truy vết toàn bộ đối tượng còn sống chạm tới được. Hai đối tượng trỏ chéo nhau nhưng cô lập khỏi toàn bộ GC Roots sẽ không thể tiếp cận và bị thu hồi. Trên ART hiện đại, máy ảo sao chép các đối tượng sống sang vùng nhớ mới rồi giải phóng nguyên vẹn vùng cũ. Một [rò rỉ bộ nhớ](concept:android-memory-leak) xảy ra khi đối tượng hết vòng đời nhưng vẫn nối vào một GC Root qua chuỗi tham chiếu mạnh.

## Đánh đổi
Lập trình viên thoát khỏi thảm họa quản lý bộ nhớ thủ công và lỗi con trỏ lơ lửng gây sập ứng dụng. Đổi lại, máy ảo tiêu tốn chu kỳ CPU đáng kể để liên tục sao chép và phân tích đồ thị đối tượng. Dù thời gian tạm dừng rất ngắn, dọn rác dồn dập vẫn tranh chấp CPU và làm nghẽn cấp phát gây giật khung hình.

## Nguyên lý gốc
Phân tích tính với tới (Reachability Analysis). Tính sống còn của thực thể không do chính nó định đoạt, mà dựa vào sự kết nối tới các điểm tựa nền tảng.

## Key insight
> Tham chiếu vòng không ngăn được dọn rác: đối tượng chỉ sống khi được neo vào luồng đang chạy, biến tĩnh hoặc JNI.

## Ví von
Giống như chiếc mỏ neo của một con tàu chìm dưới đáy biển sâu. Dù xác tàu đã hỏng nát vô dụng, sợi xích nối với mỏ neo cắm chặt vào đá ngầm vẫn giữ nó không trôi đi. Chỉ khi cắt đứt sợi xích, con tàu mới có thể được sóng biển cuốn trôi hoàn toàn.

## Khuôn mẫu chuẩn
```kotlin
// Minh họa Thread đóng vai trò GC Root vô tình giữ chặt Activity
class LeakActivity : Activity() {
    private var data = "Dữ liệu"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Thread {
            // Thread là GC Root neo giữ lambda truy cập data ngầm giữ this
            Thread.sleep(60_000)
            println(data)
        }.start()
    }
}
```

## Bẫy & ranh giới
Bẫy kinh điển là khởi tạo lớp nội danh hoặc lambda trong tác vụ nền chạy dài vô tận. Khi lambda truy cập thành viên lớp bao ngoài, trình biên dịch ngầm chèn tham chiếu `this`. Chống chỉ định dùng `Thread` thuần túy hoặc `GlobalScope` mà không hủy bỏ khi Activity bước vào trạng thái bị hủy hoại. Hãy luôn gắn kết tác vụ bất đồng bộ vào vòng đời thông qua `lifecycleScope` hoặc giải phóng tham chiếu trong phương thức dọn dẹp.

## Câu hỏi thực chiến
**Hỏi:** Hai đối tượng trỏ chéo lẫn nhau thì bộ dọn rác có chịu thua không?
Không. Bộ dọn rác duyệt xuôi từ GC Roots chứ không đi đếm số mũi tên trỏ vào đối tượng. Cả cụm trỏ vòng tròn vào nhau nhưng cô lập khỏi mọi GC Root vẫn bị giải phóng hoàn toàn.

**Hỏi:** Một Thread đang ngủ Thread.sleep() có được coi là GC Root không?
Có. Luồng còn sống là GC Root vì khung ngăn xếp của nó vẫn được máy ảo duy trì. Chừng nào luồng chưa chạy xong, mọi biến cục bộ và đối tượng bị giữ bởi lambda đều không thể thu hồi.

**Hỏi:** Gán biến bằng null có ép được Garbage Collector dọn dẹp ngay không?
Không. Gán `null` chỉ bẻ gãy một đường dẫn tham chiếu từ biến đó tới vùng nhớ Heap. Máy ảo chỉ thu hồi khi đối tượng mất toàn bộ đường nối tới GC Roots, vào thời điểm runtime tự chọn.

**Hỏi:** Đọc báo cáo rò rỉ của LeakCanary thì tìm GC Root ở đâu?
Nhìn thẳng vào nút đầu tiên ở đỉnh chuỗi tham chiếu. Dòng đầu tiên chính là GC Root đang neo giữ, thường là biến static hoặc một luồng đang chạy. Nhiệm vụ sửa lỗi là tìm mắt xích không hợp lý trong chuỗi đó để ngắt bỏ kết nối.

**Hỏi:** Biến static trong class có bị GC thu hồi bao giờ không?
Tuỳ vào việc ClassLoader nạp lớp đó còn sống hay đã bị hủy. Trong ứng dụng Android, ClassLoader mặc định sống suốt vòng đời tiến trình nên biến static gần như không bao giờ được dọn. Dữ liệu này chỉ thực sự biến mất khi toàn bộ tiến trình của ứng dụng bị hệ điều hành xóa sổ.

## Ý bắt buộc khi giải thích lại
- [ ] Liệt kê được ba nguồn GC Roots quan trọng nhất: Thread Stack, static fields và JNI globals.
- [ ] Phân biệt được cơ chế đồ thị tiếp cận (Tracing GC) với cơ chế đếm tham chiếu (Reference Counting).
- [ ] Giải thích được cách một luồng nền biến thành GC Root giữ chặt chuỗi tham chiếu tới Activity.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
