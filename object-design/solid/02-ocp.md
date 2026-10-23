# 03 · O — Open/Closed Principle

## Cách phát biểu nền tảng và bản dịch sát

**Cách diễn đạt kinh điển của OCP (Martin dẫn lại Meyer):** “Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.” ([Martin, 1996](https://d3s.mff.cuni.cz/f/teaching/nprg043/extras/martin96-open_closed_principle.pdf); Meyer, 1988).

**Dịch sát:** Các thực thể phần mềm (class, module, function, v.v.) nên mở cho việc mở rộng, nhưng đóng đối với việc sửa đổi.

## Hiểu đơn giản

**Muốn thêm cách làm mới thì bổ sung nó ở chỗ đã dành cho việc mở rộng, để phần lõi đang ổn định ít phải sửa theo.**

## Áp dụng vào case checkout

**Áp dụng vào case checkout:** thêm hạng khách bằng cách đăng ký discount policy mới; giữ công thức tính amount không đổi.

## Mục tiêu

Chỉ ra phần nào giữ ổn định, loại thay đổi nào được hỗ trợ, file nào vẫn đổi và chi phí extension point.

## Trạng thái checkout

Sau [SRP](01-srp.md), CompleteCheckout gọi OrderPricing. OrderPricing còn match theo tier. Giả sử discount đổi thường xuyên nhưng công thức tổng tiền cần ổn định.

Meyer trình bày nguyên tắc trong *Object-Oriented Software Construction* (1988); Martin trình bày một diễn giải cho thiết kế hướng đối tượng ([Martin, 2014](https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html), xem 2026-10-23). Ở đây, OCP là quyết định bảo vệ phần có biến động thực, không hứa không sửa code cũ.

## Cốt lõi: đóng phần nào, mở cho điều gì?

- **Đóng:** thuật toán tính amount.
- **Mở:** thêm discount policy.
- **Vẫn đổi:** registry/wiring đăng ký tier mới.

Nếu chỉ có vài tier ổn định, match có thể rẻ và rõ hơn.

## Sơ đồ

~~~mermaid
flowchart TD
    A["OrderPricing"] -->|"yêu cầu policy theo hạng"| B["Registry"]
    B -->|"chọn biến thể"| C["DiscountPolicy"]
    C -->|"tính số tiền"| D["Amount phải thu"]
~~~

## BAD — tier mới sửa thuật toán

**Loại snippet: minh họa, không chạy độc lập.**

~~~php
final class OrderPricing
{
    public function amountDueInMinorUnits(Order $order): int
    {
        $basisPoints = match ($order->customerTier) {
            'member' => 1_000,
            'vip' => 2_000,
            'employee' => 3_000,
            default => 0,
        };

        $discount = intdiv(
            ($order->subtotalInMinorUnits() * $basisPoints) + 5_000,
            10_000
        );

        return $order->subtotalInMinorUnits() - $discount;
    }
}
~~~

**Hành vi:** thêm tier sửa cùng method chọn chính sách và tính tổng. Nếu đây là biến động thường xuyên, policy và công thức chung chịu chung rủi ro regression.

## Refactor / GOOD — policy riêng, registry minh bạch

**Loại snippet: ví dụ gần hoàn chỉnh; Order được lược bỏ. Wiring được thể hiện ngay.**

~~~php
interface DiscountPolicy
{
    public function amountDueInMinorUnits(int $subtotal): int;
}

final class PercentageDiscount implements DiscountPolicy
{
    public function __construct(private int $basisPoints)
    {
        if ($basisPoints < 0 || $basisPoints > 10_000) {
            throw new InvalidArgumentException('Discount must be between 0% and 100%.');
        }
    }

    public function amountDueInMinorUnits(int $subtotal): int
    {
        $discount = intdiv(($subtotal * $this->basisPoints) + 5_000, 10_000);

        return $subtotal - $discount;
    }
}

final class DiscountPolicyRegistry
{
    /** @param array<string, DiscountPolicy> $policies */
    public function __construct(private array $policies)
    {
        if (!isset($policies['default'])) {
            throw new InvalidArgumentException('A default discount policy is required.');
        }
    }

    public function forTier(string $tier): DiscountPolicy
    {
        return $this->policies[$tier] ?? $this->policies['default'];
    }
}

final class OrderPricing
{
    public function __construct(private DiscountPolicyRegistry $policies) {}

    public function amountDueInMinorUnits(Order $order): int
    {
        return $this->policies->forTier($order->customerTier)
            ->amountDueInMinorUnits($order->subtotalInMinorUnits());
    }
}

// Composition root: điểm lắp ghép và đăng ký biến thể.
$registry = new DiscountPolicyRegistry([
    'member' => new PercentageDiscount(1_000),
    'vip' => new PercentageDiscount(2_000),
    'default' => new PercentageDiscount(0),
]);
~~~

Thêm employee chỉ thêm entry tại composition root. OrderPricing không biết các tỷ lệ. Unknown tier dùng default—đó là quyết định nghiệp vụ, không phải mặc định kỹ thuật. Tích subtotal × basis points có giới hạn integer; production cần ràng buộc miền giá trị và làm tròn.

## Test hành vi

**Loại snippet: mẫu PHPUnit, cần các lớp ở trên; chưa chạy trong repo Markdown.**

~~~php
public function test_percentage_discount_rounds_half_up_to_minor_unit(): void
{
    self::assertSame(
        9_004,
        (new PercentageDiscount(1_000))->amountDueInMinorUnits(10_005),
    );
}

public function test_unknown_tier_uses_default_policy(): void
{
    $registry = new DiscountPolicyRegistry([
        'default' => new PercentageDiscount(0),
    ]);

    self::assertSame(
        10_000,
        $registry->forTier('new-tier')->amountDueInMinorUnits(10_000),
    );
}
~~~

Các số trong test là minor units của cùng currency; phép làm tròn này giả định kiểu currency hai chữ số thập phân như USD. Nếu nghiệp vụ muốn báo lỗi khi tier chưa đăng ký, đổi registry và test tương ứng.

## Thay đổi, chi phí và khi không nên tách

| | Trước | Sau |
|---|---|---|
| Thêm tier | Sửa match | Thêm policy/entry |
| Công thức | Biết từng tier | Gọi contract |
| Chi phí | Nhánh đơn | Interface, registry, wiring |
| File còn đổi | OrderPricing | Composition root |

OCP chuyển điểm thay đổi, không xóa nó. Với biến thể hiếm và ổn định, abstraction có thể tốn hơn lợi ích.

## Chẩn đoán trước khi refactor

1. Có biến thể thật hoặc lịch sử thay đổi lặp lại không?
2. Biến động nằm ở policy hay thuật toán?
3. Ai chọn implementation và xử lý tier lạ?
4. Extension point nhỏ nhất diễn đạt đúng domain là gì?
5. Có thể test registry và policy độc lập không?

## Nhận diện trong code review

Tín hiệu: cùng một switch lặp lại ở nhiều nơi; mỗi tier mới sửa thuật toán ổn định; registry không có fallback/error rõ; hoặc plugin system phức tạp chỉ để tránh nhánh đơn giản.

Hỏi: “Đóng phần nào, mở cho thay đổi nào, và thêm biến thể sửa chính xác ở đâu?”

## Liên tưởng Laravel

Notification channels/storage disks là extension point hữu ích khi có nhiều biến thể. Container chỉ nối binding; không tự tạo OCP. Đừng xây plugin system khi một match dễ đọc hơn.

## Tips / Lưu ý

- Chỉ mở rộng cho biến động đã quan sát hoặc có căn cứ.
- Nói rõ phần nào được giữ ổn định và file nào vẫn đổi.
- Quy định rõ tier lạ: dùng default hay báo lỗi.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Extension point | Nơi đăng ký/thêm một biến thể mới |
| Policy | Quy tắc quyết định kết quả nghiệp vụ |
| Registry | Nơi tra và chọn policy theo khóa |

## Tự kiểm tra

1. Khi thêm tier, file nào còn đổi?
2. Vì sao đó vẫn có thể là OCP tốt?
3. Khi nào bạn giữ match thay vì registry?

## Nguồn

- Bertrand Meyer, *Object-Oriented Software Construction*, ấn bản thứ nhất, 1988, phần Open-Closed Principle.
- Robert C. Martin, [The Open-Closed Principle](https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html), 2014-05-12; xem 2026-10-23.

---

[← Trước: SRP](01-srp.md) · [Tiếp: LSP →](03-lsp.md)
