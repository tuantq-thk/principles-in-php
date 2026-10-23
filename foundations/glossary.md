# Glossary · Thuật ngữ trong repo

Bảng này thống nhất cách gọi. Mỗi bài giải thích các thuật ngữ quan trọng tại lần đầu sử dụng; bảng dùng để tra cứu nhanh.

| Thuật ngữ | Cách gọi nhất quán | Nghĩa trong repo |
|---|---|---|
| Abstraction | Trừu tượng | Mô hình hóa điều bên sử dụng cần mà không khóa vào cách thực hiện; thường được thể hiện bằng contract |
| Adapter | Bộ chuyển đổi (adapter) | Chuyển contract ứng dụng sang API/format hệ thống ngoài |
| Actor | Vai trò/nhóm yêu cầu thay đổi | Người hoặc nhóm mà một module phải đáp ứng; dùng để phân tích “lý do thay đổi” của SRP |
| Boundary | Ranh giới | Đường phân cách nơi một bên dùng contract, bên kia cung cấp implementation |
| Capability | Khả năng nghiệp vụ | Thao tác cụ thể client cần, như gửi receipt |
| Cohesion | Độ gắn kết | Mức các phần trong module cùng phục vụ một mục đích |
| Composition root | Điểm lắp ghép dependency | Nơi chọn/tạo implementation rồi truyền vào policy; có thể là bootstrap, provider hoặc factory |
| Consumer / client | Bên sử dụng | Code gọi object và dựa vào hành vi của nó |
| Contract | Hợp đồng hành vi | Kỳ vọng về input, output, lỗi, tác động phụ và bất biến; PHP interface ghi được kiểu và cấu trúc method, không tự diễn tả hết hợp đồng |
| Coupling | Mức phụ thuộc | Mức code cần biết hoặc thay đổi theo phần khác |
| Detail | Chi tiết kỹ thuật | Cơ chế như SDK, database, filesystem hoặc framework |
| Dependency Injection (DI) | Tiêm dependency | Cấp dependency từ ngoài object, thường qua constructor |
| Dependency Inversion Principle (DIP) | Nguyên tắc đảo chiều phụ thuộc | Policy và chi tiết phụ thuộc abstraction phù hợp thay vì policy gọi thẳng detail |
| Extension point | Điểm mở rộng | Nơi cho phép thêm biến thể mà không sửa phần được bảo vệ |
| Implementation | Cách hiện thực | Class cụ thể thực hiện contract |
| Indirection | Lớp trung gian | Bước phải lần qua để tìm code thực thi |
| Invariant | Bất biến | Điều kiện phải luôn đúng |
| IoC container | Container đảo quyền điều khiển | Công cụ tạo object và nối binding; không tự quyết định design |
| Policy | Chính sách nghiệp vụ | Quy tắc quyết định ứng dụng làm gì, như cách tính giảm giá; không đồng nghĩa Laravel Authorization Policy |
| Port | Contract do ứng dụng sở hữu | Giao diện mô tả nhu cầu ứng dụng với hạ tầng |
| Precondition | Điều kiện trước | Điều caller phải bảo đảm trước khi gọi |
| Postcondition | Điều kiện sau | Điều operation hứa sau khi chạy thành công |
| Semantics | Ý nghĩa hành vi | Ý nghĩa operation với caller, không chỉ kiểu trả về |
| Side effect | Tác động phụ | Thay đổi ngoài giá trị trả về, như ghi DB/gọi API |
| Variance | Tính biến thiên kiểu | Quy tắc cho phép kiểu tham số/kiểu trả về thay đổi khi override; tương thích kiểu không tự chứng minh LSP |
| Wiring | Nối dependency | Chọn implementation rồi cung cấp nó cho bên sử dụng |
| Heuristic | Kinh nghiệm định hướng | Gợi ý giúp ra quyết định khi bối cảnh chưa đủ để có quy tắc tuyệt đối |
| Pattern | Mẫu thiết kế | Cách giải quyết có tên cho một vấn đề lặp lại, có thể cần điều chỉnh theo ngữ cảnh |
| Representation | Cách biểu đạt | Nơi kiến thức được lưu trong hệ thống, như code, schema, cấu hình hoặc tài liệu |
| Accidental duplication | Lặp cú pháp tình cờ | Hai đoạn trông giống nhau nhưng biểu đạt hai khái niệm có thể thay đổi độc lập |
| Variation | Biến thể | Một cách thực hiện khác cho cùng một nhu cầu đã được xác định |
| Minor unit | Đơn vị tiền nhỏ nhất | Số nguyên dùng biểu diễn số lượng tiền theo quy ước của currency; số chữ số thập phân tùy currency |
| Currency exponent | Số chữ số thập phân của currency | Quy ước chuyển giữa đơn vị chính và minor unit; không được mặc định mọi currency có hai chữ số |
| Idempotency key | Khóa chống xử lý lặp | Định danh giúp một thao tác được gửi lại mà không tạo thêm hiệu ứng nghiệp vụ; hành vi cụ thể tùy API và thiết kế ứng dụng |
| Database transaction | Giao dịch cơ sở dữ liệu | Nhóm thao tác DB có tính toàn vẹn theo transaction; không làm lời gọi dịch vụ ngoài trở thành atomic cùng DB |

## Quy ước biên tập

- Dùng “bên sử dụng” thay cho consumer sau lần giải thích đầu; giữ consumer khi nói về khái niệm chuẩn.
- Dùng “chính sách nghiệp vụ” cho policy; tránh lẫn Laravel authorization Policy.
- Dùng “chi tiết kỹ thuật” cho SDK/database/framework, không dùng detail trơ trọi.
- Dùng “điểm lắp ghép” cho composition root và “nối dependency” cho wiring.
- Dùng “contract hành vi” khi nói về toàn bộ kỳ vọng; “PHP interface” chỉ construct trong ngôn ngữ.
- Tên API, interface và nguyên tắc giữ nguyên khi dịch làm mất khả năng tra cứu.
- Dùng “heuristic” kèm “kinh nghiệm định hướng”; không gọi DRY/YAGNI là cùng loại nguyên tắc với SOLID.
