---
id: architecture-dependency-injection
title: Dependency Injection
domain: architecture
tags: [architecture, dependency-injection, testing, modularity]
collections: [interview-fundamental]
related: [oop-solid]
created: 2026-09-07
sources:
  - Dependency injection in Android | https://developer.android.com/training/dependency-injection
  - Inversion of Control and Dependency Injection | https://martinfowler.com/articles/injection.html
  - Dependency injection — Wikipedia | https://en.wikipedia.org/wiki/Dependency_injection
diagrams: [Cơ chế|dependency-injection-testing-isolation.png|Tiêm phụ thuộc qua constructor giúp cô lập kiểm thử bằng cách tráo đổi bản giả lập không sửa code]
---

## Bản 30 giây
Khi một lớp tự gọi lệnh tạo công cụ bên trong, bạn không thể tráo bằng công cụ giả để chạy unit test. Dependency Injection đổi ngược lại: bên ngoài tạo sẵn công cụ rồi truyền vào cho đối tượng dùng. Nhờ đó, bạn tách hẳn việc dùng công cụ ra khỏi việc khởi tạo công cụ.

## Cần biết trước
- oop-solid

## Vấn đề buộc nó ra đời
Đầu thập niên 2000, cách viết test trước khi viết code (TDD) phát triển mạnh. Các kỹ sư nhận ra những lớp tự gọi `new ApiClient()` ở bên trong thì không sao viết unit test riêng lẻ được. Cứ chạy test là hàm lại gọi mạng thật hoặc ghi xuống ổ cứng, vừa chậm vừa không giả lập được lúc rớt mạng.

Tháng 1/2004, Martin Fowler viết bài luận nổi tiếng về IoC Containers. Ông đặt tên cho cách làm này là Dependency Injection (tiêm phụ thuộc), tức là truyền đối tượng cần dùng từ bên ngoài vào.

Khi dự án lớn dần, việc tự tay khởi tạo rồi truyền hàng chục tham số qua nhiều tầng rất mỏi tay. Các thư viện như Guice, Dagger và Hilt ra đời để tự động nối các đối tượng này giúp bạn.

## Nó thực sự là cái gì
Dependency Injection không phải là thư viện Hilt hay Koin mà bạn bắt buộc phải cài vào dự án. Nó chỉ là một cách tổ chức code. Thay vì tự gọi lệnh khởi tạo, đối tượng nhận các công cụ cần dùng từ bên ngoài truyền vào.

## Cơ chế
Bên ngoài tạo sẵn các đối tượng phụ thuộc rồi truyền vào lớp của bạn qua `constructor` hoặc gán vào biến thuộc tính (field injection). Khi đi cùng nguyên lý [DIP](concept:oop-solid), lớp của bạn chỉ nhận vào một `interface` thay vì lớp cụ thể. Lúc chạy ứng dụng thật thì bạn truyền bản gọi mạng thật, còn lúc chạy test thì truyền bản giả lập vào.

## Đánh đổi
Code của bạn tách hẳn việc khởi tạo ra khỏi việc tính toán nghiệp vụ nên viết unit test rất nhẹ nhàng. Đổi lại, dự án có thêm nhiều file hơn và chỗ ghép nối các đối tượng lúc khởi động sẽ phức tạp hơn. Thư viện sinh code lúc biên dịch như Dagger làm thời gian build lâu hơn. Còn thư viện tra cứu lúc chạy như Koin dễ làm sập ứng dụng nếu quên khai báo.

## Nguyên lý gốc
Đảo ngược điều khiển (Inversion of Control): nhường việc tạo đối tượng cho bên ngoài lo. Nhờ đó, lớp bên trong dễ thay đổi và dễ kiểm thử.

## Key insight
> Dependency Injection chỉ là truyền công cụ từ bên ngoài vào thay vì tự tạo bên trong, không phụ thuộc vào thư viện.

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
Bẫy hay gặp là dùng một biến toàn cục (Service Locator) để lấy công cụ ngay giữa thân hàm thay vì nhận qua `constructor`. Cách này giấu mất lớp đang cần những gì và làm các bài test sau bị dính dữ liệu cũ của bài test trước. Còn nếu `constructor` nhận quá nhiều tham số, lớp đang ôm quá nhiều việc và vi phạm nguyên lý đơn trách nhiệm trong [SOLID](concept:oop-solid). Đừng cài các thư viện DI đồ sộ vào một thư viện nhỏ độc lập vì chỉ làm dự án build chậm thêm.

## Câu hỏi thực chiến
**Hỏi:** Không dùng Hilt hay Koin thì có làm Dependency Injection được không?
Được, không bắt buộc phải có thư viện. DI chỉ là truyền đối tượng cần dùng từ bên ngoài vào thay vì tự tạo ở bên trong. Bạn tự tay viết code khởi tạo rồi truyền qua `constructor` (Manual DI) đã là làm đúng bản chất của DI rồi.

**Hỏi:** Service Locator với Dependency Injection khác nhau ở đâu khi đều lấy được phụ thuộc?
Khác nhau ở chỗ bạn có nhìn thấy ngay lớp đó cần những công cụ gì hay không. Với Service Locator, lệnh lấy công cụ nằm lẫn trong ruột hàm nên không đọc hết code thì không biết lớp cần gì. Còn DI đặt hết các công cụ cần dùng ngay trên `constructor`, truyền thiếu một thứ là trình biên dịch báo lỗi ngay.

**Hỏi:** Dự án nên chọn Hilt hay Koin, làm sao biết?
Hilt kiểm tra cây phụ thuộc lúc build nên thiếu gì là báo lỗi liền, hợp cho dự án lớn. Còn Koin nối đối tượng lúc chạy nên build nhanh hơn. Nhưng nếu quên khai báo một lớp trong Koin, mở màn hình đó lên ứng dụng mới sập.

**Hỏi:** DI có tác dụng thực sự gì ngoài việc hỗ trợ viết Unit Test?
Nó tách nơi dùng công cụ ra khỏi nơi cài đặt và khởi tạo công cụ đó. Khi đổi từ lưu file sang cơ sở dữ liệu Room, bạn chỉ đổi chỗ truyền vào mà không sửa code nghiệp vụ.

**Hỏi:** Constructor nhận tới tám tham số thì dùng DI thế nào cho gọn?
Bạn đừng tìm cách giấu bớt tham số, vì đó là dấu hiệu lớp này đang ôm quá nhiều việc và vi phạm [SOLID](concept:oop-solid). Cách sửa đúng là tách lớp đó thành các `Use Case` hoặc dịch vụ nhỏ hơn rồi mới truyền phụ thuộc vào.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được nhu cầu tráo đối tượng giả khi viết unit test chính là lý do sinh ra Dependency Injection.
- [ ] Phân biệt được việc tự truyền tham số qua constructor (Manual DI) với việc dùng thư viện như Hilt hay Koin.
- [ ] Chỉ ra được bẫy dùng Service Locator giấu phụ thuộc trong thân hàm và bẫy constructor nhận quá nhiều tham số.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
