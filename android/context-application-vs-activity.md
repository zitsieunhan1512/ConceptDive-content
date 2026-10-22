---
id: android-context-application-vs-activity
title: Context: Application vs Activity
domain: android
tags: [android, context, memory-leak, lifecycle]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-memory-leak, android-viewmodel]
created: 2026-09-15
diagrams: [Cơ chế|context-activity-vs-application.png|Activity có Theme và Window Token mở Dialog hợp lệ; Application thiếu token ném BadTokenException nhưng an toàn cho Singleton]
---

## Bản 30 giây
`Context` là cây cầu kết nối ứng dụng với tài nguyên và dịch vụ của hệ điều hành Android. `Activity` là Context gắn liền giao diện và vòng đời màn hình, dùng để hiển thị hộp thoại và nạp giao diện có chủ đề. `Application` là Context sống cùng tiến trình, phù hợp cho cơ sở dữ liệu hoặc kho lưu trữ để chống rò rỉ bộ nhớ.

## Cần biết trước
- android-activity-lifecycle
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2008, Android được thiết kế trên mô hình đa tiến trình với nhiều thành phần độc lập như Activity, Service và BroadcastReceiver. Các thành phần này cần truy cập tài nguyên hệ thống như layout, database và thông báo.

Google tạo ra lớp trừu tượng `Context` làm cổng giao tiếp chung. Tuy nhiên, sự khác biệt về vòng đời giữa các thành phần đã tạo ra lỗ hổng lớn về quản lý bộ nhớ.

Nếu lập trình viên đưa nhầm Context màn hình vào đối tượng dùng chung, [Memory leak](concept:android-memory-leak) sẽ xuất hiện và làm sập ứng dụng. Ngược lại, dùng Context toàn cục cho giao diện lại gây mất chủ đề màu sắc hoặc sập app khi mở cửa sổ.

## Nó thực sự là cái gì
`Context` KHÔNG phải là một lớp nguyên khối chứa mọi quyền hạn. Thực chất, đây là mẫu trang trí (*Decorator pattern*) bọc các dịch vụ hệ thống với các phạm vi vòng đời khác nhau.

`Activity` kế thừa qua `ContextThemeWrapper`, nắm giữ thông tin giao diện và mã định danh cửa sổ (*window token*). `Application` kế thừa trực tiếp từ `ContextWrapper`, không có giao diện và tồn tại xuyên suốt vòng đời của tiến trình ứng dụng.

## Cơ chế
Khi hiển thị `Dialog`, hệ thống yêu cầu một mã định danh cửa sổ hợp lệ gắn với Activity trên màn hình. `Application` không gắn vào bất kỳ cửa sổ nào nên ném lỗi `BadTokenException` nếu bạn cố mở hộp thoại.

Khi nạp giao diện, `Activity` áp dụng trực tiếp cấu hình chủ đề (*Theme*) được khai báo trong file cấu hình manifest. Nếu nạp layout bằng `applicationContext`, giao diện sẽ rơi về chủ đề mặc định của hệ thống và làm vỡ màu sắc thiết kế.

## Đánh đổi
Dùng `applicationContext` giúp ứng dụng miễn nhiễm với lỗi rò rỉ bộ nhớ trong các đối tượng sống lâu. Đổi lại, nó không thể xử lý giao diện và bắt buộc thêm cờ hiệu đặc biệt khi mở màn hình mới.

Dùng `Activity` mang lại trải nghiệm giao diện hoàn chỉnh nhưng đòi hỏi bạn phải giải phóng tham chiếu nghiêm ngặt khi đóng màn hình.

## Nguyên lý gốc
Nguyên lý vòng đời: đối tượng sống lâu không bao giờ được giữ tham chiếu tới đối tượng có vòng đời ngắn hơn. Vi phạm nguyên lý này là nguyên nhân gốc rễ gây ra rò rỉ bộ nhớ trên mọi nền tảng phần mềm.

## Key insight
> applicationContext không thể thay thế cho Activity, vì nó hoàn toàn thiếu thông tin chủ đề và mã định danh cửa sổ giao diện.

## Khuôn mẫu chuẩn
```kotlin
class PreferenceStore(private val appContext: Context) {
    private val prefs = appContext.getSharedPreferences("prefs", Context.MODE_PRIVATE)
}

class UserProfileActivity : AppCompatActivity() {
    fun showConfirmDialog() {
        AlertDialog.Builder(this)
            .setTitle("Xác nhận")
            .show()
    }
}
```

## Bẫy & ranh giới
Truyền `Activity` vào lớp singleton hay tác vụ nền sẽ neo giữ toàn bộ màn hình trên Heap, gây lỗi sập `OutOfMemoryError`. Dùng `applicationContext` để tạo `AlertDialog` sẽ làm ứng dụng sập ngay lập tức với lỗi `WindowManager.BadTokenException`. Khởi động Activity từ `applicationContext` buộc phải thêm cờ `FLAG_ACTIVITY_NEW_TASK`, làm đảo lộn luồng điều hướng của người dùng. Hãy luôn dùng `applicationContext` cho các đối tượng tồn tại lâu dài và chỉ dùng `Activity` cho các hiển thị trực quan.

## Câu hỏi thực chiến
**Hỏi:** Điều gì sẽ xảy ra nếu hiển thị AlertDialog bằng applicationContext?
Ứng dụng sẽ bị sập ngay lập tức. Hệ thống sẽ ném ra ngoại lệ `WindowManager.BadTokenException`. Lỗi này xảy ra vì `applicationContext` không sở hữu mã định danh cửa sổ của màn hình đang hiển thị.

**Hỏi:** Khi nào bắt buộc phải dùng Activity Context thay vì Application Context?
Bắt buộc dùng khi thực hiện các tác vụ liên quan đến hiển thị giao diện. Ví dụ điển hình gồm tạo Dialog, nạp View có chủ đề hoặc lấy dịch vụ `WindowManager`. Ngoài ra, mở màn hình mới bằng Activity Context sẽ giữ nguyên ngăn xếp điều hướng tự nhiên.

**Hỏi:** Sự khác biệt giữa getApplication() và context.applicationContext là gì?
Khác nhau ở kiểu trả về và vị trí gọi. `getApplication()` chỉ có trên Activity hoặc Service và trả về trực tiếp kiểu `Application`. Trong khi đó, `applicationContext` có trên mọi đối tượng Context và trả về kiểu trừu tượng `Context`.

**Hỏi:** Truyền Activity Context vào thư viện Room Database sẽ gây ra hậu quả gì?
Gây rò rỉ bộ nhớ nghiêm trọng trên bộ nhớ Heap. Thư viện cơ sở dữ liệu thường tồn tại dưới dạng singleton suốt vòng đời ứng dụng. Tham chiếu này khiến trình dọn rác không thể thu hồi màn hình sau khi người dùng đã đóng.

## Ý bắt buộc khi giải thích lại
- [ ] Phân cấp kế thừa: Activity có Theme và Window Token, Application chỉ là bao bọc dịch vụ chung.
- [ ] Bẫy bộ nhớ: truyền Activity vào singleton gây rò rỉ toàn bộ cây View trên Heap.
- [ ] Bẫy giao diện: dùng Application Context tạo Dialog sẽ sập app với BadTokenException.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

