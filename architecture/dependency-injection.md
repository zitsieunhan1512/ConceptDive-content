---
id: architecture-dependency-injection
title: Dependency Injection
domain: architecture
tags: [architecture, dependency-injection, testing, modularity]
collections: [interview-fundamental]
related: [oop-solid]
created: 2026-09-07
diagrams: [Cơ chế|dependency-injection-testing-isolation.png|Tiêm phụ thuộc qua constructor giúp cô lập kiểm thử bằng cách tráo đổi bản giả lập không sửa code]
---

## Bản 30 giây
Khi một lớp tự khởi tạo các công cụ nó cần, bạn không thể thay thế đồ giả để viết test. Dependency Injection đảo ngược việc này: bên ngoài sẽ đưa sẵn các công cụ vào cho đối tượng sử dụng. Nhờ đó, mã nguồn tách rời việc dùng công cụ khỏi việc tạo công cụ.

## Cần biết trước
- oop-solid

## Vấn đề buộc nó ra đời
Đầu thập niên 2000, phong trào Test-Driven Development (TDD) bùng nổ mạnh mẽ. Kỹ sư nhận ra các class tự gọi `new ApiClient()` bên trong không thể viết Unit Test độc lập. Việc gọi mạng hoặc ghi đĩa thật làm test chạy chậm và không giả lập được lỗi.

Tháng 1/2004, Martin Fowler xuất bản bài luận nổi tiếng về IoC Containers. Ông chính thức đặt tên mẫu hình này là Dependency Injection (tiêm phụ thuộc). Bản chất là đưa vật phụ thuộc từ bên ngoài vào cho đối tượng dùng.

Khi dự án lớn dần, việc tự tay truyền hàng chục tham số qua nhiều tầng trở nên nặng nề. Các framework như Guice, Dagger và Hilt ra đời để tự động hóa đồ thị phụ thuộc này.

## Nó thực sự là cái gì
Dependency Injection KHÔNG phải là thư viện Hilt hay Koin bắt buộc phải cài. Thực chất, nó là mẫu hình: thay vì tự tạo, đối tượng nhận các phụ thuộc từ bên ngoài truyền vào.

## Cơ chế
Bên ngoài khởi tạo các phụ thuộc rồi tiêm vào đối tượng qua constructor hoặc thuộc tính (field injection). Khi kết hợp [DIP](concept:oop-solid), lớp phụ thuộc vào trừu tượng thay vì lớp cụ thể. Khi chạy thật, đối tượng nhận bản cài đặt thật; khi test, ta dễ dàng đưa vào bản giả lập.

## Đánh đổi
Code tách rời logic tạo lập khỏi logic nghiệp vụ và cực kỳ dễ viết test. Đổi lại, số lượng file tăng lên và đồ thị khởi tạo ban đầu trở nên phức tạp. Dùng thư viện sinh mã như Dagger làm kéo dài thời gian build, còn thư viện runtime như Koin dễ lỗi lúc chạy.

## Nguyên lý gốc
Nguyên lý đảo ngược điều khiển (Inversion of Control): nhường quyền khởi tạo cho bên ngoài để tăng tính linh hoạt và dễ kiểm thử.

## Key insight
> Dependency Injection chỉ là việc đưa phụ thuộc từ bên ngoài vào thay vì tự khởi tạo, không đồng nhất với thư viện.

## Khuôn mẫu chuẩn
```kotlin
interface UserRepository { fun getUser(): String }
class ApiUserRepo : UserRepository { override fun getUser() = "Network" }
class FakeUserRepo : UserRepository { override fun getUser() = "Mock" }

class UserPresenter(private val repo: UserRepository) {
    fun displayUser(): String = repo.getUser()
}
```

## Bẫy & ranh giới
Bẫy tai hại là dùng Service Locator toàn cục thay cho constructor injection. Cách này giấu kín các phụ thuộc ngầm và làm test bị nhiễm độc trạng thái tĩnh. Hơn năm tham số constructor là triệu chứng vi phạm nguyên tắc đơn trách nhiệm trong [SOLID](concept:oop-solid). Chống chỉ định dùng framework DI cồng kềnh cho các thư viện độc lập vì làm tăng thời gian build không cần thiết.

## Câu hỏi thực chiến
**Hỏi:** Không dùng Hilt hay Koin thì có làm Dependency Injection được không?
Có. DI là mẫu hình đưa vật phụ thuộc từ bên ngoài vào thay vì tự khởi tạo, hoàn toàn không phụ thuộc vào thư viện. Tự tay khởi tạo và truyền tham số bằng mã thuần (Manual DI) vẫn đáp ứng trọn vẹn bản chất của mẫu hình này.

**Hỏi:** Service Locator với Dependency Injection khác nhau ở đâu khi đều lấy được phụ thuộc?
Khác nhau ở tính minh bạch của các thành phần phụ thuộc. Service Locator giấu việc tìm kiếm đối tượng vào sâu trong ruột hàm, khiến bạn không thể biết class cần gì nếu không đọc mã. Ngược lại, DI phơi bày mọi thứ tại constructor, thiếu bất kỳ đối tượng nào là trình biên dịch sẽ chặn đứng ngay lập tức.

**Hỏi:** Dự án nên chọn Hilt hay Koin, làm sao biết?
Tuỳ vào quy mô dự án và mức độ đánh đổi giữa tốc độ biên dịch với độ an toàn lúc chạy. Hilt kiểm tra đồ thị lúc build giúp phát hiện lỗi thiếu phụ thuộc sớm, rất an toàn cho các ứng dụng lớn. Koin nạp phụ thuộc lúc chạy giúp build mã nhanh hơn, nhưng tiềm ẩn nguy cơ sập app nếu bạn quên khai báo module.

**Hỏi:** DI có tác dụng thực sự gì ngoài việc hỗ trợ viết Unit Test?
Nó cô lập hoàn toàn nơi sử dụng dịch vụ khỏi logic cấu hình và khởi tạo chi tiết. Nhờ vậy, bạn dễ dàng đổi sang nguồn dữ liệu mới mà không phải sửa mã nghiệp vụ.

**Hỏi:** Constructor nhận tới tám tham số thì dùng DI thế nào cho gọn?
Đừng cố tìm cách nhồi nhét, đó là triệu chứng class đang ôm đồm quá nhiều việc và vi phạm [SOLID](concept:oop-solid). Giải pháp đúng đắn là tách class đó thành các UseCase hoặc dịch vụ chuyên biệt nhỏ hơn trước khi tiến hành tiêm phụ thuộc.

## Ý bắt buộc khi giải thích lại
- [ ] Nhu cầu cô lập kiểm thử đơn vị (Unit Test) chính là cái nôi sinh ra DI.
- [ ] Phân biệt được tiêm phụ thuộc thủ công với việc dùng thư viện như Hilt hoặc Koin.
- [ ] Bẫy Service Locator giấu phụ thuộc ngầm và bẫy class nhận quá nhiều tham số.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
