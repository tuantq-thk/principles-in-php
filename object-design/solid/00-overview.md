# 01 · Bản đồ SOLID

> SOLID là năm lăng kính để hỏi về thay đổi, contract và phụ thuộc. Chúng không phải checklist buộc thêm class/interface.

## Năm câu hỏi trong checkout

| Nguyên tắc | Câu hỏi |
|---|---|
| SRP | Ai có thể yêu cầu đổi giá, payment, lưu order hoặc receipt? |
| OCP | Thêm discount mới có sửa công thức ổn định không? |
| LSP | Provider thay thế có giữ contract hành vi không? |
| ISP | Checkout có phụ thuộc thao tác notification không dùng không? |
| DIP | Use case có import SDK payment trực tiếp không? |

Đọc [case study](../checkout-case-study.md) để theo trạng thái BAD ban đầu và cách mỗi bài thêm đúng một ranh giới.

## Chẩn đoán trước khi refactor

1. Nêu thay đổi cụ thể, không viện dẫn “có thể cần mở rộng”.
2. Tìm nơi thay đổi lan sang code không liên quan.
3. Xác định bên sử dụng (consumer), chủ sở hữu quy tắc và contract họ cần.
4. Chọn ranh giới nhỏ nhất có thể giảm tác động.
5. Tính chi phí mới: indirection, interface, wiring, test và khả năng truy vết.

## Các khái niệm không đồng nghĩa

| Khái niệm | Không tự động chứng minh |
|---|---|
| DIP | Constructor injection đã đảo hướng phụ thuộc |
| PHP interface | Implementation giữ cùng semantics/hành vi |
| OCP | Không file nào phải đổi |
| SRP | Mỗi class chỉ có một method |
| Dùng container | Ranh giới kiến trúc đã hợp lý |

PHP type system kiểm tra tương thích kiểu; contract hành vi cần mô tả và test riêng. PHP hỗ trợ covariance/contravariance từ 7.4, nhưng điều này không chứng minh business semantics ([PHP Manual](https://www.php.net/manual/en/language.oop5.variance.php), xem 2026-10-23).

## Trade-off và giới hạn

Tách module có thể giảm coupling (mức phụ thuộc), nhưng tăng indirection (phải lần qua thêm lớp), khái niệm và nơi wiring (lắp ghép implementation). Nếu chưa có biến động cụ thể, một nhánh điều kiện có thể rõ hơn abstraction. YAGNI là lời nhắc tránh xây khả năng chưa cần, không cấm chuẩn bị cho rủi ro đã biết ([Fowler](https://martinfowler.com/bliki/Yagni.html), xem 2026-10-23).

## Tự kiểm tra

1. Với provider mới, phần nào hợp lý vẫn phải đổi?
2. Vì sao một interface đúng chữ ký vẫn có thể vi phạm LSP?
3. Nêu trường hợp class cụ thể tốt hơn interface.

## Nguồn

- Robert C. Martin, [The Principles of OOD](https://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod), xem 2026-10-23.
- Martin Fowler, [Yagni](https://martinfowler.com/bliki/Yagni.html), xem 2026-10-23.

---

[← Trước: Case study checkout](../checkout-case-study.md) · [Tiếp: SRP →](01-srp.md)
