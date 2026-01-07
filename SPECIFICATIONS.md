# 🧪 Technical Test - Backend PHP Developer

## Mini Cart Validation Service

**Estimated duration**: 2-3 hours  
**Stack**: PHP 8.3+, Slim 4, MySQL  
**Architecture**: Hexagonal (Ports & Adapters)

---

## 📋 Context

You are building a multi-tenant e-commerce platform that handles checkouts for multiple clients. Each client ("Company") can have its own taxation rules and promo codes.

You need to implement a **cart validation service** that:
1. Calculates totals (subtotal, taxes, total)
2. Applies discount codes
3. Handles both taxation models (taxes included / taxes added)

---

## 🎯 Objectives

### Required Features

1. **Cart calculation**
   - Subtotal (sum of `unit_price × quantity`)
   - Discount application (percentage or fixed amount)
   - Tax calculation based on country
   - Final total

2. **Discount management**
   - `SAVE10` → 10% off
   - `FLAT500` → €5 off (500 cents)
   - `WELCOME20` → 20% off (max 1000 cents)

3. **Tax management**
   - France (FR): 20% VAT
   - Germany (DE): 19% VAT
   - United States (US): 0% (taxes added at checkout by state)
   - Canada (CA): 5% GST

4. **Two taxation modes**
   - `taxes_included: true` → Displayed prices already include VAT
   - `taxes_included: false` → VAT is added to subtotal

---

## 📥 Expected Input

```php
$request = [
    'items' => [
        [
            'sku' => 'PROD-001',
            'name' => 'Premium Widget',
            'quantity' => 2,
            'unit_price' => 2999  // €29.99 in cents
        ],
        [
            'sku' => 'PROD-002', 
            'name' => 'Basic Gadget',
            'quantity' => 1,
            'unit_price' => 4999  // €49.99 in cents
        ]
    ],
    'discount_code' => 'SAVE10',      // Optional
    'country_code' => 'FR',
    'taxes_included' => true
];
```

---

## 📤 Expected Output

```php
[
    'success' => true,
    'cart' => [
        'items' => [
            [
                'sku' => 'PROD-001',
                'name' => 'Premium Widget',
                'quantity' => 2,
                'unit_price' => 2999,
                'line_total' => 5998
            ],
            [
                'sku' => 'PROD-002',
                'name' => 'Basic Gadget', 
                'quantity' => 1,
                'unit_price' => 4999,
                'line_total' => 4999
            ]
        ],
        'subtotal' => 10997,           // Sum of line_totals
        'discount' => [
            'code' => 'SAVE10',
            'type' => 'percentage',
            'value' => 10,
            'amount' => 1100           // 10% of 10997, rounded
        ],
        'subtotal_after_discount' => 9897,
        'tax' => [
            'rate' => 20.0,
            'amount' => 1649,          // If taxes_included, extracted from total
            'included' => true
        ],
        'total' => 9897                // With taxes included = subtotal_after_discount
    ],
    'currency' => 'EUR'
]
```

---

## 🏗️ Expected Architecture

```
src/
├── Domain/
│   ├── Entity/
│   │   └── CartItem.php
│   ├── ValueObject/
│   │   ├── Money.php
│   │   ├── Percentage.php
│   │   ├── Sku.php
│   │   └── CountryCode.php
│   ├── Service/
│   │   └── CartCalculator.php
│   └── Exception/
│       ├── InvalidDiscountCodeException.php
│       └── InvalidCartException.php
│
├── Application/
│   ├── DTO/
│   │   ├── CartRequest.php
│   │   └── CartResult.php
│   ├── Service/
│   │   ├── DiscountService.php
│   │   └── TaxService.php
│   └── UseCase/
│       └── ValidateCartHandler.php
│
├── Infrastructure/
│   └── Repository/
│       └── DiscountRepository.php    // Can be a simple array for this test
│
└── Presentation/
    └── Controller/
        └── CartController.php
```

---

## 📐 Technical Specifications

### Value Objects

Monetary amounts MUST be handled via a `Money` Value Object:

```php
final readonly class Money
{
    public function __construct(
        private int $cents,
        private string $currency = 'EUR'
    ) {
        if ($cents < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }
    }
    
    public function getCents(): int;
    public function add(Money $other): Money;
    public function subtract(Money $other): Money;
    public function multiply(int $quantity): Money;
    public function percentage(Percentage $percent): Money;
    // etc.
}
```

### Calculation Rules

1. **All amounts are in cents** (avoids floating point errors)
2. **Rounding**: always round up for taxes (`ceil`)
3. **Max discount**: a discount can never make the total negative
4. **Capped discount**: `WELCOME20` has a maximum of 1000 cents reduction

### Tax Calculation (taxes included)

When `taxes_included = true`, tax is **extracted** from the price:

```
Price incl. VAT = 10000 cents
VAT rate = 20%
Price excl. VAT = 10000 / 1.20 = 8333.33 → 8333 cents
VAT = 10000 - 8333 = 1667 cents
```

### Tax Calculation (taxes added)

When `taxes_included = false`, tax is **added** to the price:

```
Price excl. VAT = 10000 cents
VAT rate = 20%
VAT = 10000 × 0.20 = 2000 cents
Price incl. VAT = 10000 + 2000 = 12000 cents
```

---

## ✅ Test Cases to Cover

### Test 1: Simple cart without discount (FR, taxes included)
```php
Input: 2× 2999 + 1× 4999, country=FR, taxes_included=true, no discount
Expected: subtotal=10997, tax=1833, total=10997
```

### Test 2: Percentage discount
```php
Input: 1× 10000, country=FR, taxes_included=true, discount=SAVE10
Expected: subtotal=10000, discount=1000, total=9000
```

### Test 3: Fixed amount discount
```php
Input: 1× 10000, country=FR, taxes_included=true, discount=FLAT500
Expected: subtotal=10000, discount=500, total=9500
```

### Test 4: Capped discount
```php
Input: 1× 100000, country=FR, taxes_included=true, discount=WELCOME20
Expected: discount capped at 1000 (not 20000)
```

### Test 5: Taxes added (US → DE)
```php
Input: 1× 10000, country=DE, taxes_included=false
Expected: subtotal=10000, tax=1900, total=11900
```

### Test 6: Invalid discount code
```php
Input: discount=INVALID123
Expected: InvalidDiscountCodeException
```

### Test 7: Empty cart
```php
Input: items=[]
Expected: InvalidCartException
```

### Test 8: Invalid quantity
```php
Input: quantity=0 or quantity=-1
Expected: InvalidCartException
```

---

## 🚀 API Endpoints

### POST /api/cart/validate

**Request:**
```json
{
    "items": [
        {"sku": "PROD-001", "name": "Widget", "quantity": 2, "unit_price": 2999}
    ],
    "discount_code": "SAVE10",
    "country_code": "FR",
    "taxes_included": true
}
```

**Response 200:**
```json
{
    "success": true,
    "cart": { ... }
}
```

**Response 400 (validation error):**
```json
{
    "success": false,
    "error": {
        "code": "INVALID_DISCOUNT_CODE",
        "message": "The discount code 'INVALID' is not valid"
    }
}
```

---

## 📦 Provided Files

You will receive a starter project with:
- `composer.json` configured (Slim 4, PHP-DI, PHPUnit)
- `config/dependencies.php` with basic DI container
- `public/index.php` with Slim bootstrap
- Empty folder structure
- This specifications document

---

## 📊 Evaluation Criteria

| Criteria | Points | Description |
|----------|--------|-------------|
| **Architecture** | /25 | Hexagonal architecture compliance, separation of concerns |
| **Value Objects** | /20 | Proper use of Money, Percentage, immutability |
| **Calculations** | /20 | Accuracy of calculations, rounding handling, edge cases |
| **Error Handling** | /15 | Domain exceptions, input validation, clear messages |
| **Tests** | /15 | Test case coverage, clean unit tests |
| **Code Quality** | /5 | Readability, strict typing, PSR-12 |

**Total: /100**

---

## ⚠️ Constraints

1. **No ORM** - Use arrays or simple repositories
2. **Strict typing** - `declare(strict_types=1)` required
3. **Readonly** - Prefer `readonly` classes for Value Objects
4. **No floats** - All amounts in cents (integers)
5. **PHPUnit tests** - Minimum 5 unit tests

---

## 💡 Tips

- Start with Value Objects (`Money`, `Percentage`)
- Implement `CartCalculator` in Domain (pure logic, no dependencies)
- Application services (`DiscountService`, `TaxService`) can use static arrays for data
- Controller should only orchestrate (no business logic)
- Think about edge cases: discount > subtotal, quantity 0, etc.

---

## 📬 Deliverable

Send a `.zip` file containing:
- Your completed project (backend + frontend)
- All tests passing (`composer test`)
- Both servers run without errors

**Good luck! 🚀**
