---
id: architecture-offline-first-repository
title: Offline-First Repository & SSOT
domain: architecture
tags: [architecture, offline-first, repository, ssot, room, caching]
collections: [interview-fundamental]
related: [architecture-clean-architecture-and-use-cases, kotlin-flow-under-the-hood, architecture-state-vs-event]
created: 2026-09-15
sources:
  - Build an offline-first app — Android | https://developer.android.com/topic/architecture/data-layer/offline-first
  - Data layer — Android App Architecture | https://developer.android.com/topic/architecture/data-layer
  - Save data using Room — Android | https://developer.android.com/training/data-storage/room
diagrams: [Cơ chế|offline-first-ssot-flow.png|Cơ sở dữ liệu cục bộ là nguồn chân lý duy nhất phát Flow lên UI, mạng chỉ nạp dữ liệu ngầm vào DB]
---

## Bản 30 giây
Nếu chỉ lấy dữ liệu thẳng từ mạng, màn hình sẽ trắng xóa hoặc báo lỗi khi vào thang máy mất sóng. Kiến trúc Offline-First chọn cơ sở dữ liệu trong máy làm nguồn chân lý duy nhất (*SSOT*). Màn hình luôn đọc từ bộ nhớ máy, còn mạng chỉ tải dữ liệu mới về cất vào cơ sở dữ liệu đó.

## Cần biết trước
- kotlin-flow-under-the-hood
- architecture-state-vs-event
- architecture-clean-architecture-and-use-cases

## Vấn đề buộc nó ra đời
Trước năm 2017, hầu hết ứng dụng Android lấy kết quả trả về từ mạng (*HTTP Responses*) rồi vẽ thẳng lên màn hình. Cứ mở sang màn hình mới là người dùng lại phải nhìn vòng xoay `ProgressBar`, và hễ rớt mạng là màn hình trống trơn.

Nếu hai màn hình cùng gọi API lấy thông tin một người dùng, dữ liệu rất dễ bị lệch nhau (*Data Inconsistency*). Bạn vừa đổi tên ở màn hình cài đặt xong, quay ra màn hình chính vẫn thấy hiện tên cũ.

Sau đó, Google đưa ra tài liệu Hướng dẫn kiến trúc ứng dụng (*Guide to App Architecture*) để chuẩn hóa cách làm Offline-First. Cơ sở dữ liệu Room trở thành nơi duy nhất phát dữ liệu cho toàn bộ màn hình trong ứng dụng.

## Nó thực sự là cái gì
Nguồn chân lý duy nhất (*SSOT*) không có nghĩa là màn hình vừa đọc từ máy vừa đọc từ mạng. Nó là quy tắc một chiều: giao diện chỉ được phép lắng nghe dữ liệu từ cơ sở dữ liệu trong máy. Còn mạng internet chỉ đóng vai trò là nơi lấy dữ liệu mới về để ghi đè vào cơ sở dữ liệu đó.

## Cơ chế
`Repository` trả về một dòng [Flow](concept:kotlin-flow-under-the-hood) nối trực tiếp với bảng dữ liệu trong Room. Khi cần làm mới dữ liệu, `Repository` gọi API mạng rồi cất thẳng kết quả vào bảng dữ liệu trong máy. Ngay khi bảng có dữ liệu mới, Room tự bắn bản cập nhật qua `Flow` để màn hình vẽ lại ngay. Nếu dữ liệu máy và mạng lệch nhau, bước đồng bộ hoặc máy chủ sẽ phân xử theo mốc thời gian.

## Đánh đổi
Cách làm này giúp ứng dụng mở lên có chữ đọc ngay và vẫn dùng bình thường khi đi vào chỗ mất sóng. Đổi lại, ứng dụng tốn thêm bộ nhớ lưu trữ trên điện thoại và bạn phải viết thêm code đồng bộ dữ liệu. Bài toán khó nhất là xử lý xung đột khi người dùng sửa cùng một mục trên hai điện thoại lúc đang mất mạng.

## Nguyên lý gốc
Nguồn chân lý duy nhất: mỗi mẩu dữ liệu chỉ có đúng một nơi giữ bản gốc đáng tin cậy trong cả ứng dụng. Các màn hình chỉ đọc lại từ nơi đó chứ không tự giữ bản sao riêng lẻ.

## Key insight
> Mạng internet không cấp dữ liệu trực tiếp cho màn hình, mà chỉ tải dữ liệu về cất vào cơ sở dữ liệu trong máy.

## Khuôn mẫu chuẩn
```kotlin
class ArticleRepository(
    private val localDao: ArticleDao,
    private val remoteApi: ArticleApi
) {
    fun getArticles(): Flow<List<Article>> = localDao.getArticlesFlow()

    suspend fun syncArticles() {
        val remoteData = remoteApi.fetchLatestArticles()
        localDao.upsertArticles(remoteData)
    }
}
```

## Bẫy & ranh giới
Nếu gán thẳng kết quả API vào `StateFlow` của `ViewModel` mà bỏ qua Database, dữ liệu giữa các màn hình sẽ lệch nhau. Cũng đừng gọi hàm đồng bộ mạng liên tục mỗi khi vẽ lại màn hình mà không kiểm tra thời gian chờ. Triệu chứng là hai màn hình hiện số khác nhau, danh sách chớp nháy và máy tụt pin nhanh vì gọi mạng mãi.

## Câu hỏi thực chiến
**Hỏi:** Tại sao ViewModel không nên nhận dữ liệu trực tiếp từ API mạng mà phải đọc qua Database?
Để mọi màn hình luôn hiện khớp nhau. Khi cùng đọc một bảng Database, bạn sửa ở màn hình chi tiết thì màn hình danh sách bên ngoài tự đổi theo.

**Hỏi:** Xử lý thế nào nếu người dùng thực hiện thao tác sửa đổi khi đang hoàn toàn mất kết nối mạng?
Bạn cứ lưu ngay thay đổi đó vào Database trong máy kèm một cờ đánh dấu "chờ gửi mạng" (*Optimistic Update*). Giao diện đổi ngay cho mượt, rồi khi có mạng lại thì tác vụ ngầm sẽ đẩy phần sửa lên máy chủ.

**Hỏi:** Room Database làm thế nào để tự động cập nhật dữ liệu mới lên màn hình mà không cần gọi lại hàm?
Room có sẵn bộ theo dõi bảng dữ liệu tên là `InvalidationTracker`. Khi bảng có lệnh thêm, sửa hay xóa, Room tự chạy lại truy vấn rồi đẩy danh sách mới vào `Flow`.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao cơ sở dữ liệu trong máy là nguồn chân lý duy nhất (SSOT) phát dữ liệu lên màn hình.
- [ ] Nêu được luồng chạy một chiều: API mạng chỉ ghi vào Database, rồi Database mới bắn Flow lên giao diện.
- [ ] Trình bày được cách lưu tạm bằng Optimistic Update khi mất mạng và cách InvalidationTracker của Room tự báo dữ liệu mới.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

