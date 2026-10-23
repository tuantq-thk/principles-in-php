# 02 · S — Single Responsibility Principle

## Cách phát biểu nền tảng và bản dịch sát

**Định nghĩa của Robert C. Martin:** “The Single Responsibility Principle (SRP) states that each software module should have one and only one reason to change.” ([Martin, 2014](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)).

**Dịch sát:** Nguyên tắc một trách nhiệm nói rằng mỗi module phần mềm nên có một và chỉ một lý do để thay đổi.

## Hiểu đơn giản

**Một đầu mối nên phục vụ một nhóm người có cùng nhu cầu; khi các nhóm khác nhau liên tục đòi sửa cùng một chỗ, trách nhiệm có thể đang bị gom quá rộng.**

## Áp dụng vào case checkout

**Áp dụng vào case checkout:** nếu nhóm kinh doanh đổi discount, không nên buộc sửa phần gọi payment hoặc ghi đơn khi các thay đổi đó độc lập.

## Mục tiêu

Nhận ra thay đổi nào bị buộc đi qua cùng module, xác định ai sở hữu quy tắc và chọn refactor nhỏ nhất giúp giảm tác động lan truyền.

## Trạng thái checkout

BAD ban đầu trong [case study](../checkout-case-study.md) tính discount, thu tiền, lưu đơn và gửi receipt trong OrderProcessor. Bài này chỉ tách phép tính giá; payment, persistence và thông báo chưa đổi.

Martin gắn “một lý do thay đổi” với actor/nhóm yêu cầu thay đổi, không phải số method ([Martin, 2014](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html), xem 2026-10-23).

## Cốt lõi: tìm nhóm quyết định thay đổi cùng nhau

Độ gắn kết (cohesion) cao khi các phần trong module cùng phục vụ mục đích. Phụ thuộc (coupling) tăng khi thay đổi bên ngoài buộc module biết hoặc sửa theo chi tiết khác. Đây là tín hiệu điều tra, không phải phép đo bằng số method.

## Sơ đồ

~~~mermaid
flowchart TD
    subgraph BAD
        A["Đổi giá"] --> C["OrderProcessor"]
        B["Đổi payment"] --> C
    end
    subgraph GOOD
        D["Đổi giá"] --> E["OrderPricing"]
        F["Đổi payment"] --> G["CompleteCheckout"]
    end
~~~

## BAD — giá nằm giữa điều phối và hạ tầng

**Loại snippet: minh họa, không chạy độc lập.** Domain types được giới thiệu ở case study.

~~~php
final class OrderProcessor
{
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

        // Gọi payment, update database, gửi receipt...
    }
}
~~~

**Hành vi quan sát được:** đổi discount buộc sửa code nằm giữa luồng payment/lưu đơn. Rủi ro nằm ở việc hai lý do thay đổi cùng một nơi, không phải độ dài class.

## Refactor / GOOD — tách phép tính, giữ điều phối

**Loại snippet: minh họa ranh giới, không chạy độc lập.** Dùng minor units và basis points; 1% = 100 basis points.

~~~php
final class OrderPricing
{
    public function amountDueInMinorUnits(Order $order): int
    {
        $basisPoints = match ($order->customerTier) {
            'member' => 1_000,
            'vip' => 2_000,
            default => 0,
        };

        $subtotal = $order->subtotalInMinorUnits();
        $discount = intdiv(($subtotal * $basisPoints) + 5_000, 10_000);

        return $subtotal - $discount;
    }
}

final class CompleteCheckout
{
    public function __construct(private OrderPricing $pricing) {}

    public function handle(Order $order): void
    {
        $amount = $this->pricing->amountDueInMinorUnits($order);
        // Payment, lưu đơn và receipt vẫn được điều phối ở đây.
    }
}
~~~

**Đổi:** tính giá có tên và seam kiểm thử riêng. **Chưa đổi:** checkout vẫn gọi detail cụ thể. Đây là bước giữa có chủ đích; OCP sẽ xem riêng cách chọn discount.

## Test hành vi

**Loại snippet: mẫu PHPUnit, phụ thuộc Order fixture; chưa chạy trong repo Markdown.**

~~~php
public function test_member_discount_is_applied(): void
{
    $order = OrderMother::withTierAndSubtotal('member', 10_000);

    self::assertSame(
        9_000,
        (new OrderPricing())->amountDueInMinorUnits($order),
    );
}
~~~

Test khóa kết quả nghiệp vụ thay vì xác nhận class/method được gọi. Phép nhân subtotal với basis points có thể overflow ở giá trị rất lớn; production cần miền giá trị, rounding, thuế và currency được xác định.

## Thay đổi, chi phí và khi không nên tách

| | Trước | Sau |
|---|---|---|
| Đổi discount | Sửa giữa processor | Sửa OrderPricing |
| Chi phí | Ít indirection | Dependency và file cần lần theo |
| Chưa giải quyết | — | Payment, SQL, receipt vẫn trực tiếp |

Nếu giá đơn giản, ổn định và luôn đổi cùng luồng checkout, tách class có thể chỉ thêm nơi đọc.

## Chẩn đoán trước khi refactor

1. Nêu hai thay đổi thực tế, không dùng “có thể sau này”.
2. Xác định nhóm sở hữu mỗi quy tắc.
3. Xem lịch sử PR: phần nào thường sửa cùng vì cấu trúc hiện tại?
4. Tách đúng logic có thể thay đổi độc lập.
5. Kiểm tra đổi giá còn chạm payment/persistence không.

## Nhận diện trong code review

Tín hiệu: PR đổi discount chạm SQL/SDK; test phép tính cần dựng database; hoặc nhánh theo actor/provider nằm cạnh orchestration.

Hỏi: “Nếu chỉ đổi quy tắc giá, file nào cần đổi? Vì sao phần lưu đơn phải tham gia?”

## Liên tưởng Laravel

Controller xử lý HTTP; Form Request xử lý input/authorization. Nếu giá còn được dùng từ queue/CLI, đừng để nó chỉ nằm trong Form Request. Không cần PricingService riêng nếu phép tính ổn định và không có lý do thay đổi độc lập.

## Tips / Lưu ý

- Đếm nhóm yêu cầu thay đổi và lịch sử PR; đừng đếm số method.
- Tách phần có lý do thay đổi độc lập; giữ orchestration dễ theo dõi.
- Một function/private method đôi khi đủ, chưa cần interface.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Module | Một đơn vị code được xem xét như một ranh giới thay đổi |
| Actor | Người/nhóm mà module cần đáp ứng |
| Cohesion | Mức các phần cùng phục vụ một mục đích |

## Tự kiểm tra

1. CompleteCheckout còn trách nhiệm nào chưa tách?
2. Khi nào OrderPricing làm tăng indirection mà không giảm thay đổi lan truyền?
3. Ai sở hữu quy tắc discount?

## Nguồn

- Robert C. Martin, [The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html), 2014-05-08; xem 2026-10-23.
- Robert C. Martin, *Agile Software Development: Principles, Patterns, and Practices*, 2003, chương SRP.

---

[← Trước: Bản đồ SOLID](00-overview.md) · [Tiếp: OCP →](02-ocp.md)
