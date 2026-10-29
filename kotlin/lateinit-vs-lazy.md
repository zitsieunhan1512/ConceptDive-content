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
Khi viết `Activity` trên Android, nhiều biến chỉ có dữ liệu sau khi `onCreate()` chạy, nên không thể gán ngay lúc khai báo. `lateinit var` dành cho biến gán sau và đổi lại được, còn `by lazy` dành cho biến `val` chỉ tính ở lần đọc đầu. Nhờ hai cách này, bạn không phải khai báo biến có dấu hỏi rồi viết `?.` hay `!!` khắp nơi.

## Cần biết trước
- kotlin-null-safety
- kotlin-val-vs-const

## Vấn đề buộc nó ra đời
Trong Java, mọi trường của lớp đều mặc định bằng null trước rồi mới gán đối tượng thật trong `onCreate()` hoặc qua thư viện Dagger. Cách làm này rất quen thuộc trên Android suốt nhiều năm.

Khi Kotlin ra đời năm 2011 với [null safety](concept:kotlin-null-safety), mọi biến không-null đều phải gán giá trị ngay lúc tạo đối tượng. Lập trình viên Android bị kẹt, vì lúc tạo `Activity` thì chưa lấy được `View` hay `ViewModel`.

Nếu không có lối mở, bạn phải khai báo biến kèm dấu hỏi rồi thêm `!!` ở mọi chỗ dùng. Nhóm thiết kế Kotlin đưa vào `lateinit` và `by lazy` để bạn dời việc gán giá trị sang thời điểm sau.

## Nó thực sự là cái gì
`lateinit` không có phép tự đi tìm dữ liệu để gán cho biến. Nó chỉ là một trường Java bình thường kèm lời hứa của bạn với compiler rằng biến sẽ được gán trước khi đọc. Ngược lại, `by lazy` không phải là một biến đơn lẻ mà là ủy quyền thuộc tính (*property delegation*). Nó cất khối lệnh khởi tạo vào một đối tượng bọc riêng, đợi lần đầu bạn đọc biến mới lấy ra chạy.

## Cơ chế
Ở bytecode JVM, `lateinit` tạo trường Java cùng mức hiển thị với setter (`lateinit var` là `public` thì trường cũng `public`). Compiler tự chèn câu kiểm tra `if (value == null)` vào hàm đọc để ném lỗi nếu bạn đọc trước khi gán. Còn `by lazy` tạo một đối tượng `SynchronizedLazyImpl` trên Heap để giữ khối lệnh tính giá trị. Ở lần đọc đầu, nó dùng [khóa kiểm tra hai lớp](concept:jvm-race-condition-memory-model) (*double-checked locking*) để tính một lần rồi cất kết quả cho các lần sau.

## Đánh đổi
`lateinit` đọc ghi trực tiếp vào trường Java nên chạy nhanh và không tốn thêm đối tượng bọc. Đổi lại, nó không dùng được cho kiểu nguyên thủy như `Int` hay `Boolean`, và không tự khóa khi nhiều luồng cùng gán. Còn `by lazy` giữ biến chỉ đọc (`val`) và an toàn đa luồng, nhưng tốn một đối tượng trên Heap cùng khóa ở lần đọc đầu.

## Nguyên lý gốc
Chưa cần dùng tới thì chưa tạo ra, đợi đúng lúc gọi tới lần đầu mới cấp bộ nhớ và tính toán (*Lazy Initialization*).

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
Mặc định `by lazy` tốn thêm đối tượng bọc trên Heap và khóa đồng bộ, còn `lateinit` tốn một phép kiểm tra null mỗi lần đọc. Đừng dùng `lateinit` khi nhiều luồng cùng gán mà không khóa, và đừng đọc `by lazy` sớm nếu nó cần dữ liệu sau `onCreate()`. Triệu chứng khi gọi `lateinit` lúc chưa gán là app sập ngay với lỗi `UninitializedPropertyAccessException`. Còn nếu hai biến `by lazy` gọi chéo nhau trong khối khởi tạo, app sẽ sập vì lỗi tràn ngăn xếp `StackOverflowError`.

## Câu hỏi thực chiến
**Hỏi:** Tại sao lateinit không áp dụng được cho kiểu nguyên thủy như Int hay Boolean?
Vì ở tầng JVM, compiler dùng giá trị `null` để đánh dấu là biến `lateinit` chưa được gán. Các kiểu nguyên thủy như `int` hay `boolean` trong Java luôn có số mặc định là `0` hoặc `false` chứ không thể mang `null`. Nếu muốn đánh dấu `null`, Kotlin phải bọc chúng thành đối tượng `Integer` trên Heap làm tốn thêm bộ nhớ.

**Hỏi:** Ba chế độ của LazyThreadSafetyMode khác nhau như thế nào?
Chúng khác nhau ở cách xử lý khi nhiều luồng cùng đọc biến ở lần đầu tiên. `SYNCHRONIZED` dùng khóa để chỉ một luồng được tính; `PUBLICATION` cho nhiều luồng cùng tính nhưng chỉ giữ lại kết quả nào xong trước. Còn `NONE` bỏ hẳn khóa nên chạy nhanh nhất, rất hợp cho biến chỉ dùng trên luồng chính Android.

**Hỏi:** Làm sao để kiểm tra một biến lateinit đã được khởi tạo hay chưa mà không bị sập app?
Bạn gọi `this::presenter.isInitialized` để kiểm tra xem trường bên dưới đã khác null hay chưa. Cách viết này đọc thẳng vào trường nội bộ nên không làm nổ lỗi chưa khởi tạo. Lưu ý là bạn chỉ gọi được nó ở bên trong cùng lớp hoặc lớp bọc ngoài gần nhất.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được lateinit (lời hứa gán sau cho biến var) với by lazy (đối tượng ủy quyền tính một lần cho biến val).
- [ ] Giải thích được bytecode: trường của lateinit có cùng mức hiển thị với setter và kiểm tra null khi đọc, còn by lazy bọc trong SynchronizedLazyImpl.
- [ ] Nêu được vì sao lateinit không dùng cho Int/Boolean và khi nào nên truyền LazyThreadSafetyMode.NONE cho by lazy trên Android.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
