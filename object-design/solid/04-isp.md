# 05 · I — Interface Segregation Principle

## Cách phát biểu nền tảng và bản dịch sát

**Robert C. Martin:** “Clients should not be forced to depend upon interfaces that they do not use.” ([Martin, 1996](https://d3s.mff.cuni.cz/f/teaching/nprg043/extras/martin96-interface_segregation_principle.pdf)).

**Dịch sát:** Không nên buộc các client phụ thuộc vào những interface mà chúng không sử dụng.

## Hiểu đơn giản

**Đừng đưa người ta cả chùm chìa khóa khi họ chỉ cần một chiếc.**

## Áp dụng vào case checkout

**Áp dụng vào case checkout:** Checkout chỉ cần gửi receipt, nên không phụ thuộc API gửi SMS, cập nhật vận chuyển hay promotion.

## Mục tiêu

Nhận ra client bị buộc phụ thuộc contract quá rộng; phân biệt ISP (phụ thuộc thừa) với LSP (hành vi thay thế sai); tránh chia interface vụn vặt.

## Trạng thái checkout

Checkout chỉ cần gửi receipt. Notification module còn SMS, shipping update và promotion. Bài này tách capability receipt; payment port/wiring hoàn thiện ở DIP.

Martin mô tả ISP là client không bị buộc phụ thuộc interface nó không dùng ([Martin, 1996](https://objectmentor.com/resources/articles/isp.pdf), xem 2026-10-23).

## Cốt lõi: interface rộng gây coupling cho ai?

Nhiều method không tự động xấu nếu cùng nhóm consumer cần chúng và chúng thay đổi cùng nhau. Vấn đề là một client chỉ cần một phần nhưng vẫn phải compile, mock hoặc đổi theo phần còn lại.

## Sơ đồ

~~~mermaid
flowchart TD
    A["Checkout"] -->|"BAD: phụ thuộc"| B["OrderNotifier rộng"]
    B --> C["Receipt"]
    B --> D["SMS / shipping / promotion"]
    A -. "GOOD: chỉ cần" .-> E["SendsOrderReceipt"]
~~~

## BAD — checkout nhìn thấy API thông báo rộng

**Loại snippet: minh họa, không chạy độc lập.**

Đây là lát cắt chỉ tập trung dependency receipt; các dependency payment, pricing và repository của use case được lược bỏ.

~~~php
interface OrderNotifier
{
    public function sendReceipt(Order $order): void;
    public function sendSms(string $phone, string $message): void;
    public function sendShippingUpdate(Order $order): void;
    public function sendPromotion(Customer $customer): void;
}

final class CompleteCheckout
{
    public function __construct(private OrderNotifier $notifier) {}

    public function sendReceipt(Order $order): void
    {
        $this->notifier->sendReceipt($order);
    }
}
~~~

Checkout gọi một method nhưng type-hint toàn API. Thêm method có thể buộc implementer/test double không liên quan cập nhật. Nếu adapter email để sendSms rỗng hoặc ném “not supported”, đó còn là tín hiệu LSP.

## Refactor / GOOD — contract theo capability

**Loại snippet: minh họa, không chạy độc lập.**

~~~php
interface SendsOrderReceipt
{
    public function sendReceipt(Order $order): void;
}

interface SendsShippingUpdate
{
    public function sendShippingUpdate(Order $order): void;
}

final class CompleteCheckout
{
    public function __construct(private SendsOrderReceipt $receipts) {}

    public function sendReceipt(Order $order): void
    {
        $this->receipts->sendReceipt($order);
    }
}
~~~

Email adapter implement SendsOrderReceipt; shipping workflow dùng SendsShippingUpdate. Một class có thể implement cả hai nếu có đủ hành vi.

## Test hành vi

**Loại snippet: mẫu PHPUnit, cần Order fixture; chưa chạy trong repo Markdown.**

~~~php
public function test_checkout_uses_receipt_capability(): void
{
    $receipts = new class implements SendsOrderReceipt {
        public ?Order $sent = null;

        public function sendReceipt(Order $order): void
        {
            $this->sent = $order;
        }
    };
    $order = OrderMother::aPaidOrder();

    (new CompleteCheckout($receipts))->sendReceipt($order);

    self::assertSame($order, $receipts->sent);
}
~~~

Fake chỉ implement capability checkout cần; không phải viết method giả cho SMS/shipping.

## Mối liên hệ với LSP

- **ISP:** client bị buộc nhìn/phụ thuộc method không cần.
- **LSP:** implementation không giữ hành vi contract đã hứa.

Tách contract xử lý coupling phía consumer; từng implementation vẫn phải đúng contract nhỏ mà nó nhận.

## Thay đổi, chi phí và khi không nên tách

| | Contract rộng | Theo capability |
|---|---|---|
| Thêm SMS | Có thể ảnh hưởng implementer/test double không liên quan | Receipt consumer không đổi |
| Chi phí | Ít interface | Nhiều contract phải đặt tên/quản lý |

Nếu mọi consumer dùng method cùng thay đổi, interface chung có thể rõ hơn. Đừng tách một interface cho mỗi method theo máy móc.

## Chẩn đoán trước khi refactor

1. Liệt kê consumer, đánh dấu method mỗi client dùng.
2. Xác định nhóm method thay đổi cùng nhau.
3. Tìm no-op, “not supported”, mock method dư hoặc package dependency thừa.
4. Tách theo nhu cầu client và giữ contract hành vi rõ.

## Nhận diện trong code review

Tín hiệu: test checkout phải mock shipping; thay promotion làm client receipt sửa; adapter ném “not supported”; hoặc use case phụ thuộc API framework rộng chỉ để gửi một receipt.

Hỏi: “Consumer này cần chính xác capability nào? Nó đang phụ thuộc phần nào ngoài nhu cầu?”

## Liên tưởng Laravel

Laravel Notifications có nhiều channel; use case có thể chỉ cần “gửi receipt”. Để framework ở adapter nếu ranh giới đó giảm coupling thật. Không cần bọc từng method Laravel.

## Tips / Lưu ý

- Xem interface từ phía từng consumer; một interface nhiều method không tự động sai.
- No-op và lỗi “not supported” là tín hiệu cần xem cả ISP lẫn LSP.
- Chia theo nhóm thao tác thường được dùng/thay đổi cùng nhau, không chia một method một interface.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Client/consumer | Code sử dụng một interface |
| Capability | Một khả năng cụ thể client cần |
| Fat interface | Interface buộc client thấy nhiều thao tác ngoài nhu cầu |

## Tự kiểm tra

1. Ai chịu ảnh hưởng khi OrderNotifier thêm SMS?
2. Vì sao not-supported còn gợi ý vấn đề LSP?
3. Khi nào interface sáu method vẫn cohesive?

## Nguồn

- Robert C. Martin, [The Interface Segregation Principle](https://objectmentor.com/resources/articles/isp.pdf), 1996; xem 2026-10-23.
- Laravel Documentation 13.x, [Notifications](https://laravel.com/docs/13.x/notifications), xem 2026-10-23.

---

[← Trước: LSP](03-lsp.md) · [Tiếp: DIP →](05-dip.md)
