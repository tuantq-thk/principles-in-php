# GRASP: phân công trách nhiệm cho object

## Cách phát biểu nền tảng

GRASP là viết tắt của **General Responsibility Assignment Software Patterns**. Craig Larman trình bày các pattern/nguyên tắc này như một cách có hệ thống để suy nghĩ xem trách nhiệm nên được giao cho object nào. O’Reilly mô tả năm nội dung đầu gồm Information Expert, Creator, High Cohesion, Low Coupling và Controller; bốn nội dung còn lại gồm Polymorphism, Indirection, Pure Fabrication và Protected Variations ([Larman, Applying UML and Patterns, chương 17 và 25](https://www.oreilly.com/library/view/applying-uml-and/0131489062/ch17.html), ấn bản 3, 2004; kiểm tra 2026-10-23).

## Hiểu đơn giản

**Hãy đặt hành vi gần dữ liệu cần cho hành vi đó, trừ khi cách đặt ấy làm các phần khác phụ thuộc quá chặt hoặc khiến object ôm quá nhiều việc.**

## Tại sao cần GRASP

Một yêu cầu nghiệp vụ thường không tự nói rõ method phải nằm trong class nào. Đặt hết xử lý ở controller tạo một nơi biết quá nhiều; đẩy mọi method vào entity có thể biến entity thành “god object”. GRASP đưa ra câu hỏi để phân công trách nhiệm có lý do thay vì chọn class theo cảm tính.

## Các pattern/nguyên tắc chính

| Tên | Câu hỏi thực hành | Cảnh báo |
|---|---|---|
| Information Expert | Object nào có thông tin cần thiết để thực hiện trách nhiệm? | Không có nghĩa dữ liệu phải tự truy cập mọi dịch vụ/hạ tầng |
| Creator | Object nào hợp lý để tạo object mới vì chứa, dùng hoặc biết cách khởi tạo nó? | Không giao mọi việc tạo object cho một factory tổng quát |
| High Cohesion | Các trách nhiệm trong object có cùng mục đích và quản lý được không? | Không đo cohesion bằng số method |
| Low Coupling | Có thể giảm hiểu biết trực tiếp giữa các phần không liên quan không? | Không tối ưu coupling đến mức tạo quá nhiều lớp trung gian |
| Controller | Object nào nhận và điều phối một sự kiện hệ thống? | Web controller không nên trở thành nơi chứa toàn bộ nghiệp vụ |
| Polymorphism | Có biến thể theo loại/hành vi nào phù hợp để phân phối qua kiểu? | Chỉ dùng khi có biến thể thật, không phải thay mọi if |
| Indirection | Có cần một trung gian để hai phần ít biết về nhau hơn? | Trung gian cũng làm đường đi khó lần theo |
| Pure Fabrication | Có nên tạo một service không phải khái niệm domain để giảm coupling/tăng cohesion? | Service là lựa chọn thiết kế, không phải mặc định cho mọi nghiệp vụ |
| Protected Variations | Có thể đặt contract quanh một biến động đã nhận diện để phần ổn định ít bị ảnh hưởng không? | Không dự đoán và trừu tượng hóa mọi tương lai |

## Sơ đồ phân công trong checkout

~~~mermaid
flowchart TD
    A["HTTP request"] --> B["CheckoutController"]
    B --> C["CompleteCheckout"]
    C --> D["OrderPricing"]
    C --> E["PaymentGateway"]
    C --> F["OrderRepository"]
~~~

Controller nhận sự kiện; use case điều phối; các cộng tác viên sở hữu khả năng cụ thể. Đây là một lựa chọn cho ví dụ, không phải phân tầng bắt buộc.

## BAD — HTTP controller điều phối toàn bộ checkout

**Loại snippet: minh họa, không chạy độc lập.**

~~~php
final class CheckoutController
{
    public function __construct(
        private OrderRepository $orders,
        private OrderPricing $pricing,
        private PaymentGateway $payments,
        private SendsOrderReceipt $receipts,
        private SendsOrderReceipt $receipts,
    ) {}

    public function store(CheckoutRequest $request): CheckoutResponse
    {
        $order = $this->orders->get($request->orderId());
        $amount = new Money(
            $this->pricing->amountDueInMinorUnits($order),
            $order->currency(),
        );
        $reference = $this->payments->charge($amount);
        $paidOrder = $this->orders->markPaid($order, $reference);
        $this->receipts->sendReceipt($paidOrder);

        return CheckoutResponse::paid($paidOrder->id(), $reference);
    }
}
~~~

**Hành vi quan sát được:** controller vừa chuyển request thành dữ liệu nghiệp vụ, vừa tải đơn, điều phối giá/payment/persistence/receipt và dựng response. Thay đổi HTTP mapping hoặc thứ tự use case đều chạm cùng method.

## GOOD — giao điều phối use case cho application service

**Loại snippet: minh họa ranh giới, không chạy độc lập.**

~~~php
final class CheckoutController
{
    public function __construct(private CompleteCheckout $checkout) {}

    public function store(CheckoutRequest $request): CheckoutResponse
    {
        $result = $this->checkout->handle($request->orderId());

        return CheckoutResponse::paid($result->orderId, $result->paymentReference);
    }
}

final class CompleteCheckout
{
    public function __construct(
        private OrderRepository $orders,
        private OrderPricing $pricing,
        private PaymentGateway $payments,
    ) {}

    public function handle(OrderId $id): CheckoutResult
    {
        $order = $this->orders->get($id);
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
~~~

**Điều thay đổi trong ví dụ:** bốn cộng tác viên và trình tự nghiệp vụ giữ nguyên; trách nhiệm điều phối được chuyển khỏi adapter HTTP vào CompleteCheckout. Controller vẫn chuyển request sang OrderId và định dạng CheckoutResult thành response. Đây là một cách phân công phù hợp với ví dụ, không phải danh sách class bắt buộc.

Information Expert có thể gợi ý để Order tính subtotal nếu dữ liệu dòng hàng thuộc về Order; Pure Fabrication có thể gợi ý OrderPricing nếu việc đặt logic vào entity làm cohesion hoặc phụ thuộc xấu đi.

## Test hành vi

**Loại snippet: test minh họa có thể kiểm chứng khi cung cấp fixture/fake nêu dưới đây; repo không kèm project PHP để chạy.**

~~~php
public function test_checkout_uses_member_discount_and_returns_payment_reference(): void
{
    $order = OrderMother::withTierAndSubtotal(
        CustomerTier::Member,
        subtotalInMinorUnits: 10_000,
        currency: Currency::USD,
    );
    $orders = new InMemoryOrderRepository($order);
    $payments = new RecordingPaymentGateway();
    $pricing = new OrderPricing(new DiscountPolicyRegistry([
        'member' => new PercentageDiscount(1_000),
        'default' => new PercentageDiscount(0),
    ]));
    $receipts = new FakeOrderReceiptSender();

    $result = (new CompleteCheckout($orders, $pricing, $payments, $receipts))
        ->handle($order->id());

    self::assertSame(9_000, $payments->lastCharge()->minorUnits);
    self::assertTrue($orders->isPaid($order->id()));
    self::assertSame($order->id(), $result->orderId);
}
~~~

Test nêu rõ tier Member, currency USD và discount 10%; nó dùng cùng registry/API với bài OCP/DIP. Fixtures và fake là minh họa, không được định nghĩa trong repo. Test tập trung vào kết quả use case, không chứng minh cách phân công là duy nhất hoặc tối ưu.

Fixture cần tạo một Order có ID ổn định, tier `Member`, subtotal `10_000` minor units và currency USD. `RecordingPaymentGateway` phải lưu `Money` nhận được và trả một `PaymentReference`; `InMemoryOrderRepository` phải cập nhật trạng thái khi `markPaid` được gọi; receipt fake chỉ ghi nhận lời gọi. Cấu trúc đó làm cho các assertion trên có nghĩa kiểm chứng được, thay vì dựa vào fixture ngầm.

## Khi nào hữu ích, khi nào quá mức

GRASP hữu ích khi một method phình to, trách nhiệm bị đặt ở controller không phù hợp, hoặc team tranh luận class nào nên sở hữu một hành vi. Nó không đòi tách mọi thao tác thành service. Nếu một quy tắc ngắn chỉ dùng tại một nơi và cùng thay đổi với entity, method trên entity có thể rõ hơn.

## Chẩn đoán và code review

1. Xác định trách nhiệm bằng động từ cụ thể: tính giá, tạo dòng hàng, ghi nhận thanh toán.
2. Liệt kê thông tin cần thiết để thực hiện nó.
3. Tìm object đang nắm thông tin đó; cân nhắc rủi ro khi đặt hành vi ở đó.
4. Kiểm tra cohesion, coupling, khả năng kiểm thử và ai điều phối use case.
5. Chỉ tạo service/intermediary khi nó giải quyết một vấn đề chỉ ra được.

Hỏi trong review: **“Object này có dữ liệu hoặc lý do nghiệp vụ nào để chịu trách nhiệm đó? Nếu chuyển đi, coupling nào giảm và đường đi nào trở nên dài hơn?”**

## Liên tưởng Laravel

Laravel controller thường là điểm vào HTTP, nhưng framework không quyết định nơi nghiệp vụ phải nằm. Form Request phù hợp cho validate/authorization liên quan request; use case/application service có ích khi cùng hành vi được gọi từ HTTP, queue hoặc CLI. Không cần tạo service chỉ để đổi tên một method controller.

## Tips / Lưu ý

- Dùng GRASP để so sánh phương án, không dùng như checklist chín mục.
- “Information Expert” không đồng nghĩa với Active Record hoặc để entity tự gọi database.
- “Low Coupling” luôn có đánh đổi với indirection và khả năng lần theo luồng.
- Controller nhận sự kiện không đồng nghĩa controller là nơi chứa nghiệp vụ.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Responsibility assignment | Phân công trách nhiệm cho object có lý do phù hợp |
| Information Expert | Object có dữ liệu cần thiết để thực hiện một trách nhiệm |
| Pure Fabrication | Object/service được tạo vì lợi ích thiết kế dù không phải khái niệm domain |
| Controller (GRASP) | Object nhận và điều phối sự kiện hệ thống |

## Câu hỏi tự kiểm tra

1. Khi nào Information Expert gợi ý đặt phép tính vào Order, khi nào service riêng hợp lý hơn?
2. Vì sao Pure Fabrication không phải dấu hiệu thiết kế xấu?
3. Controller và use case/application service khác nhau ở loại trách nhiệm nào?

## Nguồn

- Craig Larman, [Applying UML and Patterns, ấn bản 3, chương 17](https://www.oreilly.com/library/view/applying-uml-and/0131489062/ch17.html), 2004.
- Craig Larman, [Applying UML and Patterns, chương 25](https://www.oreilly.com/library/view/applying-uml-and/0131489062/ch25.html), 2004.
- Ngày kiểm tra liên kết: 2026-10-23.

---

[← Trước: DIP](solid/05-dip.md) · [Tiếp: DRY →](../simplicity-and-change/dry.md)
