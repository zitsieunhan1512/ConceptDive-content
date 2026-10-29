---
id: android-context-application-vs-activity
title: Context: Application vs Activity
domain: android
tags: [android, context, memory-leak, lifecycle]
collections: [interview-fundamental]
related: [android-activity-lifecycle, android-memory-leak, android-viewmodel]
created: 2026-09-15
sources:
  - Context API reference — Android Developers | https://developer.android.com/reference/android/content/Context
  - ContextThemeWrapper API reference — Android Developers | https://developer.android.com/reference/android/view/ContextThemeWrapper
  - Application API reference — Android Developers | https://developer.android.com/reference/android/app/Application
diagrams: [Cơ chế|context-activity-vs-application.png|Activity có Theme và Window Token mở Dialog hợp lệ; Application thiếu token ném BadTokenException nhưng an toàn cho Singleton]
---

## Bản 30 giây
`Context` là đầu mối để ứng dụng của bạn lấy tài nguyên và gọi dịch vụ của hệ điều hành Android. `Activity` là `Context` gắn với màn hình đang mở, dùng để bật hộp thoại và nạp giao diện theo đúng bảng màu. Còn `Application` là `Context` sống suốt lúc ứng dụng chạy, hợp để truyền vào cơ sở dữ liệu nhằm tránh rò rỉ bộ nhớ.

## Cần biết trước
- android-activity-lifecycle
- android-memory-leak

## Vấn đề buộc nó ra đời
Năm 2008, Android được thiết kế với nhiều thành phần độc lập như Activity, Service và BroadcastReceiver. Các thành phần này đều cần một cách chung để đọc file layout, mở database hay gửi thông báo.

Google tạo ra lớp trừu tượng `Context` làm đầu mối giao tiếp chung. Nhưng vì mỗi thành phần có tuổi thọ dài ngắn khác nhau, việc dùng chung một kiểu `Context` rất dễ gây lỗi bộ nhớ.

Nếu bạn truyền nhầm `Context` của màn hình vào đối tượng sống lâu, [Memory leak](concept:android-memory-leak) sẽ xảy ra và làm ứng dụng hết RAM. Ngược lại, lấy `Context` toàn cục đi mở hộp thoại thì ứng dụng sẽ sập ngay hoặc mất sạch màu sắc giao diện.

## Nó thực sự là cái gì
`Context` không phải là một đối tượng vạn năng ở đâu cũng giống nhau. Nó dùng mẫu bọc ngoài (*Decorator pattern*) để gắn thêm khả năng tùy theo tuổi thọ từng thành phần.

`Activity` kế thừa từ `ContextThemeWrapper`, nên nó có thêm bảng màu giao diện và thẻ căn cước cửa sổ (*window token*). Trong khi đó, `Application` kế thừa thẳng từ `ContextWrapper`, không gắn với cửa sổ nào và sống từ lúc mở tới lúc tắt tiến trình.

## Cơ chế
Khi bạn mở một `Dialog`, hệ thống đòi phải có thẻ `window token` hợp lệ để biết đặt hộp thoại đè lên màn hình nào. Vì `Application` không gắn với cửa sổ nào cả, truyền nó vào `Dialog` sẽ làm hệ thống ném lỗi `BadTokenException` ngay.

Còn khi nạp file XML giao diện, `Activity` tự áp bảng màu (*Theme*) bạn đã khai báo trong manifest lên các nút bấm. Nếu bạn nạp giao diện bằng `applicationContext`, các View sẽ lấy màu mặc định của hệ thống và làm lệch thiết kế của bạn.

## Đánh đổi
Truyền `applicationContext` vào các đối tượng sống lâu giúp bạn yên tâm không bao giờ làm rò rỉ màn hình. Đổi lại, nó không mở được hộp thoại và nếu dùng để mở màn hình mới thì phải gắn thêm cờ tạo ngăn xếp riêng.

Dùng `Activity` thì vẽ giao diện và mở hộp thoại đầy đủ. Tuy nhiên, bạn phải gỡ mọi tham chiếu tới nó ngay khi màn hình đóng.

## Nguyên lý gốc
Đối tượng sống lâu không bao giờ được giữ tham chiếu tới đối tượng có tuổi thọ ngắn hơn mình.

## Key insight
> applicationContext không thể thay thế cho Activity, vì nó hoàn toàn thiếu thông tin bảng màu và thẻ window token của cửa sổ giao diện.

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
Truyền `Activity` vào biến singleton hay việc chạy ngầm sẽ giữ chặt cả màn hình trên Heap, dẫn tới lỗi sập `OutOfMemoryError`. Ngược lại, đưa `applicationContext` vào `AlertDialog.Builder` sẽ làm ứng dụng sập ngay lập tức với lỗi `WindowManager.BadTokenException`. Nếu gọi `startActivity` từ `applicationContext`, bạn buộc phải thêm cờ `FLAG_ACTIVITY_NEW_TASK`, dễ làm lệch thứ tự các màn hình đang mở. Vì vậy, hãy truyền `applicationContext` cho kho dữ liệu sống lâu và chỉ dùng `Activity` cho các việc hiển thị trên màn hình.

## Câu hỏi thực chiến
**Hỏi:** Điều gì sẽ xảy ra nếu hiển thị AlertDialog bằng applicationContext?
Ứng dụng sẽ sập ngay khi dòng lệnh mở hộp thoại chạy. Hệ thống ném ra lỗi `WindowManager.BadTokenException`. Lý do là `applicationContext` không có thẻ `window token` để báo cho hệ thống biết hộp thoại này thuộc về màn hình nào.

**Hỏi:** Khi nào bắt buộc phải dùng Activity Context thay vì Application Context?
Bạn bắt buộc phải dùng `Activity` cho mọi việc đụng tới giao diện trên màn hình. Ví dụ như mở `Dialog`, nạp file XML cần giữ đúng màu sắc `Theme`, hoặc lấy dịch vụ `WindowManager`. Ngoài ra, mở màn hình mới từ `Activity` sẽ xếp màn hình đó nối tiếp tự nhiên vào ngăn xếp hiện tại.

**Hỏi:** Sự khác biệt giữa getApplication() và context.applicationContext là gì?
Hai cách này trả về cùng một đối tượng duy nhất trong bộ nhớ, chỉ khác nhau ở kiểu dữ liệu và nơi gọi. Hàm `getApplication()` chỉ có sẵn bên trong `Activity` hoặc `Service` và trả về đúng kiểu `Application`. Còn `applicationContext` gọi được từ bất kỳ biến `Context` nào và trả về kiểu chung `Context`.

**Hỏi:** Truyền Activity Context vào thư viện Room Database sẽ gây ra hậu quả gì?
Màn hình đó sẽ bị rò rỉ trên bộ nhớ Heap sau khi người dùng bấm thoát. Vì đối tượng Room Database thường là singleton sống suốt từ lúc mở app, biến `Activity` bị giữ bên trong cũng không được dọn. Nếu thư viện không tự đổi sang `applicationContext`, cả cây giao diện cũ sẽ nằm kẹt lại trong RAM.

## Ý bắt buộc khi giải thích lại
- [ ] Phân biệt được Activity kế thừa ContextThemeWrapper (có Theme và window token) với Application chỉ kế thừa ContextWrapper.
- [ ] Giải thích được vì sao truyền Activity vào singleton làm rò rỉ cả cây View trên Heap.
- [ ] Nêu được lý do dùng applicationContext tạo Dialog gây sập ứng dụng với lỗi BadTokenException.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

