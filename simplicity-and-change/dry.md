# DRY — Don’t Repeat Yourself

## Cách phát biểu nền tảng

David Thomas và Andrew Hunt phát biểu DRY trong *The Pragmatic Programmer*: mỗi phần kiến thức cần có một representation duy nhất, rõ ràng và có thẩm quyền trong hệ thống. Họ cũng lưu ý DRY rộng hơn việc tránh lặp code; nó liên quan đến việc biểu đạt kiến thức trong hệ thống ([trích đoạn mục DRY do nhà xuất bản cung cấp](https://media.pragprog.com/titles/tpp20/dry.pdf), ấn bản kỷ niệm 20 năm, 2019; kiểm tra 2026-10-23).

## Hiểu đơn giản

**Đừng để cùng một quy tắc nghiệp vụ có nhiều bản độc lập phải sửa đồng bộ.**

## Điều DRY nhắm đến

Đoạn code giống nhau là tín hiệu để điều tra, không phải bằng chứng đủ để hợp nhất. Hai đoạn khác cú pháp vẫn có thể lặp một quy tắc; hai đoạn giống hệt có thể thuộc hai quy tắc khác nhau.

Một nơi biểu đạt có thẩm quyền không nhất thiết là một file duy nhất: có thể dùng code generation hoặc schema chung. Điều quan trọng là thay đổi của một kiến thức không buộc con người phải nhớ đồng bộ nhiều bản nguồn độc lập.

## BAD — cùng một giới hạn discount được mã hóa hai lần

**Loại snippet: minh họa, không chạy độc lập.**

~~~php
final class OrderPricing
{
    public function amountDue(int $subtotal, int $discountBasisPoints): int
    {
        $discountBasisPoints = min($discountBasisPoints, 2_000);

        return $subtotal - intdiv($subtotal * $discountBasisPoints, 10_000);
    }
}

final class CheckoutValidator
{
    public function validateDiscount(int $discountBasisPoints): void
    {
        if ($discountBasisPoints > 2_000) {
            throw new InvalidArgumentException('Discount is too large.');
        }
    }
}
~~~

**Hành vi:** một nơi âm thầm giới hạn discount; nơi kia từ chối giá trị vượt giới hạn. Nếu quy tắc “tối đa 20%” đổi mà chỉ một nơi được cập nhật, validation và tính giá bất đồng.

## Refactor / GOOD — một policy sở hữu giới hạn

**Loại snippet: minh họa ranh giới, không chạy độc lập.**

~~~php
final class DiscountRate
{
    private function __construct(private int $basisPoints) {}

    public static function fromBasisPoints(int $basisPoints): self
    {
        if ($basisPoints < 0 || $basisPoints > 2_000) {
            throw new InvalidArgumentException('Discount must be between 0% and 20%.');
        }

        return new self($basisPoints);
    }

    public function basisPoints(): int
    {
        return $this->basisPoints;
    }
}

final class OrderPricing
{
    public function amountDue(int $subtotal, DiscountRate $rate): int
    {
        $discount = intdiv(($subtotal * $rate->basisPoints()) + 5_000, 10_000);

        return $subtotal - $discount;
    }
}
~~~

**Đổi:** mọi caller phải có một DiscountRate hợp lệ trước khi tính. **Chi phí:** tạo value object, constructor private, validation và đường chuyển đổi input. Nếu giới hạn chỉ được dùng một lần hoặc validation policy khác theo context, abstraction này có thể không đáng giá.

## Test hành vi

**Loại snippet: test mẫu; PHPUnit syntax, chưa chạy trong repo này.**

~~~php
public function test_discount_rate_rejects_values_above_the_business_limit(): void
{
    $this->expectException(InvalidArgumentException::class);

    DiscountRate::fromBasisPoints(2_001);
}

public function test_pricing_uses_the_validated_rate(): void
{
    $rate = DiscountRate::fromBasisPoints(1_000);

    self::assertSame(9_000, (new OrderPricing())->amountDue(10_000, $rate));
}
~~~

Test bảo vệ giới hạn và kết quả tính, nhưng không khẳng định mọi trường hợp trùng code cần gộp.

## Cái gì không nên DRY một cách máy móc

Giả sử phí giao hàng và phí xử lý hiện cùng là 500 minor units. Nếu chúng có chủ sở hữu, thay đổi và ngữ nghĩa khác nhau, một hằng số tên “FEE” dùng chung có thể tạo coupling sai. Cùng giá trị hiện tại không có nghĩa cùng kiến thức.

Ngược lại, nếu cùng quy tắc thuế được chép ở API, báo giá và invoice, hãy quyết định đâu là nguồn có thẩm quyền và cách các thành phần còn lại dùng hoặc sinh ra representation đó.

## Chẩn đoán và code review

- Đây là lặp cú pháp hay lặp một quyết định nghiệp vụ?
- Hai bản có cần thay đổi cùng lúc theo một yêu cầu không?
- Chúng có cùng owner và cùng ngữ nghĩa không?
- Nếu gộp, abstraction có tên theo khái niệm nghiệp vụ hay chỉ là “Utils”?
- Có test nào thể hiện nguy cơ lệch quy tắc?
- Chi phí đồng bộ nhiều bản có lớn hơn chi phí abstraction không?

## Liên hệ với SOLID và GRASP

DRY và SRP có thể cùng gợi ý đặt một quy tắc ở nơi có cohesion cao. Nhưng DRY không buộc một class chịu toàn bộ trách nhiệm; SRP cũng không yêu cầu mọi logic giống nhau phải gom vào một module. GRASP Information Expert giúp cân nhắc nơi có dữ liệu để áp dụng quy tắc; quyết định cuối còn phụ thuộc coupling và ownership.

## Tips / Lưu ý

- Khi thấy copy/paste, tìm nguồn kiến thức chung trước khi tạo helper.
- Đừng tạo abstraction chỉ dựa trên “hai đoạn đang giống nhau”.
- Trùng code là mùi để hỏi; trùng quy tắc mới là lý do DRY đáng quan tâm.
- Cân nhắc duplication có chủ ý khi các khái niệm dự kiến thay đổi độc lập.

## Sơ đồ nhận diện duplication

~~~mermaid
flowchart TD
    A["Hai đoạn trông giống nhau"] --> B{"Cùng biểu đạt một quy tắc?"}
    B -->|"Có"| C["Tìm representation có thẩm quyền"]
    B -->|"Không"| D["Giữ riêng, tránh coupling sai"]
~~~

Sơ đồ là câu hỏi điều tra, không phải thuật toán tự động hợp nhất code.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Knowledge duplication | Một quy tắc/kiến thức có nhiều bản độc lập |
| Representation | Nơi hoặc cách quy tắc được biểu đạt trong hệ thống |
| Accidental duplication | Code giống nhau nhưng thuộc các khái niệm khác nhau |
| Authoritative representation | Nguồn có thẩm quyền để thay đổi một quy tắc |

## Câu hỏi tự kiểm tra

1. Vì sao hai đoạn code giống hệt có thể không vi phạm DRY?
2. Một quy tắc thuế xuất hiện ở ba ứng dụng thì “một representation” có nhất thiết là một file code?
3. Điều gì sẽ khó thay đổi hơn nếu hợp nhất hai khái niệm chỉ tình cờ dùng cùng con số?

## Nguồn

- David Thomas, Andrew Hunt, *The Pragmatic Programmer, 20th Anniversary Edition*, mục “The Evils of Duplication”; [trích đoạn do nhà xuất bản cung cấp](https://media.pragprog.com/titles/tpp20/dry.pdf), 2019.
- [Trang sách của nhà xuất bản Pragmatic Bookshelf](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/).
- Ngày kiểm tra: 2026-10-23.

---

[← Trước: GRASP](../object-design/grasp.md) · [Tiếp: YAGNI →](yagni.md)
