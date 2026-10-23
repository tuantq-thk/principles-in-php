# Cách đọc repo: nguyên tắc, heuristic và pattern

## Mục đích

Repo tập hợp các lăng kính thiết kế có nguồn gốc và phạm vi khác nhau. Trước khi học chúng như một danh sách, cần biết mỗi loại khái niệm giúp trả lời loại câu hỏi nào.

| Loại | Câu hỏi thường trả lời | Ví dụ trong repo |
|---|---|---|
| Nguyên tắc thiết kế | Một quyết định có dấu hiệu gây vấn đề gì, và nên hướng đến điều gì? | SOLID |
| Nhóm nguyên tắc/mẫu phân công trách nhiệm | Nên đặt trách nhiệm này ở object nào? | GRASP |
| Heuristic / lời khuyên kinh nghiệm | Trong hoàn cảnh chưa chắc chắn, lựa chọn nào thường giảm lãng phí? | DRY, YAGNI |
| Pattern | Một vấn đề lặp lại có cấu trúc giải pháp thường dùng nào? | Strategy được nhắc trong OCP |
| Kiểu kiến trúc | Cách tổ chức các phần lớn và quan hệ giữa chúng là gì? | Ports and Adapters; chưa phải bài hiện tại |

Các nhóm này có thể giao nhau trong cách áp dụng, nhưng không đồng nghĩa. Ví dụ, Strategy là pattern có thể hỗ trợ OCP; việc dùng Strategy không tự chứng minh thiết kế đang tuân thủ OCP.

## Nguyên tắc không phải mệnh lệnh máy móc

Một nguyên tắc giúp đặt câu hỏi và nhận ra rủi ro. Nó không quyết định thay bạn rằng mọi class phải có interface, mọi đoạn lặp phải gộp, hay mọi nhánh điều kiện phải chuyển thành polymorphism.

Trước khi refactor, viết ra:

1. **Thay đổi nào đang xảy ra?** Nêu yêu cầu hoặc lịch sử thay đổi cụ thể.
2. **Tác động hiện tại là gì?** File nào phải sửa, test nào khó viết, hoặc thay đổi nào lan sang phần không liên quan?
3. **Ai sở hữu quyết định?** Nhóm nghiệp vụ, bên dùng API hay hạ tầng?
4. **Ranh giới nhỏ nhất là gì?** Tách trách nhiệm, định nghĩa contract, hay chỉ đặt tên cho một phép tính?
5. **Chi phí mới là gì?** Thêm indirection, wiring, kiểu dữ liệu, interface, test doubles, hoặc nơi cần đồng bộ?
6. **Bằng chứng thành công là gì?** Một thay đổi cụ thể phải trở nên cục bộ hơn, hay một hành vi phải được bảo vệ bằng test.

## Quy ước BAD / GOOD

- BAD phải có input, hành vi và điểm gây khó khăn đủ rõ để người đọc tự nhận ra vấn đề.
- GOOD nên đổi một ranh giới chính. Nếu đồng thời đổi domain model, database, framework và test strategy, không thể biết thay đổi nào tạo ra lợi ích.
- GOOD phải cho thấy wiring: ai tạo/chọn implementation, ai truyền nó vào consumer, và file nào vẫn phải đổi.
- Trade-off phải cụ thể: lợi ích là loại thay đổi nào trở nên rẻ hơn; chi phí là thêm điều gì phải hiểu hoặc duy trì.
- “Tốt hơn” luôn có điều kiện. Một hàm ngắn, một match expression hoặc concrete dependency có thể là lựa chọn đúng khi biến động chưa xuất hiện.

## Mức độ kiểm chứng snippet

| Nhãn | Ý nghĩa |
|---|---|
| Minh họa | Cố ý lược type, fixture hoặc adapter; không chạy độc lập |
| Gần hoàn chỉnh | Thể hiện các thành phần chính, nhưng có thể thiếu bootstrap/SDK |
| Wiring khái niệm | Chỉ ra nơi ghép implementation; không phải cấu hình có thể copy |
| Test mẫu | Nêu hành vi nên khóa; chỉ được gọi là đã chạy khi ghi môi trường và kết quả thực tế |

## Khi hai hướng dẫn kéo ngược nhau

Đừng chọn bằng cách đếm nguyên tắc. Hãy chọn theo rủi ro hiện tại:

- Nếu hai đoạn giống nhau nhưng thay đổi theo hai quy tắc khác nhau, gộp chúng có thể tạo coupling.
- Nếu một abstraction hỗ trợ thêm biến thể chưa có bằng chứng, YAGNI cảnh báo chi phí dự đoán.
- Nếu một đối tượng có nhiều method nhưng tất cả thuộc cùng một trách nhiệm, số method không tự chứng minh SRP bị vi phạm.

Các tình huống này được phân tích sâu hơn trong [bài về tương tác giữa nguyên tắc](interactions.md).

## Nguồn và giới hạn

GRASP được trình bày trong sách của Craig Larman như một nhóm pattern/nguyên tắc để phân công trách nhiệm; cách gọi cụ thể có thể khác theo ấn bản ([O’Reilly: Applying UML and Patterns, chương 17](https://www.oreilly.com/library/view/applying-uml-and/0131489062/ch17.html); kiểm tra 2026-10-23). Các nhãn trong bảng trên là phân loại biên tập của repo để người học không nhầm phạm vi.

---

[← README](../README.md) · [Tiếp: Case study checkout →](../object-design/checkout-case-study.md)
