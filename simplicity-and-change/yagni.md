# YAGNI — You Aren’t Gonna Need It

## Cách phát biểu nền tảng

YAGNI là lời khuyên trong Extreme Programming: không thêm khả năng chỉ vì có thể cần trong tương lai; Fowler mô tả nó như một phần của thiết kế đơn giản và phát triển tăng dần ([Martin Fowler, Yagni](https://martinfowler.com/bliki/Yagni.html), kiểm tra 2026-10-23).

## Hiểu đơn giản

**Đừng trả chi phí xây và duy trì một khả năng khi chưa có nhu cầu đủ căn cứ.**

## YAGNI không có nghĩa là gì

- Không có nghĩa bỏ qua yêu cầu đã xác nhận.
- Không có nghĩa không thiết kế cho độ tin cậy, bảo mật hoặc giới hạn pháp lý.
- Không có nghĩa chọn code khó đổi; code có thể giữ cấu trúc rõ mà chưa cần extension framework.
- Không có nghĩa phải chờ sự cố rồi mới xử lý mọi rủi ro.

YAGNI nói về phần mềm được xây cho nhu cầu suy đoán. Quyết định nên cân nhắc xác suất thay đổi, chi phí đổi sau này, tác động nếu sai và chi phí duy trì abstraction ngay từ bây giờ.

## BAD — hệ thống discount plugin trước nhu cầu

**Loại snippet: minh họa kiến trúc giả định, không chạy độc lập.**

~~~php
interface DiscountPlugin {}

interface DiscountPluginFactory
{
    public function make(string $name): DiscountPlugin;
}

interface DiscountPluginLoader
{
    public function loadFromRemoteRegistry(): iterable;
}

final class DiscountPluginManager
{
    public function __construct(
        private DiscountPluginLoader $loader,
        private DiscountPluginFactory $factory,
    ) {}
}
~~~

Ứng dụng hiện chỉ có hai hạng khách hàng, không có plugin bên thứ ba hay yêu cầu cấu hình động. Hành vi hiện tại chưa cần plugin, nhưng hệ thống đã có loader, factory, registry và hợp đồng extension. Người đọc phải hiểu lifecycle và wiring trước khi thêm một discount đơn giản.

## GOOD — biểu diễn nhu cầu đang có, giữ đường nâng cấp

**Loại snippet: minh họa; cách chọn policy được thể hiện rõ.**

~~~php
final class OrderPricing
{
    public function amountDueInMinorUnits(Order $order): int
    {
        $basisPoints = match ($order->customerTier()) {
            CustomerTier::Member => 1_000,
            CustomerTier::Vip => 2_000,
            default => 0,
        };

        $subtotal = $order->subtotalInMinorUnits();
        $discount = intdiv(($subtotal * $basisPoints) + 5_000, 10_000);

        return $subtotal - $discount;
    }
}
~~~

**Đổi:** nhu cầu hiện tại được biểu diễn trực tiếp, ít khái niệm và không cần wiring plugin. **Chi phí/rủi ro:** thêm tier phải sửa match và test; nếu tier tăng nhanh, được cấu hình bởi nhiều tenant hoặc đổi độc lập thường xuyên, điểm mở rộng có thể trở thành giá trị thực. Khi đó refactor dựa trên bằng chứng là bước tiếp theo, không phải thất bại của thiết kế.

## Kiểm tra test và khả năng tiến hóa

**Loại snippet: test mẫu; PHPUnit syntax, chưa chạy trong repo này.**

~~~php
public function test_vip_customer_gets_the_documented_discount(): void
{
    $order = OrderMother::withTierAndSubtotal(CustomerTier::Vip, 10_000);

    self::assertSame(
        8_000,
        (new OrderPricing())->amountDueInMinorUnits($order),
    );
}
~~~

Test ghi lại quy tắc đang hỗ trợ. Nó không ngăn thiết kế mở rộng; khi yêu cầu variation xuất hiện, test sẽ làm thay đổi có căn cứ và bảo vệ hành vi hiện tại.

## Chẩn đoán trước khi thêm abstraction

1. Ghi nhu cầu đã xác nhận và ai xác nhận nó.
2. Tìm thay đổi tương tự trong lịch sử hoặc roadmap đã cam kết.
3. Ước lượng chi phí hiện tại nếu chưa abstraction và chi phí duy trì abstraction.
4. Tìm lựa chọn đơn giản có thể refactor an toàn khi thông tin mới đến.
5. Nêu tín hiệu sẽ khiến team tạo extension point sau này.

## Nhận diện trong code review

Hỏi:

- Ai đang cần extension point này hôm nay?
- Có bao nhiêu implementation thực tế, không tính giả định?
- Chúng khác nhau ở contract hay chỉ ở cấu hình?
- Có thể diễn đạt nhu cầu hiện tại bằng một hàm, enum hoặc match rõ ràng không?
- Chi phí đổi sau này thực sự lớn đến mức nào?
- Nếu không xây ngay, điều gì có thể bị mất hoặc trở nên nguy hiểm?

## Liên hệ OCP và SOLID

OCP bảo vệ phần ổn định khỏi biến động đã nhận diện. YAGNI ngăn việc thêm điểm mở rộng cho biến động chưa có cơ sở. Hai hướng này không mâu thuẫn: OCP cần một điểm biến động có lý do; YAGNI nhắc tính cả chi phí của abstraction.

## Tips / Lưu ý

- “Có thể cần” là giả thuyết, không phải requirement.
- Nhưng một rủi ro bảo mật/pháp lý/tính toàn vẹn dữ liệu đã biết không phải suy đoán tùy tiện.
- Ưu tiên cấu trúc dễ hiểu và test được; hoãn extension framework, không hoãn chất lượng cơ bản.
- Ghi trigger cho refactor tương lai, ví dụ số policy tăng hoặc biến thể được cấu hình bởi tenant.

## Sơ đồ quyết định

~~~mermaid
flowchart TD
    A["Có nhu cầu đã xác nhận?"] -->|"Chưa"| B["Chưa xây extension point"]
    A -->|"Có"| C["Ước lượng chi phí và rủi ro"]
    C --> D["Chọn giải pháp nhỏ nhất đáp ứng nhu cầu"]
~~~

“Chưa xây extension point” không có nghĩa bỏ qua bảo mật, dữ liệu hoặc yêu cầu đã cam kết.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Speculative feature | Khả năng được xây dựa trên dự đoán chưa có căn cứ đủ mạnh |
| Extension point | Ranh giới cho phép thêm biến thể |
| Refactor trigger | Tín hiệu cụ thể để xem lại cấu trúc khi nhu cầu thay đổi |

## Câu hỏi tự kiểm tra

1. Vì sao thêm plugin framework có thể làm tổng chi phí cao dù chưa có bug?
2. Trường hợp nào “chưa có user yêu cầu” vẫn không đủ để trì hoãn thiết kế?
3. Tín hiệu cụ thể nào khiến bạn thay match bằng registry/policy?

## Nguồn

- Martin Fowler, [Yagni](https://martinfowler.com/bliki/Yagni.html), cập nhật/kiểm tra 2026-10-23.
- Kent Beck, *Extreme Programming Explained: Embrace Change*, ấn bản 2, 2004; phần về simple design và phát triển tăng dần.
- Ngày kiểm tra liên kết: 2026-10-23.

---

[← Trước: DRY](dry.md) · [Tiếp: Các nguyên tắc tương tác →](../foundations/interactions.md)
