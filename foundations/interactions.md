# Khi các nguyên tắc tương tác

## Vì sao cần đọc bài này

Trong code thật, một quyết định hiếm khi chỉ liên quan một nguyên tắc. Câu hỏi hữu ích không phải “nguyên tắc nào thắng”, mà là “thay đổi/rủi ro nào đang cần giảm, và đánh đổi nào chấp nhận được”.

## DRY và cohesion: giống code chưa chắc là cùng kiến thức

Hai phép kiểm tra có thể trông giống nhau nhưng thuộc hai quy tắc nghiệp vụ khác nhau. Nếu gom chúng vào một helper chỉ vì cùng vài dòng, một thay đổi về quy tắc A có thể vô tình ảnh hưởng B. Ngược lại, nếu hai nơi cùng mã hóa một giới hạn nghiệp vụ, chúng có thể là hai bản sao của một kiến thức và sẽ lệch nhau khi yêu cầu đổi.

**Câu hỏi chẩn đoán:** nếu quy tắc thay đổi, các vị trí này có luôn cần thay đổi đồng thời vì cùng một lý do không? Nếu không, giữ riêng có thể đúng; nếu có, tìm một representation có thẩm quyền rồi kiểm tra chi phí dùng chung.

## OCP và YAGNI: bảo vệ điểm biến động đã biết

OCP khuyến khích bảo vệ phần ổn định khỏi loại thay đổi lặp lại. YAGNI nhắc rằng extension point cũng có chi phí: interface, registry, wiring và đường điều khiển mới.

Một nhánh match theo ba mức giảm giá ổn định có thể dễ đọc hơn một hệ thống policy có thể cấu hình. Khi lịch sử thay đổi cho thấy thêm chính sách thường xuyên và thuật toán chung cần ổn định, abstraction có căn cứ hơn. OCP không yêu cầu mở rộng mọi khả năng; YAGNI không yêu cầu bỏ qua biến động đã quan sát.

## SRP và GRASP High Cohesion: liên quan nhưng không cùng câu hỏi

SRP hỏi một module có đang phục vụ nhiều nhóm quyết định thay đổi độc lập không. High Cohesion hướng đến việc đặt các trách nhiệm liên quan gần nhau trong một phần thiết kế. Một class có thể có nhiều thao tác nhưng cohesion cao nếu chúng phục vụ cùng một mục đích; một trách nhiệm nghiệp vụ có thể được phân cho nhiều object nếu điều đó làm ranh giới và phối hợp rõ hơn.

**Câu hỏi chẩn đoán:** nhóm nào yêu cầu thay đổi, và các dữ liệu/hành vi nào tự nhiên cần phối hợp để thực hiện trách nhiệm?

## ISP và LSP: interface nhỏ vẫn cần contract tốt

ISP giúp bên sử dụng không phải phụ thuộc vào thao tác họ không cần. Nhưng tách interface thành các capability nhỏ không bảo đảm implementation thay thế giữ đúng hành vi. Một interface SendReceipt có thể có chữ ký thống nhất nhưng adapter vẫn được phép im lặng bỏ qua lỗi hoặc gửi hai lần nếu contract không nói rõ điều gì được hứa.

**Câu hỏi chẩn đoán:** consumer có nhìn thấy capability thừa không, và mọi implementation có cùng precondition, postcondition, lỗi và side effect không?

## DIP và DI: hướng phụ thuộc khác cách cấp object

DIP là nguyên tắc về hướng phụ thuộc source code và quyền sở hữu abstraction. Dependency Injection là cách cấp dependency từ bên ngoài thay vì tự tạo bên trong. Constructor injection có thể giúp test, nhưng nếu use case vẫn type-hint Stripe SDK thì việc inject SDK chưa đảo hướng kiến trúc.

**Câu hỏi chẩn đoán:** abstraction thuộc về bên nào, và module nghiệp vụ có thể được biên dịch/hiểu mà không biết SDK cụ thể không?

## Quy trình ra quyết định khi có xung đột

1. Ghi một yêu cầu thay đổi thực tế.
2. Nêu failure mode hoặc chi phí hiện tại.
3. Chọn đúng một ranh giới để thay đổi.
4. Viết test hành vi trước/sau nếu hành vi có thể quan sát được.
5. So sánh chi phí trước và sau: số khái niệm, nơi wiring, phạm vi sửa và khả năng kiểm tra.
6. Giữ lại phương án đơn giản hơn nếu lợi ích chưa đo được hoặc chưa có bằng chứng biến động.

## Checklist code review

- Có vấn đề hiện hữu nào cần giải quyết, hay chỉ là dự phòng cho giả định?
- Abstraction có tên theo nhu cầu của consumer hay theo SDK hiện tại?
- Hai đoạn code giống nhau có cùng một chủ sở hữu quy tắc không?
- GOOD giảm loại coupling nào và tạo thêm indirection nào?
- Có thể thêm biến thể bằng wiring rõ ràng hay phải sửa registry trung tâm?
- Test kiểm tra hành vi hay chỉ xác nhận cấu trúc class?

## Nguồn

- David Thomas, Andrew Hunt, *The Pragmatic Programmer, 20th Anniversary Edition*, mục DRY; [trích đoạn do nhà xuất bản cung cấp](https://media.pragprog.com/titles/tpp20/dry.pdf), xem 2026-10-23.
- Martin Fowler, [Yagni](https://martinfowler.com/bliki/Yagni.html), xem 2026-10-23.
- Craig Larman, [Applying UML and Patterns, chương GRASP](https://www.oreilly.com/library/view/applying-uml-and/0131489062/ch17.html), ấn bản 3, 2004; xem 2026-10-23.

---

[← Trước: YAGNI](../simplicity-and-change/yagni.md) · [Tiếp: Glossary →](glossary.md)
