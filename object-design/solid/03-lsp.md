# 04 · L — Liskov Substitution Principle

## Cách phát biểu nền tảng và bản dịch sát

**Liskov và Wing mô tả quan hệ subtype bằng hành vi:** “the objects of the subtype ought to behave the same as those of the supertype as far as any program using supertype objects can tell.” ([Liskov & Wing, 1994](https://www.cs.columbia.edu/~wing/publications/LiskovWing94.pdf)).

**Dịch sát:** Các đối tượng của kiểu con nên hành xử giống các đối tượng của kiểu cha, xét theo những gì bất kỳ chương trình nào dùng đối tượng kiểu cha có thể nhận biết.

## Hiểu đơn giản

**Thay một món đồ bằng món tương đương mà người dùng vẫn làm được việc như trước, không bị bất ngờ bởi điều kiện hay kết quả khác.**

## Áp dụng vào case checkout

**Áp dụng vào case checkout:** thay Stripe bằng FastPay mà checkout vẫn gửi input hợp lệ và nhận kết quả/lỗi theo cùng contract.

## Mục tiêu

Mô tả contract qua điều kiện trước/sau, bất biến, lỗi; phân biệt lỗi type với sai hành vi; dùng chung contract test cho nhiều adapter.

## Trạng thái checkout

Sau [OCP](02-ocp.md), checkout đã có policy giá. Bài này thay provider ở ranh giới payment: Stripe và FastPay phải giữ contract mà CompleteCheckout dùng.

Liskov và Wing định nghĩa subtype theo hành vi quan sát của chương trình ([A Behavioral Notion of Subtyping](https://dl.acm.org/doi/10.1145/197320.197383), 1994; xem 2026-10-23).

## Cốt lõi: contract trước implementation

Giả định contract của ví dụ quy định: caller chỉ truyền Money dương thuộc tập currency mà checkout công bố chung cho mọi PaymentGateway implementation; khi thành công, `charge` trả PaymentReference không rỗng; khi provider từ chối, adapter ném PaymentDeclined; lỗi hạ tầng được ánh xạ sang lỗi ứng dụng đã định nghĩa; thao tác charge không tự đổi trạng thái Order. `Money` giữ amount theo integer minor units và currency, còn currency chịu trách nhiệm quy ước exponent. Đây là contract giả định cho bài học, không phải bảo đảm của mọi SDK.

Điều kiện trước là caller phải bảo đảm trước khi gọi. Điều kiện sau là operation hứa khi hoàn tất. Bất biến phải luôn đúng trong vòng đời object. Contract còn gồm ngoại lệ và tác động phụ quan sát được. Quy tắc thay thế thường được diễn đạt: subtype không siết điều kiện trước, không làm yếu điều kiện sau, và phải giữ bất biến của kiểu cha. Đây là cách kiểm tra hành vi, không chỉ so chữ ký PHP.

| Phần contract | Kỳ vọng ở PaymentGateway | Dấu hiệu vi phạm khi thay implementation |
|---|---|---|
| Điều kiện trước | Mọi Money dương thuộc currency đã công bố đều được nhận | Provider chỉ nhận USD dù contract hứa nhiều currency |
| Điều kiện sau | Thành công trả reference hợp lệ | Trả reference rỗng hoặc trạng thái chưa rõ như thể thành công |
| Bất biến | Charge không tự đánh dấu Order đã lưu là paid | Adapter âm thầm sửa Order hoặc persistence |
| Lỗi | Decline thành PaymentDeclined; lỗi hạ tầng theo contract | Ném exception vendor khiến caller phải biết SDK |
| Tác động phụ | Có thể thu tiền theo ngữ nghĩa đã công bố; không làm thêm việc ngoài contract | Retry tạo charge kép hoặc gửi email |

## Sơ đồ

~~~mermaid
flowchart TD
    A["PaymentGateway contract"] --> B["Stripe adapter"]
    A --> C["FastPay adapter"]
    B --> D["Cùng contract tests"]
    C --> D
~~~

## BAD — chữ ký giống, điều kiện và lỗi khác

**Loại snippet: minh họa, không chạy độc lập.**

~~~php
interface PaymentGateway
{
    public function charge(Money $amount): PaymentReference;
}

final class FastPayGateway implements PaymentGateway
{
    public function charge(Money $amount): PaymentReference
    {
        if ($amount->currency !== 'USD') {
            throw new RuntimeException('Unsupported currency');
        }

        // Gọi provider...
    }
}
~~~

Giả sử contract cho phép EUR. FastPay thu hẹp input hợp lệ xuống USD và ném lỗi khác với PaymentDeclined. Caller có thể xử lý sai dù chữ ký tương thích.

## Refactor / GOOD — adapter giữ cùng hành vi quan sát được

**Loại snippet: minh họa, phụ thuộc SDK/domain types.**

~~~php
final class FastPayGateway implements PaymentGateway
{
    public function charge(Money $amount): PaymentReference
    {
        try {
            $response = $this->client->charge(
                amountInMinorUnits: $amount->minorUnits,
                currency: $amount->currency,
            );
        } catch (FastPayDeclined $exception) {
            throw new PaymentDeclined(previous: $exception);
        }

        return new PaymentReference($response->id);
    }
}
~~~

Adapter dịch request/response/lỗi sang ngôn ngữ ứng dụng. Nếu FastPay không hỗ trợ EUR, không giả vờ nó thực hiện contract rộng: tách capability hoặc chọn provider ở nơi biết yêu cầu currency.

## Test hành vi dùng chung

**Loại snippet: mẫu PHPUnit; cần gateway được cấu hình cho sandbox/mocks. Chưa chạy trong repo.**

~~~php
public function test_success_returns_non_empty_reference(): void
{
    $gateway = $this->gatewayConfiguredForSuccess();

    $reference = $gateway->charge(Money::usd(1_000));

    self::assertNotSame('', $reference->value);
}

public function test_decline_uses_domain_exception(): void
{
    $gateway = $this->gatewayConfiguredForDecline();

    $this->expectException(PaymentDeclined::class);
    $gateway->charge(Money::usd(1_000));
}
~~~

Chạy cùng bộ test cho mọi implementation. Không gửi giao dịch thật; cấu hình sandbox/mock transport. Thêm currency, invalid amount, timeout/idempotency nếu chúng thuộc contract.

## Thay đổi, chi phí và liên hệ kiểu PHP

PHP kiểm tra signature và variance; covariance/contravariance được hỗ trợ từ PHP 7.4. Type checker không chứng minh reference không rỗng, điều kiện tiền tệ được giữ, hoặc lỗi đã được map đúng ([PHP Manual: Variance](https://www.php.net/manual/en/language.oop5.variance.php), xem 2026-10-23).

Chi phí LSP là contract rõ hơn và bộ test chung. Nếu không có nhiều implementation, vẫn có thể viết contract cho một adapter để tránh mơ hồ; nhưng đừng tạo hierarchy chỉ vì nghĩ sẽ có provider khác.

## Ví dụ phụ: Square/Rectangle

Nếu Rectangle cho phép đổi width/height độc lập, Square kế thừa nhưng buộc hai cạnh bằng nhau có thể phá kỳ vọng của caller. Đây không phải lệnh cấm inheritance: nếu caller chỉ cần area(), hai hình có thể cùng thực hiện contract nhỏ đó.

## Chẩn đoán trước khi refactor

1. Viết contract quan sát được: input hợp lệ, output, lỗi, bất biến và tác động phụ.
2. Kiểm tra subtype không siết điều kiện trước, không làm yếu điều kiện sau, và giữ invariants.
3. Dùng cùng contract test cho mọi implementation; bổ sung test lỗi và side effect quan trọng.
4. Xác định khác biệt provider có thể được adapter ánh xạ mà không đổi nghĩa không.
5. Nếu không giữ được contract, tách capability hoặc giới hạn nơi chọn provider.

## Nhận diện trong code review

Tín hiệu: method “not supported”; currency/provider phụ thuộc hẹp hơn; exception khác làm caller vỡ; hoặc thành công không trả dữ liệu cần thiết.

Hỏi: “Một test contract viết từ góc nhìn caller có thể chạy nguyên vẹn trên mọi adapter không? Có subtype nào từ chối input hợp lệ, hứa ít hơn hoặc tạo side effect khác không?”

## Liên tưởng Laravel

Container resolve implementation theo binding, không xác minh hành vi. Dùng chung contract tests cho Stripe/FastPay; binding đúng type chỉ chứng minh tìm thấy object.

## Tips / Lưu ý

- Viết rõ điều kiện đầu vào, kết quả thành công, lỗi và side effect trước khi đánh giá implementation.
- Chạy cùng contract tests trên mọi provider.
- Nếu provider thiếu capability, đừng giả vờ nó hỗ trợ; tách contract hoặc giới hạn nơi chọn.

## Glossary riêng của bài

| Thuật ngữ | Nghĩa trong bài |
|---|---|
| Precondition | Điều caller phải bảo đảm trước khi gọi |
| Postcondition | Điều operation hứa sau khi thành công |
| Invariant | Điều kiện phải luôn đúng trong vòng đời object |

## Tự kiểm tra

1. FastPay đã siết điều kiện trước nào?
2. Vì sao đổi exception phá caller dù signature giữ nguyên?
3. Khi nào tách capability thay vì ép provider vào contract chung?

## Nguồn

- Barbara Liskov và Jeannette Wing, [A Behavioral Notion of Subtyping](https://dl.acm.org/doi/10.1145/197320.197383), 1994.
- PHP Manual, [Covariance and Contravariance](https://www.php.net/manual/en/language.oop5.variance.php), hỗ trợ từ PHP 7.4; xem 2026-10-23.

---

[← Trước: OCP](02-ocp.md) · [Tiếp: ISP →](04-isp.md)
