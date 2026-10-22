---
id: architecture-offline-first-repository
title: Offline-First Repository & SSOT
domain: architecture
tags: [architecture, offline-first, repository, ssot, room, caching]
collections: [interview-fundamental]
related: [architecture-clean-architecture-and-use-cases, kotlin-flow-under-the-hood, architecture-state-vs-event]
created: 2026-09-15
diagrams: [Cơ chế|offline-first-ssot-flow.png|Cơ sở dữ liệu cục bộ là nguồn chân lý duy nhất phát Flow lên UI, mạng chỉ nạp dữ liệu ngầm vào DB]
---

## Bản 30 giây
Ứng dụng phụ thuộc trực tiếp vào mạng internet sẽ lập tức bị tê liệt khi người dùng mất sóng hoặc kết nối chập chờn. Kiến trúc ưu tiên ngoại tuyến giải quyết bằng cách dùng cơ sở dữ liệu cục bộ làm nguồn chân lý duy nhất (*SSOT*). Giao diện luôn phát dữ liệu từ bộ nhớ máy, còn mạng chỉ đóng vai trò cập nhật ngầm vào bộ nhớ.

## Cần biết trước
- kotlin-flow-under-the-hood
- architecture-state-vs-event
- architecture-clean-architecture-and-use-cases

## Vấn đề buộc nó ra đời
Trước năm 2017, hầu hết ứng dụng Android hiển thị dữ liệu trực tiếp từ các phản hồi mạng (*HTTP Responses*). Mỗi lần mở màn hình, người dùng phải nhìn vòng quay tải trang (*ProgressBar*) và nhận thông báo lỗi khi rớt mạng.

Nếu hai màn hình cùng gọi API đọc chung một đối tượng, dữ liệu rất dễ bị lệch pha nhau (*Data Inconsistency*). Người dùng sửa thông tin ở một nơi nhưng nơi khác vẫn hiển thị phiên bản cũ kỹ.

Google phát hành kiến trúc Hướng dẫn xây dựng ứng dụng (*Guide to App Architecture*) nhằm chuẩn hóa mô hình Offline-First. Cơ sở dữ liệu Room trở thành nguồn dữ liệu trung tâm duy nhất của toàn bộ ứng dụng.

## Nó thực sự là cái gì
Nguồn chân lý duy nhất (*SSOT*) KHÔNG phải là việc lưu trữ dữ liệu trùng lặp ở cả bộ nhớ máy và API mạng. Thực chất, đây là một luồng dữ liệu một chiều nghiêm ngặt: tầng hiển thị chỉ quan sát cơ sở dữ liệu nội bộ.

Mạng internet chỉ đóng vai trò là một nguồn cấp đẩy dữ liệu mới vào cơ sở dữ liệu nội bộ mà thôi.

## Cơ chế
Repository trả về một dòng dữ liệu [Flow](concept:kotlin-flow-under-the-hood) liên kết trực tiếp với bảng dữ liệu cục bộ. Khi kích hoạt đồng bộ, Repository gọi dịch vụ mạng và lưu trực tiếp kết quả vào cơ sở dữ liệu.

Cơ sở dữ liệu tự động phát tín hiệu thay đổi qua Flow, giúp giao diện tự động làm mới tức thì. Tầng đồng bộ hoặc máy chủ sẽ phân xử xung đột theo mốc thời gian, không do cơ sở dữ liệu tự quyết định.

## Đánh đổi
Mô hình này giúp ứng dụng khởi động tức thì và hoạt động mượt mà trong điều kiện không có mạng internet. Đổi lại, dung lượng lưu trữ trên thiết bị tăng lên và lập trình viên phải viết thêm mã đồng bộ dữ liệu.

Bài toán giải quyết xung đột khi nhiều thiết bị cùng sửa một bản ghi đòi hỏi thuật toán phức tạp.

## Nguyên lý gốc
Nguyên lý nguồn chân lý duy nhất: dữ liệu chỉ có một nguồn gốc đáng tin cậy duy nhất trong toàn hệ thống. Mọi thực thể khác chỉ được phép sao chép dữ liệu để đọc chứ không được tự ý duy trì trạng thái riêng.

## Key insight
> Mạng internet không phải nguồn cung cấp dữ liệu cho giao diện, nó chỉ là một đầu vào nạp vào bộ nhớ máy.

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
Cập nhật trực tiếp kết quả API lên StateFlow của ViewModel mà không ghi vào cơ sở dữ liệu sẽ phá vỡ tính đồng bộ. Chống chỉ định kích hoạt đồng bộ mạng lặp lại liên tục mà không có cơ chế điều tiết hoặc bộ nhớ đệm tạm thời. Triệu chứng gồm màn hình hiển thị dữ liệu lệch pha, chớp nháy giao diện và cạn kiệt pin do gọi mạng vô tận.

## Câu hỏi thực chiến
**Hỏi:** Tại sao ViewModel không nên nhận dữ liệu trực tiếp từ API mạng mà phải đọc qua Database?
Để bảo đảm tính nhất quán. Khi đọc qua Database, mọi thay đổi từ bất kỳ đâu đều được phản ánh đồng bộ lên toàn bộ các màn hình hiển thị.

**Hỏi:** Xử lý thế nào nếu người dùng thực hiện thao tác sửa đổi khi đang hoàn toàn mất kết nối mạng?
Ghi nhận ngay vào Database nội bộ kèm cờ chờ đồng bộ (*Optimistic Update*). Khi có mạng trở lại, một worker chạy ngầm sẽ đẩy các thay đổi chưa đồng bộ lên máy chủ.

**Hỏi:** Room Database làm thế nào để tự động cập nhật dữ liệu mới lên màn hình mà không cần gọi lại hàm?
Nhờ cơ chế quan sát bảng dữ liệu (*InvalidationTracker*). Khi có thao tác chèn hoặc cập nhật, Room tự động truy vấn lại và đẩy danh sách mới qua dòng Flow.

## Ý bắt buộc khi giải thích lại
- [ ] Local Database là Single Source of Truth (SSOT) duy nhất phát dữ liệu lên UI.
- [ ] Remote Network chỉ ghi dữ liệu vào Local Database chứ không phát thẳng lên UI.
- [ ] Cơ chế Optimistic Update và InvalidationTracker của Room.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

