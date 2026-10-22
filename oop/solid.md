---
id: oop-solid
title: SOLID
domain: oop
tags: [oop, solid, clean-architecture, software-design]
collections: [interview-fundamental]
related: [oop-four-pillars, architecture-dependency-injection]
created: 2026-09-07
---

## Bản 30 giây
Khi phần mềm thêm tính năng, sửa code thường làm hỏng các chỗ cũ đang chạy. Bộ năm nguyên lý SOLID hướng dẫn chia code thành các khối độc lập có ranh giới rõ ràng. Nhờ đó, bạn có thể mở rộng tính năng mà không gây đổ vỡ dây chuyền cho toàn bộ hệ thống.

## Cần biết trước
- oop-four-pillars

## Vấn đề buộc nó ra đời
Cuối thập niên 1990, các hệ thống phần mềm C++ và Java phình to nhanh chóng. Chúng rơi vào tình trạng mà Robert C. Martin (Uncle Bob) gọi là "sự thối rữa thiết kế" (design rot). Chỉ sửa định dạng in ấn cũng làm vỡ module thanh toán do liên kết quá chặt.

Năm 2000, Uncle Bob công bố bài viết tổng hợp các nguyên tắc thiết kế hướng đối tượng. Đến năm 2004, Michael Feathers ghép các chữ cái đầu thành từ viết tắt SOLID. Bộ năm nguyên lý này ra đời từ kinh nghiệm dẹp bỏ mã nguồn dễ gãy trong dự án thực tế.

Mỗi chữ cái giải quyết một nỗi đau cấu trúc cụ thể: mã nguồn cứng nhắc, dễ gãy và không thể tái sử dụng. Chúng là ranh giới kỹ thuật giúp hệ thống chịu đựng được sự thay đổi liên tục của nghiệp vụ.

## Nó thực sự là cái gì
SOLID KHÔNG phải là công thức thần thánh để viết code cho đẹp mắt. Thực chất, nó là chiến lược quản trị phụ thuộc, giúp cô lập phạm vi hỏng hóc khi sửa đổi mã nguồn.

## Cơ chế
S gom hàm đổi cùng lý do vào một lớp, còn O mở rộng qua trừu tượng thay vì sửa mã cũ. L ép lớp con bảo toàn hợp đồng hành vi của lớp cha. I xé nhỏ giao diện để bên gọi không dính hàm thừa. D đảo ngược phụ thuộc: các module đều dựa vào trừu tượng.

## Đánh đổi
Hệ thống có khả năng chịu đựng thay đổi cao, dễ viết unit test và tách nhánh phát triển. Đổi lại, số lượng file và interface tăng vọt, tạo thêm tầng gián tiếp (indirection). Nếu áp dụng mù quáng cho dự án nhỏ, code sẽ bị phân mảnh và khó đọc hơn rất nhiều.

## Nguyên lý gốc
Nguyên lý giảm thiểu liên kết: cô lập các phần biến động sau ranh giới ổn định để dập tắt hiệu ứng domino khi lỗi.

## Key insight
> SOLID không sinh ra để làm đẹp code, mà để trả lời câu hỏi: sửa chỗ này thì vỡ chỗ nào?

## Ví von
Hệ thống vách ngăn chống cháy trên tàu biển. Khi một khoang bị bốc cháy, cửa lập tức sập xuống cô lập khoang đó, ngăn tai nạn nhấn chìm toàn bộ con tàu.

## Bẫy & ranh giới
Tạo cặp đôi vô nghĩa `FooInterface` và `FooImpl` chỉ có một bản cài đặt duy nhất là bẫy thừa thãi phổ biến nhất. Bẫy thứ hai là vi phạm Liskov: lớp con ném lỗi `UnsupportedOperationException` ở hàm kế thừa, gây sập app bất thình lình lúc chạy. Hãy áp dụng [Dependency Injection](concept:architecture-dependency-injection) để hiện thực hóa chữ D, và dùng [Bốn tính chất OOP](concept:oop-four-pillars) làm nền tảng. Chống chỉ định tách interface cho module CRUD đơn giản hoặc script ngắn vì chỉ gây phân mảnh và tăng chi phí điều hướng code.

## Câu hỏi thực chiến
**Hỏi:** Có phải class nào cũng bắt buộc phải tạo interface tương ứng thì mới chuẩn SOLID?
Không phải. Tạo mù quáng từng interface cho mỗi class khi chỉ có duy nhất một nơi dùng chỉ gây rác mã nguồn. Interface chỉ có ích khi thực sự có nhiều bản cài đặt hoặc khi cần làm giả dữ liệu để viết unit test.

**Hỏi:** Khi nào không nên áp dụng đầy đủ năm nguyên lý SOLID?
Khi viết các module CRUD đơn giản, tiện ích ngắn hoặc sản phẩm thử nghiệm cần kiểm chứng nhanh ý tưởng. SOLID luôn đánh đổi bằng việc tăng vọt số lượng file, interface và các tầng trung gian điều hướng. Áp dụng cứng nhắc cho các tác vụ đơn giản chỉ làm mã nguồn bị phân mảnh và khó đọc thêm.

**Hỏi:** Làm sao nhận biết code đang vi phạm nguyên lý thay thế Liskov (LSP)?
Dấu hiệu rõ nhất là lớp con ghi đè hàm kế thừa bằng cách ném ngoại lệ `UnsupportedOperationException` hoặc để trống. Một dấu hiệu phổ biến khác là bên gọi phải dùng lệnh `is` để kiểm tra kiểu cụ thể trước khi thực thi. Khi lớp con không thể thay thế vị trí của cha mà không làm hỏng ứng dụng, hợp đồng LSP đã bị phá vỡ.

**Hỏi:** Single Responsibility (SRP) có phải là một class chỉ được làm đúng một việc duy nhất?
Không đúng. SRP quy định một class chỉ có một lý do duy nhất để thay đổi, tương ứng với một tác nhân nghiệp vụ. Class vẫn có thể chứa nhiều hàm, miễn là yêu cầu sửa đổi chúng luôn đến từ cùng một bộ phận chịu trách nhiệm.

**Hỏi:** Nguyên lý đảo ngược phụ thuộc (DIP) và [Dependency Injection](concept:architecture-dependency-injection) có phải là một không?
Không phải. DIP là nguyên lý thiết kế yêu cầu các mô-đun nghiệp vụ phải phụ thuộc vào giao diện trừu tượng thay vì lớp cụ thể. Còn [Dependency Injection](concept:architecture-dependency-injection) là kỹ thuật chuyển giao đối tượng phụ thuộc từ bên ngoài vào, giúp bạn hiện thực hoá nguyên lý DIP.

## Ý bắt buộc khi giải thích lại
- [ ] Bản chất SOLID là quản trị rủi ro phụ thuộc để giới hạn phạm vi đổ vỡ, không phải thẩm mỹ.
- [ ] Phân biệt được trách nhiệm cốt lõi của từng chữ cái: S, O, L, I, D.
- [ ] Bẫy sinh interface thừa (Interface Soup) và bẫy lớp con phá vỡ hợp đồng ném ngoại lệ vi phạm Liskov.

## Tôi đã gặp nó ở đâu
