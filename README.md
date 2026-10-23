# Principles in PHP

> Cẩm nang tiếng Việt về các nguyên tắc thiết kế phần mềm, học qua ví dụ PHP và quyết định có trade-off.

Repo này bắt đầu từ SOLID và mở rộng sang các hướng dẫn thiết kế có liên quan. Mục tiêu không phải biến mọi nguyên tắc thành luật, mà giúp người đọc nhận ra vấn đề cụ thể, chọn thay đổi nhỏ nhất có ích, rồi giải thích chi phí của lựa chọn đó.

**Phạm vi:** tài liệu Markdown thuần, không có source project hay test runner. Snippet được đánh dấu là minh họa, wiring khái niệm hoặc test mẫu; không coi chúng là đã chạy. Các ví dụ nhắm cú pháp PHP 8.2+; phiên bản và ngày kiểm chứng của claim liên quan được ghi trong bài. Tài liệu tham khảo phiên bản PHP hiện hành có thể đổi theo thời gian.

## Bắt đầu từ đâu

| Thứ tự | Tài liệu | Vì sao đọc |
|---|---|---|
| 1 | [Cách dùng và phân loại](foundations/how-to-read.md) | Phân biệt nguyên tắc, heuristic, pattern và kiểu kiến trúc |
| 2 | [Case study checkout](object-design/checkout-case-study.md) | Theo một thiết kế từ BAD đến các bước refactor |
| 3 | [Bản đồ SOLID](object-design/solid/00-overview.md) | Năm câu hỏi về thay đổi, contract và phụ thuộc |
| 4 | SRP → OCP → LSP → ISP → DIP | Đi từ trách nhiệm đến biến động, thay thế và hướng phụ thuộc |
| 5 | [GRASP](object-design/grasp.md) | Phân công trách nhiệm giữa các object |
| 6 | [DRY](simplicity-and-change/dry.md) → [YAGNI](simplicity-and-change/yagni.md) | Tránh lặp kiến thức và tránh xây khả năng chưa có căn cứ |
| 7 | [Các nguyên tắc tương tác](foundations/interactions.md) | Nhìn trade-off khi nhiều nguyên tắc cùng tác động |
| Tra cứu | [Glossary](foundations/glossary.md) | Nghĩa các thuật ngữ được dùng thống nhất |

## Bản đồ nội dung

~~~text
principles-in-php/
├── README.md
├── foundations/
│   ├── how-to-read.md
│   ├── interactions.md
│   └── glossary.md
├── object-design/
│   ├── checkout-case-study.md
│   ├── grasp.md
│   └── solid/
│       ├── 00-overview.md
│       ├── 01-srp.md
│       ├── 02-ocp.md
│       ├── 03-lsp.md
│       ├── 04-isp.md
│       ├── 05-dip.md
│       └── 06-laravel.md
└── simplicity-and-change/
    ├── dry.md
    └── yagni.md
~~~

## Quy ước của mỗi bài

Các bài nguyên tắc dùng chung khung: phát biểu nền tảng và nguồn → diễn giải dễ nhớ → vấn đề/điều kiện áp dụng → trạng thái case study (nếu phù hợp) → sơ đồ → BAD có hành vi quan sát được → GOOD thay đổi một ranh giới chính → test hoặc cách kiểm chứng → thay đổi được hỗ trợ và chi phí mới → khi không nên áp dụng → chẩn đoán/code review → tips → glossary → câu hỏi tự kiểm tra → nguồn.

GOOD không được coi là tốt chỉ vì có thêm interface hay class. Bài phải nêu rõ wiring, phần vẫn cần đổi, và chi phí mới. Với nguyên tắc không phù hợp tự nhiên với checkout, ví dụ độc lập được ưu tiên thay vì nhồi vào case.

## API checkout dùng xuyên suốt

Sau bước DIP, các bài GRASP/DIP/Laravel dùng cùng API ứng dụng: `CompleteCheckout::handle(OrderId $orderId): CheckoutResult`. Use case tải đơn, tính giá, charge qua `PaymentGateway`, ghi nhận payment và gửi receipt. Các bài SRP/OCP/ISP cố ý trình bày lát cắt trung gian để tập trung một nguyên tắc; phần [case study](object-design/checkout-case-study.md) ghi rõ giả định và giới hạn transaction/idempotency.

## Cách học

1. Đọc BAD và diễn đạt bằng lời hành vi gây khó khăn.
2. Xác định yêu cầu thay đổi cụ thể cùng phần chịu ảnh hưởng.
3. Đọc GOOD, tìm đúng ranh giới vừa thay đổi và xem wiring.
4. Nêu được lợi ích, chi phí mới và trường hợp mà BAD ban đầu hợp lý hơn.
5. Dùng câu hỏi code review để kiểm tra quyết định trong code thật.

## PHP và kiểm chứng

Ví dụ dùng cú pháp tương thích PHP 8.2 trở lên. PHP 8.2 đang ở giai đoạn hỗ trợ bảo mật đến 2026-12-31 theo [Supported Versions](https://www.php.net/supported-versions.php); kiểm tra trạng thái lại trước khi chọn runtime triển khai. Enums có từ PHP 8.1; readonly classes và DNF types có từ PHP 8.2 ([PHP Manual: Enumerations](https://www.php.net/manual/en/language.types.enumerations.php), [PHP 8.2 new features](https://www.php.net/manual/en/migration82.new-features.php); kiểm tra 2026-10-23).

Repo hiện là tài liệu, không có dependency cài đặt hay kết quả PHPUnit/PHPStan/Psalm được tuyên bố. Khi snippet có thể chạy độc lập, bài phải nói rõ điều kiện chạy và phiên bản đã xác minh. Không xem một ví dụ minh họa là bằng chứng runtime.

## Nguồn và cập nhật

Ưu tiên bài gốc, nghiên cứu, sách của tác giả/người đặt tên khái niệm, PHP Manual và tài liệu framework chính thức. Link được gắn sát claim; ghi năm xuất bản nếu biết và ngày kiểm tra đối với thông tin có thể thay đổi. Ngày kiểm tra không có nghĩa nội dung nguồn là chuẩn tuyệt đối; các diễn giải thực hành được ghi rõ là lựa chọn biên tập.

## Phạm vi mở rộng

Repo hiện bao gồm SOLID, GRASP, DRY và YAGNI. GRASP được giới thiệu như nhóm mẫu/nguyên tắc phân công trách nhiệm; DRY và YAGNI là hướng dẫn thực hành. Các pattern và kiểu kiến trúc như Ports and Adapters sẽ chỉ được thêm vào một nhóm riêng nếu mở rộng phạm vi sau này, không xếp ngang hàng với các bài hiện có.

---

[Bắt đầu: Cách đọc repo →](foundations/how-to-read.md)
