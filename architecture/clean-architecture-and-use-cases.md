---
id: architecture-clean-architecture-and-use-cases
title: Clean Architecture & Use Cases
domain: architecture
tags: [architecture, clean-architecture, use-case, domain-layer, dependency-inversion]
collections: [interview-fundamental]
related: [architecture-dependency-injection, architecture-mvvm-vs-mvi, architecture-state-vs-event, architecture-offline-first-repository]
created: 2026-09-15
sources:
  - Domain layer — Android App Architecture | https://developer.android.com/topic/architecture/domain-layer
  - Guide to app architecture — Android | https://developer.android.com/topic/architecture
  - The Clean Architecture — Robert Martin | https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
diagrams: [Cơ chế|clean-architecture-dependency-rule.png|Tầng UI và Data phụ thuộc hướng tâm vào Domain thuần Kotlin theo nguyên lý đảo ngược phụ thuộc]
---

## Bản 30 giây
Nếu bạn dồn hết logic tính toán vào `ViewModel` hoặc `Repository`, code sẽ dính chặt vào thư viện Android và rất khó viết test. Clean Architecture tách riêng một tầng `Domain` thuần Kotlin ở giữa, gồm các lớp nhỏ gọi là `Use Case`. Mỗi `Use Case` chỉ lo đúng một việc nghiệp vụ nên bạn chạy unit test trên máy tính cực nhanh mà không cần máy ảo.

## Cần biết trước
- architecture-dependency-injection
- architecture-mvvm-vs-mvi

## Vấn đề buộc nó ra đời
Năm 2012, Robert C. Martin giới thiệu Clean Architecture để trị căn bệnh code nghiệp vụ dính chặt vào thư viện bên ngoài. Nhiều dự án cứ đổi thư viện cơ sở dữ liệu hay giao diện là phải viết lại cả phần tính toán.

Trên Android, ban đầu `ViewModel` chỉ nhận dữ liệu từ `Repository` rồi đưa lên màn hình. Nhưng khi tính năng phức tạp dần, `ViewModel` phình thành lớp khổng lồ hàng ngàn dòng vừa tính tiền giỏ hàng vừa lo chuyển trang.

Tách tầng `Domain` với các `Use Case` riêng giúp bạn cắt nhỏ từng luồng nghiệp vụ ra khỏi `ViewModel`. Code gọn hẳn lại, dễ đọc và nhiều người có thể chia nhau viết các `Use Case` cùng lúc mà không đụng hàng.

## Nó thực sự là cái gì
`Use Case` không phải là một lớp trung gian vô bổ chỉ đứng giữa gọi lại đúng một hàm của `Repository`. Nó là nơi giữ quy tắc nghiệp vụ cốt lõi và gom dữ liệu từ hai hay nhiều `Repository` lại với nhau. Tầng `Domain` chứa `Use Case` tuyệt đối không được `import` bất kỳ lớp nào của thư viện Android.

## Cơ chế
Quy tắc phụ thuộc (*Dependency Rule*) bắt buộc tầng ngoài chỉ được gọi vào tầng trong, không bao giờ đi chiều ngược lại. Tầng `Domain` ở trong chỉ khai báo `interface` của `Repository` chứ không cần biết dữ liệu lấy từ mạng hay từ máy. Tầng `Data` ở ngoài sẽ viết code thực thi các `interface` đó theo nguyên lý [đảo ngược phụ thuộc](concept:architecture-dependency-injection). Mỗi `Use Case` thường đặt tên hàm là `operator fun invoke` để bạn gọi thẳng đối tượng gọn như gọi một hàm.

## Đánh đổi
Tách tầng như vậy giúp bạn chạy unit test mất vài phần nghìn giây và dùng lại được `Use Case` ở nhiều màn hình. Khi đi cùng mô hình [Offline-First](concept:architecture-offline-first-repository), `Use Case` cũng giúp bạn ghép nối các nguồn dữ liệu rất gọn. Đổi lại, dự án sẽ đẻ thêm khá nhiều file và lớp chuyển đổi dữ liệu trung gian. Với những màn hình chỉ đọc rồi hiện danh sách đơn giản (CRUD), tạo thêm `Use Case` chỉ để gọi một dòng `Repository` là vẽ việc.

## Nguyên lý gốc
Nguyên lý đơn trách nhiệm: mỗi lớp chỉ nên có một lý do duy nhất để phải sửa code. Khi tách rời khỏi công cụ hiển thị, logic tính toán của bạn vẫn giữ nguyên dù sau này đổi sang framework giao diện khác.

## Key insight
> `Use Case` không phải lớp bọc gọi hộ `Repository`, mà là bức tường ngăn thư viện Android tràn vào làm bẩn logic nghiệp vụ.

## Khuôn mẫu chuẩn
```kotlin
class GetEligibleDiscountsUseCase(
    private val userRepository: UserRepository,
    private val promoRepository: PromoRepository
) {
    suspend operator fun invoke(userId: String): List<Discount> {
        val user = userRepository.getUser(userId)
        return promoRepository.getPromos().filter { it.isApplicable(user) }
    }
}
```

## Bẫy & ranh giới
Tạo hàng loạt `Use Case` mà bên trong chỉ gọi lại đúng một dòng của `Repository` là bẫy vẽ việc (*Over-engineering*) hay gặp nhất. Bạn cũng tuyệt đối không được `import` các lớp của Android như `Context` hay `Bundle` vào tầng `Domain`. Nếu vi phạm, chạy unit test trên JVM sẽ nổ lỗi `Method not mocked`, dự án phình to và không đem sang Kotlin Multiplatform (*KMP*) được.

## Câu hỏi thực chiến
**Hỏi:** Có bắt buộc mọi hàm trong ViewModel đều phải đi qua Use Case không?
Không bắt buộc. Nếu màn hình chỉ lấy danh sách hiển thị mà không tính toán hay lọc điều kiện, `ViewModel` cứ gọi thẳng `Repository`.

**Hỏi:** Tại sao tầng Domain trong Clean Architecture tuyệt đối cấm import thư viện Android?
Để giữ cho phần nghiệp vụ chạy độc lập với hệ điều hành. Khi không dính tới Android SDK, bạn chạy được toàn bộ unit test thẳng trên JVM của máy tính trong chớp mắt.

**Hỏi:** Một Use Case có thể gọi một Use Case khác không?
Có thể, nhưng bạn nên hạn chế. Chỉ nên gọi khi một luồng nghiệp vụ lớn cần dùng lại một bước kiểm tra hay tính toán đã viết sẵn ở `Use Case` nhỏ.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được vì sao tầng Domain và Use Case phải thuần Kotlin, không import thư viện Android.
- [ ] Trình bày được Dependency Rule: tầng UI và Data ở ngoài phụ thuộc ngược vào tầng Domain ở trong.
- [ ] Chỉ ra được bẫy tạo Use Case rỗng chỉ gọi chuyển tiếp một dòng cho Repository (Over-engineering).
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

