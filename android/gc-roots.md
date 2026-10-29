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
Khi ứng dụng chạy, bộ thu gom rác cần biết đối tượng nào bạn còn dùng và đối tượng nào đã bỏ đi. Nó bắt đầu từ các điểm gốc luôn sống gọi là `GC Roots`, rồi lần theo từng biến tham chiếu để đánh dấu. Bất kỳ đối tượng nào không nối về được một `GC Root` nào đều sẽ bị dọn khỏi bộ nhớ.

## Cần biết trước
- android-memory-leak

## Vấn đề buộc nó ra đời
McCarthy làm bộ thu gom rác Mark-and-Sweep cho Lisp năm 1959, công bố năm 1960.

Trước đó, cách đếm số tham chiếu chịu thua khi hai đối tượng trỏ chéo vào nhau, vì số đếm không bao giờ về không. Lỗi tham chiếu vòng này làm các thiết bị ít RAM nhanh chóng hết sạch bộ nhớ và sập tiến trình.

Năm 2008, Android đưa máy ảo Dalvik rồi sau đó là ART vào quản lý bộ nhớ tự động. Cách duyệt từ `GC Roots` giải quyết gọn tham chiếu vòng bằng cách chỉ giữ lại những đối tượng nối được về điểm gốc.

## Nó thực sự là cái gì
Bộ thu gom rác trên Android không đi đếm xem mỗi đối tượng có bao nhiêu mũi tên trỏ vào. Thay vào đó, nó đi duyệt các mũi tên xuất phát từ một nhóm điểm gốc đặc biệt gọi là `GC Roots`. Ba nhóm `GC Roots` chính là biến trên ngăn xếp của luồng đang chạy, biến `static`, và tham chiếu toàn cục từ JNI. Nếu một đối tượng không còn đường tham chiếu mạnh nào nối từ ít nhất một `GC Root`, nó được coi là rác.

## Cơ chế
Khi dọn dẹp, bộ thu gom rác đi từ các `GC Roots` và lần theo tham chiếu để đánh dấu đối tượng sống. Nhờ vậy, hai đối tượng dù trỏ chéo nhau nhưng đã đứt đường nối với `GC Roots` vẫn bị thu hồi như thường. Trên máy ảo ART hiện nay, hệ thống chép các đối tượng còn sống sang vùng nhớ mới rồi xóa trắng vùng cũ. Ngược lại, [rò rỉ bộ nhớ](concept:android-memory-leak) xảy ra khi màn hình đã đóng nhưng vẫn dính vào một `GC Root` qua chuỗi tham chiếu mạnh.

## Đánh đổi
Bạn không phải tự viết lệnh giải phóng bộ nhớ và tránh được lỗi truy cập vào vùng nhớ đã xóa. Đổi lại, máy ảo phải tốn CPU để duyệt đồ thị đối tượng và chép dữ liệu sang vùng nhớ mới. Nếu bạn tạo và vứt đối tượng liên tục, việc dọn rác dồn dập sẽ chiếm CPU và làm màn hình bị giật.

## Nguyên lý gốc
Đối tượng sống hay chết không phụ thuộc vào chính nó. Nó phụ thuộc vào việc từ `GC Roots` có còn đường đi tới nó hay không (Reachability Analysis).

## Key insight
> Tham chiếu vòng không ngăn được dọn rác: đối tượng chỉ sống khi được neo vào luồng đang chạy, biến tĩnh hoặc JNI.

## Ví von
Hãy tưởng tượng một con tàu đã hỏng nằm dưới biển nhưng vẫn nối xích vào chiếc mỏ neo cắm chặt dưới đáy đá. Chỉ khi bạn cắt đứt sợi xích nối với mỏ neo đó, sóng biển mới cuốn con tàu đi được.

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
Bẫy hay gặp nhất là mở một luồng hoặc khối lambda chạy ngầm thật lâu bên trong `Activity`. Khi khối lambda đó đọc một biến của `Activity`, trình biên dịch sẽ ngầm giữ luôn tham chiếu `this` tới cả màn hình. Vì vậy, đừng tạo `Thread` tự do hay dùng `GlobalScope` mà không hủy khi `Activity` đóng lại. Hãy chạy việc ngầm trong `lifecycleScope` để nó tự hủy và cắt đứt đường nối về `GC Root` ngay khi người dùng thoát màn hình.

## Câu hỏi thực chiến
**Hỏi:** Hai đối tượng trỏ chéo lẫn nhau thì bộ dọn rác có chịu thua không?
Không. Bộ dọn rác trên Android đi từ các `GC Roots` ra ngoài, chứ không đếm số mũi tên trỏ vào từng đối tượng. Khi cả cụm hai đối tượng đó không còn đường nào nối về `GC Roots`, bộ dọn rác sẽ thu hồi cả hai.

**Hỏi:** Một Thread đang ngủ Thread.sleep() có được coi là GC Root không?
Có. Chừng nào luồng chưa chạy xong hàm `run()`, ngăn xếp của nó vẫn sống và được tính là một `GC Root`. Vì vậy, mọi đối tượng mà khối lệnh trong luồng đó đang nắm giữ đều không thể được dọn đi.

**Hỏi:** Gán biến bằng null có ép được Garbage Collector dọn dẹp ngay không?
Không. Gán `null` chỉ cắt đứt một sợi dây nối từ biến đó tới đối tượng trên vùng nhớ Heap. Máy ảo chỉ thu hồi đối tượng khi nó mất sạch mọi đường nối về `GC Roots` và đến đúng đợt dọn rác của hệ thống.

**Hỏi:** Đọc báo cáo rò rỉ của LeakCanary thì tìm GC Root ở đâu?
Bạn nhìn ngay dòng đầu tiên trên cùng của chuỗi tham chiếu trong báo cáo. Dòng đó chính là `GC Root` đang giữ chặt màn hình của bạn, thường là một biến `static` hoặc một luồng đang chạy. Việc của bạn là đọc từ trên xuống để tìm mắt xích giữ tham chiếu thừa và cắt nó đi.

**Hỏi:** Biến static trong class có bị GC thu hồi bao giờ không?
Biến `static` chỉ được thu hồi khi chính `ClassLoader` đã nạp lớp đó bị gỡ khỏi bộ nhớ. Trên Android, `ClassLoader` chính sống suốt vòng đời tiến trình nên biến `static` hầu như không bao giờ mất. Mọi thứ gán vào biến `static` sẽ nằm trong RAM tới khi bạn gán `null` hoặc tắt ứng dụng.

## Ý bắt buộc khi giải thích lại
- [ ] Kể được ba nguồn GC Roots quan trọng nhất: ngăn xếp luồng đang chạy (Thread Stack), biến static và tham chiếu JNI toàn cục.
- [ ] Phân biệt được cách duyệt từ gốc (Tracing GC) với cách đếm tham chiếu (Reference Counting) khi gặp tham chiếu vòng.
- [ ] Giải thích được cách một luồng chạy ngầm trở thành GC Root giữ chặt chuỗi tham chiếu tới Activity đã đóng.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
