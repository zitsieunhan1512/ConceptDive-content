---
id: architecture-clean-architecture-and-use-cases
title: Clean Architecture & Use Cases
domain: architecture
tags: [architecture, clean-architecture, use-case, domain-layer, dependency-inversion]
collections: [interview-fundamental]
related: [architecture-dependency-injection, architecture-mvvm-vs-mvi, architecture-state-vs-event, architecture-offline-first-repository]
created: 2026-09-15
diagrams: [Cơ chế|clean-architecture-dependency-rule.png|Tầng UI và Data phụ thuộc hướng tâm vào Domain thuần Kotlin theo nguyên lý đảo ngược phụ thuộc]
---

## Bản 30 giây
Nhồi nhét logic nghiệp vụ vào ViewModel hoặc Repository khiến mã nguồn dính chặt vào Android SDK và khó kiểm thử. Kiến trúc sạch giải quyết bằng cách cô lập tầng Domain độc lập hoàn toàn, đại diện bởi các ca sử dụng (*Use Cases*). Mỗi Use Case chỉ đảm nhiệm một hành vi nghiệp vụ duy nhất, giúp kiểm thử nhanh chóng mà không cần thiết bị giả lập.

## Cần biết trước
- architecture-dependency-injection
- architecture-mvvm-vs-mvi

## Vấn đề buộc nó ra đời
Năm 2012, Robert C. Martin công bố mô hình Clean Architecture để giải quyết vấn đề mã nguồn bị phụ thuộc quá mức vào các framework ngoại vi. Các ứng dụng phần mềm thường bị sụp đổ cấu trúc khi thư viện bên thứ ba thay đổi.

Trong Android, ViewModel ban đầu đóng vai trò tiếp nhận dữ liệu từ Repository. Nhưng khi logic phình to, ViewModel biến thành các lớp khổng lồ chứa hàng ngàn dòng mã trộn lẫn tính toán và điều hướng.

Sự xuất hiện của tầng Domain với các Use Case chuyên biệt giúp bóc tách từng luồng nghiệp vụ riêng lẻ. Mã nguồn trở nên trong sáng, dễ đọc và cho phép nhiều kỹ sư làm việc song song mà không xung đột.

## Nó thực sự là cái gì
`Use Case` KHÔNG phải là một lớp chuyển tiếp dữ liệu trung gian vô bổ chỉ làm động tác gọi lại hàm của Repository. Thực chất, đây là ranh giới đóng gói quy tắc nghiệp vụ cốt lõi và điều phối dữ liệu từ nhiều nguồn.

Tầng Domain chứa Use Case tuyệt đối không chứa bất kỳ câu lệnh nhập xuất (*import*) nào từ thư viện Android.

## Cơ chế
Quy tắc phụ thuộc (*Dependency Rule*) quy định các tầng bên ngoài chỉ được phép phụ thuộc hướng vào tầng bên trong. Tầng Domain định nghĩa các giao diện (*Interfaces*) của Repository mà không cần quan tâm dữ liệu lưu ở đâu.

Tầng Data bên ngoài sẽ triển khai các giao diện này dựa trên nguyên lý [đảo ngược phụ thuộc](concept:architecture-dependency-injection). Mỗi Use Case thường nạp chồng toán tử gọi hàm (*operator invoke*) để tạo cú pháp gọi trực tiếp tự nhiên.

## Đánh đổi
Phân tách tầng giúp việc viết Unit Test đạt tốc độ tính bằng phần nghìn giây và tái sử dụng logic tối đa. Khi phối hợp với mô hình [Offline-First](concept:architecture-offline-first-repository), Use Case đóng vai trò kết nối mượt mà giữa các nguồn dữ liệu.

Đổi lại, số lượng tệp tin và lớp dữ liệu trung gian tăng lên nhanh chóng trong toàn bộ dự án. Với các tính năng CRUD đơn giản, việc tạo Use Case chỉ để gọi một hàm Repository là sự lãng phí cấu trúc.

## Nguyên lý gốc
Nguyên lý đơn trách nhiệm: mỗi thành phần chỉ nên có một và chỉ một lý do duy nhất để thay đổi. Sự độc lập khỏi công cụ hiển thị giúp logic nghiệp vụ trường tồn qua nhiều thế hệ framework công nghệ.

## Key insight
> Use Case không phải lớp bọc vô thưởng vô phạt, mà là tường lửa ngăn Android SDK xâm lấn vào logic nghiệp vụ.

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
Tạo Use Case chỉ làm nhiệm vụ ủy quyền một dòng duy nhất cho Repository là bẫy thừa thãi cấu trúc (*Over-engineering*). Chống chỉ định nhập (*import*) bất kỳ lớp Android nào như Context hoặc Bundle vào bên trong tầng Domain. Triệu chứng gồm lỗi kiểm thử `Method not mocked`, mã nguồn phình to và phá vỡ tính tương thích đa nền tảng (*KMP*).

## Câu hỏi thực chiến
**Hỏi:** Có bắt buộc mọi hàm trong ViewModel đều phải đi qua Use Case không?
Không bắt buộc. Nếu chỉ đọc dữ liệu đơn giản không có quy tắc nghiệp vụ, ViewModel có thể gọi thẳng Repository để tránh thừa code.

**Hỏi:** Tại sao tầng Domain trong Clean Architecture tuyệt đối cấm import thư viện Android?
Để bảo toàn tính độc lập nền tảng. Khi không phụ thuộc Android SDK, toàn bộ logic nghiệp vụ có thể chạy kiểm thử trên máy ảo JVM với tốc độ cực cao.

**Hỏi:** Một Use Case có thể gọi một Use Case khác không?
Có thể, nhưng cần hạn chế. Chỉ nên gọi chéo khi một quy trình nghiệp vụ lớn cần tái sử dụng một luật nghiệp vụ con đã được kiểm chứng.

## Ý bắt buộc khi giải thích lại
- [ ] Tầng Domain và Use Case độc lập 100% khỏi Android Framework.
- [ ] Dependency Rule: tầng ngoài phụ thuộc tầng trong qua đảo ngược phụ thuộc.
- [ ] Tránh bẫy tạo Use Case rỗng chỉ gọi lại Repository (Over-engineering).
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu

