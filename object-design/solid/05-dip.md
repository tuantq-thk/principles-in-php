# 06 · D — Dependency Inversion Principle

## Cách phát biểu nền tảng và bản dịch sát

**Robert C. Martin:** “High-level modules should not depend upon low-level modules. Both should depend upon abstractions. Abstractions should not depend upon details. Details should depend upon abstractions.” ([Martin, 1996](https://objectmentor.com/resources/articles/dip.pdf)).

**Dịch sát:** Module cấp cao không nên phụ thuộc vào module cấp thấp. Cả hai nên phụ thuộc vào trừu tượng. Trừu tượng không nên phụ thuộc vào chi tiết. Chi tiết nên phụ thuộc vào trừu tượng.

## Hiểu đơn giản

**Bên đặt ra nhu cầu định nghĩa hợp đồng; bên cung cấp cách thực hiện tuân theo hợp đồng đó.**

## Áp dụng vào case checkout

Use case `CompleteCheckout::handle(OrderId): CheckoutResult` phụ thuộc vào `PaymentGateway`; adapter Stripe chuyển hợp đồng ứng dụng thành lời gọi provider. Điểm lắp ghép chọn adapter và truyền nó vào use case.

## Cốt lõi: hướng mã nguồn khác hướng runtime

- **Mã nguồn:** `CompleteCheckout` tham chiếu `PaymentGateway`, không import Stripe SDK.
- **Lúc chạy:** composition root tạo Stripe adapter rồi truyền vào use case.
- **Trừu tượng (abstraction/port):** hợp đồng mô tả nhu cầu của ứng dụng.
- **Adapter:** chuyển hợp đồng sang API bên ngoài và ánh xạ kết quả/lỗi về lại.

~~~mermaid
flowchart TD
    A["CompleteCheckout"] -. "phụ thuộc trong source" .-> B["PaymentGateway"]
    C["Stripe adapter"] -. "implements" .-> B
    D["Composition root"] -->|"chọn và cấp"| C
~~~

## BAD — use case tự tạo SDK

**Snippet minh họa, không chạy độc lập.**

~~~php
final class CompleteCheckout
{
    public function handle(OrderId $orderId): CheckoutResult
    {
        $order = Order::findOrFail($orderId->value);
        $stripe = new StripeClient(getenv('STRIPE_SECRET'));
        $response = $stripe->paymentIntents->create([
            'amount' => $order->amountDueInMinorUnits(),
            'currency' => $order->currency(),
        ]);
        // Ghi order và gửi receipt...
    }
}
~~~

Use case đang tự tải dữ liệu, phụ thuộc SDK, biết format provider và tự khởi tạo chi tiết cấu hình. Test nghiệp vụ khó thay provider bằng fake, còn việc đổi SDK lan vào nơi điều phối nghiệp vụ.

## Refactor / GOOD — hợp đồng ứng dụng, adapter và wiring

**Snippet minh họa gần hoàn chỉnh; các domain type và SDK là dependency được lược bỏ. `charge` là API của ứng dụng, không khẳng định tên hàm SDK cụ thể.**

~~~php
interface PaymentGateway
{
    public function charge(Money $amount): PaymentReference;
}

interface OrderRepository
{
    public function get(OrderId $id): Order;
    public function markPaid(Order $order, PaymentReference $reference): PaidOrder;
}

final class CompleteCheckout
{
    public function __construct(
        private OrderRepository $orders,
        private OrderPricing $pricing,
        private PaymentGateway $payments,
        private SendsOrderReceipt $receipts,
    ) {}

    public function handle(OrderId $orderId): CheckoutResult
    {
        $order = $this->orders->get($orderId);
        $amount = new Money(
            $this->pricing->amountDueInMinorUnits($order),
            $order->currency(),
        );
        $reference = $this->payments->charge($amount);
        $paidOrder = $this->orders->markPaid($order, $reference);
        $this->receipts->sendReceipt($paidOrder);

        return new CheckoutResult($paidOrder->id(), $reference);
    }
}

final class StripePaymentGateway implements PaymentGateway
{
    public function __construct(private StripeClient $client) {}

    public function charge(Money $amount): PaymentReference
    {
        // Adapter ánh xạ Money và lỗi provider sang contract ứng dụng.
        $response = $this->client->createPayment($amount);
        return new PaymentReference($response->id);
    }
}

// Composition root: tạo đủ dependency và chọn implementation.
$gateway = new StripePaymentGateway($stripeClient);
$checkout = new CompleteCheckout($orders, $pricing, $gateway, $receipts);
~~~

GOOD giữ cùng nghiệp vụ cốt lõi nhưng chuyển phụ thuộc SDK ra adapter. Đổi provider vẫn đòi đổi adapter, cấu hình và wiring; abstraction không làm chi phí biến mất. Adapter phải giữ contract theo [LSP](03-lsp.md).

## Test hành vi với fake

**Mẫu test PHPUnit; có thể kiểm chứng khi cung cấp các fixture/fake được mô tả bên dưới. Repo tài liệu không kèm project PHPUnit để chạy.**

~~~php
public function test_checkout_charges_gateway_and_returns_result(): void
{
    $order = OrderMother::readyForPayment(Currency::USD, 10_000);
    $orders = new InMemoryOrderRepository($order);
    $gateway = new RecordingPaymentGateway(new PaymentReference('pay_123'));
    $pricing = new OrderPricing(new DiscountPolicyRegistry([
        'default' => new PercentageDiscount(0),
    ]));
    $receipts = new FakeOrderReceiptSender();
    $checkout = new CompleteCheckout($orders, $pricing, $gateway, $receipts);

    $result = $checkout->handle($order->id());

    self::assertSame(1, $gateway->chargeCount());
    self::assertSame(10_000, $gateway->lastCharge()->minorUnits);
    self::assertSame('pay_123', $result->paymentReference->value);
    self::assertTrue($orders->isPaid($order->id()));
}
~~~

Fixture tạo order USD có subtotal 10,000 minor units. Recording gateway giữ Money được charge và trả reference cố định; repository đổi trạng thái ở `markPaid`; receipt fake ghi nhận lệnh gửi. Adapter provider cần test riêng cho mapping request/response và lỗi. Test use case không chứng minh tích hợp Stripe đúng.

## DIP, DI và container

| Khái niệm | Câu hỏi |
|---|---|
| DIP | Policy phụ thuộc abstraction hay detail? |
| Dependency Injection (DI) | Dependency được cấp cho object bằng cách nào? |
| IoC container | Công cụ nào tạo object và nối binding? |

Inject `StripeClient` qua constructor vẫn để policy phụ thuộc vendor. Có thể thực hiện DIP bằng factory thủ công; container không bắt buộc.

## Wiring Laravel 13

**Snippet binding minh họa; tên class và cấu hình tùy ứng dụng.** Laravel 13 hỗ trợ khai báo implementation trên interface bằng `#[Bind]`, hoặc cấu hình binding tường minh trong service provider ([Service Container, Laravel 13.x](https://laravel.com/docs/13.x/container), xem 2026-10-23).

~~~php
use Illuminate\Container\Attributes\Bind;

#[Bind(StripePaymentGateway::class)]
interface PaymentGateway
{
    public function charge(Money $amount): PaymentReference;
}
~~~

Có thể chọn provider qua service provider thay vì attribute, nhất là khi lựa chọn phụ thuộc cấu hình runtime. Đây là wiring, không chứng minh contract, LSP hay ranh giới kiến trúc đúng.

## Thay đổi, chi phí và khi không nên thêm abstraction

| | BAD | GOOD |
|---|---|---|
| Use case biết | Stripe SDK và format request | `PaymentGateway` |
| Đổi provider | Sửa use case cùng SDK call | Adapter, cấu hình và wiring |
| Chi phí | Ít lớp, phụ thuộc chặt | Port, adapter, ánh xạ lỗi, binding |

Nếu SDK ổn định và không cần seam kiểm thử/thay vendor, interface có thể chưa đáng. Hãy chỉ thêm indirection khi nó bảo vệ một thay đổi có cơ sở.

## Chẩn đoán và nhận diện trong code review

1. Use case import/type-hint detail nào?
2. Contract dùng ngôn ngữ ứng dụng hay DTO vendor?
3. Adapter ánh xạ kết quả và lỗi ra sao?
4. Điểm lắp ghép ở đâu, ai chọn implementation?
5. Đổi provider sửa file nào, và lợi ích có xứng chi phí không?

Tín hiệu cần xem: `new StripeClient` trong use case; application layer import SDK; test nghiệp vụ phải boot SDK; binding interface không rõ nơi; hoặc container bị gọi như service locator trong logic nghiệp vụ.

## Tips / Lưu ý

- DIP nói về hướng phụ thuộc; DI nói về cách cấp object.
- Đặt contract ở phía ứng dụng cần nó; giữ kiểu và lỗi vendor trong adapter.
- Nêu rõ điểm lắp ghép và lựa chọn implementation.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Port | Contract ứng dụng định nghĩa cho điều nó cần |
| Adapter | Chuyển đổi contract sang API bên ngoài và ánh xạ kết quả/lỗi |
| Composition root | Điểm tạo object graph và chọn implementation |
| DI | Cách cấp dependency cho object |

## Tự kiểm tra

1. Inject `StripeClient` đã đủ DIP chưa? Vì sao?
2. Khi đổi provider, phần nào vẫn hợp lý phải thay?
3. Ai kiểm tra adapter giữ contract?

## Nguồn

- Robert C. Martin, [The Dependency Inversion Principle](https://objectmentor.com/resources/articles/dip.pdf), 1996; xem 2026-10-23.
- Laravel Documentation 13.x, [Service Container](https://laravel.com/docs/13.x/container), xem 2026-10-23.

---

[← Trước: ISP](04-isp.md) · [Tiếp: Liên tưởng Laravel →](06-laravel.md)
