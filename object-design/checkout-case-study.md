# Case study · Checkout đơn hàng

Đây là ví dụ nối các bài SOLID. Mỗi bài lấy một thay đổi cụ thể làm trọng tâm; các đoạn GOOD là lát cắt học tập, không phải một ứng dụng hoàn chỉnh.

## Bối cảnh và giả định

Checkout tải đơn theo ID, tính discount, thu tiền qua provider, lưu trạng thái đã thanh toán và gửi biên nhận. Những nhóm thay đổi có thể khác nhau: kinh doanh sở hữu discount; nhóm payment sở hữu provider; vận hành sở hữu persistence; chăm sóc khách hàng sở hữu receipt. Đây là giả định để học, không thay cho việc kiểm tra lịch sử thay đổi của hệ thống thật.

**Giả định tiền tệ:** các con số trong ví dụ là integer minor units của cùng một currency; các ví dụ số học cụ thể giả định currency có hai chữ số thập phân như USD. Không được suy ra rằng mọi currency đều có hai chữ số. Production phải để Money/currency biết exponent và adapter chuyển đúng format mà payment provider yêu cầu. Ví dụ discount dùng basis points: 100 basis points = 1%.

## BAD ban đầu

**Loại snippet: minh họa, không chạy độc lập.** Domain types, SDK và xử lý lỗi được lược bỏ. Phép tính float chỉ minh họa code bị gom trách nhiệm, không dùng cho tiền thật.

~~~php
final class OrderProcessor
{
    public function __construct(private PDO $database) {}

    public function checkout(Order $order): void
    {
        $percent = match ($order->customerTier) {
            'member' => 10,
            'vip' => 20,
            default => 0,
        };
        $amount = (int) round(
            $order->subtotalInMinorUnits() * (100 - $percent) / 100
        );

        $stripe = new StripeClient(getenv('STRIPE_SECRET'));
        $stripe->paymentIntents->create([
            'amount' => $amount,
            'currency' => $order->currency,
        ]);

        $statement = $this->database->prepare(
            'UPDATE orders SET status = :status WHERE id = :id'
        );
        $statement->execute(['status' => 'paid', 'id' => $order->id]);
        (new EmailNotifier())->sendReceipt($order, $amount);
    }
}
~~~

Quy tắc giá, SDK payment, SQL và receipt cùng bị điều phối trong một method. Giá trị integer ở đây không tự bảo đảm currency mapping đúng; xem giả định tiền tệ ở trên.

## Trình tự học

| Bài | Ranh giới được khảo sát | Điều vẫn cần thay đổi |
|---|---|---|
| SRP | Tách phép tính OrderPricing khỏi xử lý checkout | Checkout vẫn gọi implementation cụ thể |
| OCP | Đưa biến thể discount vào DiscountPolicy và registry | Composition root thêm entry cho tier mới |
| LSP | Định nghĩa hành vi PaymentGateway cho mọi provider | Adapter/provider vẫn cần test contract |
| ISP | Để checkout chỉ thấy SendsOrderReceipt | Workflow khác có thể cần capability khác |
| DIP | Đưa port về phía ứng dụng và SDK vào adapter | Adapter, cấu hình và wiring vẫn đổi khi thay provider |

Đây là tiến trình tư duy chứ không phải yêu cầu thêm đủ các lớp trên cho mọi dự án.

## API use case thống nhất sau DIP

Các bài GRASP, DIP và Laravel dùng cùng một hợp đồng use case:

~~~php
final class CompleteCheckout
{
    public function handle(OrderId $orderId): CheckoutResult;
}
~~~

`CheckoutResult` chứa định danh Order và PaymentReference. `Money` chứa số nguyên minor units cùng currency. Các kiểu domain được lược bỏ ở chữ ký rút gọn; các bài sau chỉ trích phần liên quan nguyên tắc. Case study giả định payment được chấp nhận trước khi ghi trạng thái paid; thứ tự này không giải quyết crash/retry, xem giới hạn transaction bên dưới.

Luồng khái niệm:

~~~text
HTTP controller
  → CompleteCheckout::handle(OrderId)
    → OrderRepository::get(OrderId)
    → OrderPricing::amountDueInMinorUnits(Order)
    → PaymentGateway::charge(Money)
    → OrderRepository::markPaid(Order, PaymentReference)
    → SendsOrderReceipt::sendReceipt(PaidOrder)
  → CheckoutResult
~~~

Composition root (điểm lắp ghép object graph) chọn StripePaymentGateway, repository và receipt adapter rồi cấp vào use case. Laravel service provider là một vị trí có thể làm việc đó.

## Thanh toán, persistence và giới hạn giao dịch

Database transaction chỉ có thể bảo vệ các thao tác database tham gia cùng giao dịch. Nó không biến lời gọi payment provider thành phần atomic của commit đó. Không giữ DB transaction mở quanh network call với giả định rằng rollback DB sẽ đảo ngược khoản thu.

Một tình huống cần thiết kế recovery: provider đã thu tiền nhưng tiến trình chết trước khi `markPaid` được commit. Order có thể còn pending trong khi tiền đã thu. Retry mù có thể thu lần hai; không retry có thể để trạng thái treo. Vì vậy hệ thống thật cần một luồng trạng thái và đối soát rõ ràng.

**Idempotency** là bảo đảm một lần thử lại cùng thao tác không tạo thêm hiệu ứng nghiệp vụ ngoài dự kiến. Khóa idempotency của provider thường giúp provider nhận diện request lặp; nó không tự đảm bảo ứng dụng lưu order đúng, gửi receipt đúng một lần, hay các provider khác có cùng semantics. Cần xác định khóa ổn định cho cùng lần checkout, phạm vi/thời hạn khóa theo tài liệu provider, cách xử lý request đồng thời, kết quả timeout không rõ, và cách tra/đối soát trạng thái. Ví dụ Stripe lưu kết quả cho cùng key và có chính sách thời hạn/validation riêng; không khái quát hành vi đó cho mọi API ([Stripe API: Idempotent requests](https://docs.stripe.com/api/idempotent_requests), kiểm tra 2026-10-23).

Tùy nghiệp vụ, recovery có thể dùng trạng thái payment pending, retry có kiểm soát, webhook/đối soát định kỳ, transactional outbox cho event nội bộ, hoặc thao tác bù trừ. Outbox giúp ghi event cùng database state rồi phát sau; bản thân nó cũng không tạo transaction phân tán với provider. Case study không chọn một production workflow cụ thể; mục đích là nêu chính xác ranh giới và câu hỏi cần quyết định.

## Phạm vi chưa giải quyết

- Không triển khai transaction, payment state machine, retry, idempotency, webhook/đối soát, outbox, refund hay chargeback; các thuật ngữ này chỉ nêu giới hạn và câu hỏi thiết kế.
- Không giả định mọi provider hỗ trợ cùng currency, lỗi hoặc semantics.
- Không coi test mẫu là test đã chạy: repo không chứa project PHP hay test suite.
- Không coi sơ đồ/các lớp minh họa là kiến trúc bắt buộc cho một checkout đơn giản.

---

[← README](../README.md) · [Tiếp: Bản đồ SOLID →](solid/00-overview.md)
