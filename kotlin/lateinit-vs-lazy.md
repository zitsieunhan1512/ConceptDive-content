---
id: kotlin-lateinit-vs-lazy
title: lateinit var vs by lazy
domain: kotlin
tags: [kotlin, initialization, bytecode, memory]
collections: [interview-fundamental]
related: [kotlin-val-vs-const, kotlin-null-safety, android-activity-lifecycle]
diagrams: [Cơ chế|lateinit-vs-lazy-mechanisms.png|lateinit kiểm tra null trực tiếp trên trường Java; by lazy ủy quyền qua đối tượng SynchronizedLazyImpl với khóa hai lớp]
created: 2026-09-15
---

## Bản 30 giây
`lateinit var` và `by lazy` giúp trì hoãn việc gán giá trị cho biến khi chưa thể khởi tạo ngay. `lateinit` dùng cho biến có thể thay đổi (`var`), phù hợp khi framework gán giá trị sau như trong vòng đời Android. `by lazy` dùng cho biến chỉ đọc (`val`), chỉ chạy đoạn mã khởi tạo ở lần đầu tiên bạn đọc giá trị.

## Cần biết trước
- kotlin-null-safety
- kotlin-val-vs-const

## Vấn đề buộc nó ra đời
Trong Java, mọi trường đối tượng đều có thể mang giá trị null ban đầu. Lập trình viên thường gán đối tượng sau trong `onCreate()` của Android hoặc qua Dagger.

Khi Kotlin ra đời năm 2011 với hệ thống [null safety](concept:kotlin-null-safety), mọi biến không-null bắt buộc phải có giá trị ngay khi tạo. Điều này gây bế tắc cho Android vì không thể khởi tạo View hay ViewModel trước khi Activity gắn vào hệ thống.

Nếu không có giải pháp, kỹ sư buộc phải để biến có dấu hỏi rồi lạm dụng toán tử ép kiểu `!!`. Nhóm thiết kế Kotlin đã tạo ra `lateinit` và `by lazy` để tháo gỡ vấn đề này.

## Nó thực sự là cái gì
`lateinit` KHÔNG phải cú pháp thần kỳ tự tìm cách gán biến. Thực chất, nó chỉ là một trường Java thông thường kèm lời hứa với compiler rằng biến sẽ có dữ liệu trước khi đọc.

Ngược lại, `by lazy` KHÔNG phải một biến thông thường. Nó là cơ chế ủy quyền thuộc tính (*property delegation*), dùng đối tượng bọc để tính giá trị ở lần đọc đầu tiên.

## Cơ chế
Ở tầng bytecode JVM, `lateinit` lưu dữ liệu trong một trường Java riêng tư thông thường. Compiler tự động chèn thêm lệnh kiểm tra `if (value == null)` trước mỗi lần đọc để ném lỗi nếu biến chưa gán.

Với `by lazy`, hàm ủy quyền tạo một đối tượng `SynchronizedLazyImpl` trên bộ nhớ Heap để giữ khối mã khởi tạo. Khi đọc lần đầu, nó dùng [khóa kiểm tra hai lớp](concept:jvm-race-condition-memory-model) (*double-checked locking*) để tính giá trị an toàn đa luồng rồi lưu lại.

## Đánh đổi
`lateinit` có tốc độ đọc ghi cực nhanh như trường Java gốc. Tuy nhiên, nó không hỗ trợ kiểu nguyên thủy (`Int`, `Boolean`) và không bảo đảm an toàn khi đọc ghi từ nhiều luồng.

`by lazy` mang lại tính bất biến (`val`) và an toàn đa luồng. Đổi lại, bạn tốn thêm bộ nhớ Heap cho đối tượng bọc và chịu chi phí khóa luồng khi khởi tạo.

## Nguyên lý gốc
Nguyên lý khởi tạo lười biếng (*Lazy Initialization*): trì hoãn cấp phát tài nguyên và bộ nhớ cho đến khi thực sự cần dùng. Cơ chế này giúp rút ngắn thời gian khởi động của chương trình.

## Key insight
> by lazy không phải là biến thông thường, mà là đối tượng ủy quyền tính toán một lần và đồng bộ luồng trên Heap.

## Khuôn mẫu chuẩn
```kotlin
class UserProfileActivity : AppCompatActivity() {
    lateinit var presenter: UserPresenter

    private val userAdapter by lazy(LazyThreadSafetyMode.NONE) {
        UserAdapter(onItemClick = { presenter.onSelect(it) })
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        presenter = UserPresenter()
    }
}
```

## Bẫy & ranh giới
`by lazy` mặc định tốn một object bao bọc và chi phí khóa luồng; `lateinit` tốn thêm lệnh so sánh null ở mỗi lần đọc. Không dùng `lateinit` trong môi trường đa luồng vì không có cơ chế khóa bảo vệ. Không dùng `by lazy` cho các đối tượng phụ thuộc vào dữ liệu chỉ có sau khi Activity khởi chạy. Đọc `lateinit` quá sớm sẽ gây lỗi sập `UninitializedPropertyAccessException`, còn `by lazy` phụ thuộc vòng tròn sẽ gây lỗi `StackOverflowError`.

## Câu hỏi thực chiến
**Hỏi:** Tại sao lateinit không áp dụng được cho kiểu nguyên thủy như Int hay Boolean?
Không thể dùng vì kiểu nguyên thủy trong Java không thể mang giá trị null. Trình biên dịch dựa vào giá trị null nội bộ để kiểm tra biến đã khởi tạo hay chưa. Nếu hỗ trợ, ngôn ngữ phải đổi sang kiểu bao bọc (boxed object) gây tốn bộ nhớ.

**Hỏi:** Ba chế độ của LazyThreadSafetyMode khác nhau như thế nào?
Khác nhau ở cơ chế khóa luồng. SYNCHRONIZED dùng khóa an toàn tuyệt đối; PUBLICATION cho nhiều luồng cùng chạy nhưng chỉ lấy kết quả đầu tiên. NONE không có khóa, chạy nhanh nhất và phù hợp nhất cho luồng giao diện Android.

**Hỏi:** Làm sao để kiểm tra một biến lateinit đã được khởi tạo hay chưa mà không bị sập app?
Dùng cú pháp `this::presenter.isInitialized`. Biến phản chiếu này kiểm tra trực tiếp trường nội bộ mà không kích hoạt ngoại lệ. Cách này chỉ áp dụng được cho thuộc tính có thể truy cập trực tiếp trong cùng một lớp.

## Ý bắt buộc khi giải thích lại
- [ ] lateinit là lời hứa với compiler cho var, by lazy là ủy quyền thuộc tính cho val.
- [ ] Cơ chế bytecode: lateinit kiểm tra null nội bộ, by lazy dùng đối tượng bao bọc có khóa hai lớp.
- [ ] Đánh đổi: lateinit không hỗ trợ kiểu nguyên thủy, by lazy tốn bộ nhớ Heap và chi phí khóa luồng.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

