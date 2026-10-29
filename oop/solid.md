---
id: oop-solid
title: SOLID
domain: oop
tags: [oop, solid, clean-architecture, software-design]
collections: [interview-fundamental]
related: [oop-four-pillars, architecture-dependency-injection]
sources:
  - SOLID principles — Wikipedia | https://en.wikipedia.org/wiki/SOLID
  - Single-responsibility principle — Wikipedia | https://en.wikipedia.org/wiki/Single-responsibility_principle
  - Dependency inversion principle — Wikipedia | https://en.wikipedia.org/wiki/Dependency_inversion_principle
diagrams: [Cơ chế|solid.png|Năm nguyên tắc SOLID, mỗi nguyên tắc một câu ý chính và một ví dụ]
created: 2026-09-07
---

## Bản 30 giây
Khi ứng dụng thêm tính năng mới, sửa một đoạn code cũ rất dễ làm hỏng những chỗ đang chạy tốt. Năm nguyên lý SOLID giúp bạn chia code thành các khối có ranh giới rõ ràng. Nhờ đó, bạn thêm được tính năng mới mà không làm hỏng dây chuyền cả hệ thống.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Cuối thập niên 1990, các dự án C++ và Java phình to rất nhanh. Chúng rơi vào cảnh mà Robert C. Martin (Uncle Bob) gọi là "sự thối rữa thiết kế" (design rot). Nhiều khi bạn chỉ sửa định dạng in hóa đơn mà làm hỏng luôn phần thanh toán vì code dính chặt vào nhau.

Năm 2000, Uncle Bob viết bài tổng hợp các nguyên tắc thiết kế hướng đối tượng. Đến năm 2004, Michael Feathers ghép các chữ cái đầu thành tên gọi SOLID. Năm nguyên lý này đúc kết từ việc chữa những dự án thực tế cứ sửa một chỗ là hỏng một chùm.

Mỗi chữ cái giải quyết một nỗi khổ quen thuộc: code cứng nhắc, sửa đâu vỡ đó và không đem đi tái sử dụng được. Chúng đặt ra ranh giới để dự án vẫn đứng vững khi yêu cầu nghiệp vụ thay đổi liên tục.

## Nó thực sự là cái gì
SOLID không phải là bộ luật trang trí cho code trông sang hơn. Nó sắp xếp chiều phụ thuộc giữa các lớp. Nhờ đó, khi bạn sửa một chỗ thì phạm vi bị ảnh hưởng chỉ gói gọn rất nhỏ.

## Cơ chế
Chữ S gom các hàm đổi vì cùng lý do về một lớp, còn chữ O thêm tính năng qua lớp mới. Chữ L bắt lớp con phải giữ đúng lời hứa hành vi của lớp cha. Chữ I tách `interface` lớn thành các mảnh nhỏ để nơi gọi không phải gánh hàm thừa. Chữ D đảo chiều phụ thuộc: cả tầng cao lẫn tầng thấp đều nhìn vào `interface` chứ không gọi thẳng lớp cụ thể.

## Đánh đổi
Code dễ viết unit test, dễ tách việc cho nhiều người làm chung và ít hỏng vặt khi đổi yêu cầu. Đổi lại, dự án sinh ra nhiều file và `interface` hơn, khiến bạn phải nhảy qua thêm một tầng trung gian (indirection) khi đọc code. Nếu áp máy móc cho tính năng nhỏ, người đọc phải nhảy qua hàng chục file chỉ để tìm vài dòng lệnh.

## Nguyên lý gốc
Giữ cho các khối ít dính vào nhau: đặt những phần hay thay đổi sau một `interface` ổn định để lỗi không lan dây chuyền.

## Key insight
> SOLID không sinh ra để làm đẹp code, mà để trả lời câu hỏi: sửa chỗ này thì vỡ chỗ nào?

## Ví von
SOLID giống hệ thống vách ngăn chống cháy trên tàu thủy. Khi một khoang bốc cháy, cửa ngăn sập xuống ngay để lửa không lan sang nhấn chìm cả con tàu.

## Bẫy & ranh giới
Bẫy phổ biến là mỗi lớp lại đẻ kèm cặp `FooInterface` và `FooImpl` dù dự án chỉ có một bản cài đặt. Bẫy thứ hai là vi phạm chữ L (Liskov): lớp con ghi đè hàm rồi ném `UnsupportedOperationException`, làm ứng dụng sập lúc chạy. Bạn hãy kết hợp [Dependency Injection](concept:architecture-dependency-injection) để áp dụng chữ D, và nắm vững [Bốn tính chất OOP](concept:oop-four-pillars) làm gốc. Đừng tách `interface` cho những màn hình thêm-sửa-xóa (CRUD) đơn giản vì chỉ làm người đọc mỏi tay bấm tìm file.

## Câu hỏi thực chiến
**Hỏi:** Có phải class nào cũng bắt buộc phải tạo interface tương ứng thì mới chuẩn SOLID?
Không phải. Nếu một lớp chỉ có một cách chạy duy nhất và không cần làm giả khi test, tạo thêm `interface` chỉ làm chật dự án. Bạn chỉ nên tách `interface` khi thực sự có từ hai cách cài đặt trở lên hoặc cần tráo bằng bản giả trong unit test.

**Hỏi:** Khi nào không nên áp dụng đầy đủ năm nguyên lý SOLID?
Khi bạn viết các màn hình CRUD đơn giản, hàm tiện ích ngắn hoặc bản thử nghiệm cần làm nhanh để kiểm chứng ý tưởng. Áp dụng đủ SOLID luôn đẻ thêm nhiều file, `interface` và tầng gọi bắc cầu. Với bài toán nhỏ, làm vậy chỉ khiến bạn phải mở năm bảy file mới đọc hết một luồng chạy.

**Hỏi:** Làm sao nhận biết code đang vi phạm nguyên lý thay thế Liskov (LSP)?
Dấu hiệu rõ nhất là lớp con ghi đè hàm của lớp cha nhưng bên trong lại ném lỗi `UnsupportedOperationException` hoặc để rỗng. Một dấu hiệu nữa là nơi gọi hàm cứ phải dùng `is` kiểm tra xem đối tượng thuộc lớp con nào rồi mới dám gọi. Khi bạn thay lớp con vào chỗ của lớp cha mà ứng dụng chạy sai, nghĩa là hợp đồng Liskov đã bị vỡ.

**Hỏi:** Single Responsibility (SRP) có phải là một class chỉ được làm đúng một việc duy nhất?
Không đúng. SRP nói rằng một lớp chỉ nên có một lý do để thay đổi, tức chỉ phục vụ một nghiệp vụ. Lớp đó vẫn có thể chứa nhiều hàm, miễn là khi nghiệp vụ kia đổi quy định thì các hàm này mới cùng sửa.

**Hỏi:** Nguyên lý đảo ngược phụ thuộc (DIP) và [Dependency Injection](concept:architecture-dependency-injection) có phải là một không?
Không phải. DIP yêu cầu lớp xử lý nghiệp vụ chỉ gọi qua `interface` thay vì gọi thẳng lớp chi tiết bên dưới. Còn [Dependency Injection](concept:architecture-dependency-injection) là cách bạn truyền đối tượng từ bên ngoài vào `constructor`, giúp thực hiện nguyên lý DIP đó.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được SOLID dùng để khoanh vùng chỗ hỏng khi sửa code chứ không phải để làm đẹp mã nguồn.
- [ ] Nêu được ý nghĩa của từng chữ cái S, O, L, I, D bằng ví dụ cụ thể.
- [ ] Chỉ ra được bẫy tạo cặp FooInterface/FooImpl thừa thãi và bẫy lớp con ném UnsupportedOperationException vi phạm Liskov.

## Tôi đã gặp nó ở đâu
