# SOLID trong ngữ cảnh Laravel

> Laravel cung cấp cơ chế nhận request và nối object; framework không tự quyết định ranh giới thiết kế đúng.

## Bản đồ từ checkout

| Nguyên tắc | Liên tưởng Laravel | Câu hỏi |
|---|---|---|
| SRP | Controller, Form Request, use case | Quy tắc nào đổi độc lập với HTTP? |
| OCP | Notification channel, storage disk, payment adapter | Có biến thể thật cần thêm không? |
| LSP | Nhiều adapter cùng contract | Chúng giữ cùng input, kết quả và lỗi không? |
| ISP | Interface theo khả năng bên gọi cần | Checkout có cần toàn API notification không? |
| DIP | Contract ứng dụng được container nối | Use case có phụ thuộc SDK cụ thể không? |

## BAD — controller thực hiện toàn bộ checkout

**Snippet minh họa, không chạy độc lập.**

~~~php
final class CheckoutController
{
    public function store(CheckoutRequest $request): JsonResponse
    {
        // Tính discount, gọi Stripe, lưu order, cập nhật inventory,
        // gửi receipt và chuyển lỗi provider thành HTTP response...
    }
}
~~~

HTTP mapping, quy tắc giá, payment và persistence cùng nằm trong action. Khó test giá riêng nếu phải dựng framework/provider.

## GOOD — controller giữ ranh giới HTTP

**Snippet minh họa Laravel; domain class thuộc ứng dụng.**

~~~php
final class CheckoutController
{
    public function __construct(private CompleteCheckout $checkout) {}

    public function store(CheckoutRequest $request): JsonResponse
    {
        $validated = $request->validated();
        $result = $this->checkout->handle(
            new OrderId((string) $validated['order_id']),
        );

        return response()->json([
            'order_id' => $result->orderId->value,
            'payment_reference' => $result->paymentReference->value,
        ]);
    }
}
~~~

Controller chỉ chuyển input đã validate sang `OrderId`, gọi API thống nhất `CompleteCheckout::handle(OrderId): CheckoutResult`, rồi định dạng HTTP response. Nó vẫn có trách nhiệm HTTP; use case giữ điều phối nghiệp vụ.

## Container nối object; contract vẫn do ứng dụng thiết kế

Laravel container tự resolve nhiều class concrete. Với interface, ứng dụng phải chỉ ra implementation cần chọn. Laravel 13 hỗ trợ `#[Bind]` trên interface hoặc binding tường minh trong service provider ([Service Container, Laravel 13.x](https://laravel.com/docs/13.x/container), xem 2026-10-23):

~~~php
use Illuminate\\Container\\Attributes\\Bind;

#[Bind(StripePaymentGateway::class)]
interface PaymentGateway
{
    public function charge(Money $amount): PaymentReference;
}
~~~

Nếu lựa chọn phụ thuộc cấu hình, service provider có thể làm điểm lắp ghép:

~~~php
$this->app->bind(PaymentGateway::class, StripePaymentGateway::class);
~~~

Đây là hai cách wiring; không cần dùng cả hai cho cùng binding. Container thực hiện dependency injection, nhưng không kiểm tra LSP, không tự làm thiết kế OCP và không quyết định SRP/ISP. Facade cũng không tự động vi phạm DIP; cần đánh giá dependency và test seam cụ thể ([Contracts, Laravel 13.x](https://laravel.com/docs/13.x/contracts), xem 2026-10-23).

## Nhận diện trong code review

Thay “controller/service chưa SOLID” bằng câu hỏi cụ thể:

1. Yêu cầu nào thay đổi?
2. Phần nào bị sửa theo dù không liên quan?
3. Ranh giới mới bảo vệ thay đổi nào?
4. Chi phí mới là interface, binding hay indirection nào?
5. Có phương án đơn giản hơn vẫn đáp ứng nhu cầu không?

## Trade-off và giới hạn

Không cần use case class cho mọi route CRUD. Dùng contract Laravel hay contract ứng dụng tùy ranh giới cần bảo vệ. Không bọc từng facade/service chỉ để tránh framework; wrapper không tạo giá trị nếu không tách thay đổi hoặc làm test/adapter rõ hơn.

## Tự kiểm tra

1. Trong GOOD, phần nào vẫn thuộc HTTP?
2. Nếu một route chỉ CRUD đơn giản, cần use case riêng không?
3. Ai kiểm tra các `PaymentGateway` implementation giữ cùng contract?

## Nguồn

- Laravel Documentation 13.x, [Service Container](https://laravel.com/docs/13.x/container), xem 2026-10-23.
- Laravel Documentation 13.x, [Contracts](https://laravel.com/docs/13.x/contracts), xem 2026-10-23.

---

[← Trước: DIP](05-dip.md) · [Tiếp: GRASP →](../grasp.md)
