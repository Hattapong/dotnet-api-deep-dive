# Part 9 — Validation

> **Deep Dive Series: .NET Web API Internals**  
> Validation คือด่านแรกที่ป้องกัน bad data เข้าสู่ระบบ  
> บทนี้จะดูทุกชั้นของ validation ตั้งแต่ attribute จนถึง custom business rules

---

## 1. ทำไม Validation ถึงสำคัญ?

```csharp
// ❌ ไม่มี validation — ปัญหาทุกอย่างตามมา
[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    var user = new User(dto.Name, dto.Email, dto.Age);
    _repo.Save(user);  // name = null? email = "notanemail"? age = -5?
    return Ok(user);   // DB constraint error? null reference? หรือแย่กว่านั้น?
}
```

Validation ควรเกิด **ก่อน** ที่ข้อมูลจะไปถึง business logic เพื่อ:
- ป้องกัน invalid state เข้าสู่ระบบ
- ส่ง error กลับที่ชัดเจน ไม่ใช่ 500
- ลด defensive code ใน service layer

```
Request
   │
   ▼
[ Model Binding ]     → parse JSON → C# object
   │
   ▼
[ Validation ]        ← จุดที่บทนี้พูดถึง
   │
   ├── invalid → 400 ProblemDetails (ออกไปเลย)
   │
   ▼ valid
[ Controller Action ]
   │
   ▼
[ Service / Business Logic ]
```

---

## 2. Data Annotations — Validation แบบ Built-in

วิธีที่เร็วที่สุด — แปะ attribute บน DTO property

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2,
        ErrorMessage = "Name must be between 2 and 100 characters")]
    public string Name { get; set; } = "";

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    [StringLength(200)]
    public string Email { get; set; } = "";

    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; set; }

    [Phone]
    public string? PhoneNumber { get; set; }

    [Url]
    public string? Website { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{6}$",
        ErrorMessage = "Passport must be 2 letters followed by 6 digits")]
    public string? PassportNumber { get; set; }

    [Compare("Email", ErrorMessage = "Emails do not match")]
    public string? ConfirmEmail { get; set; }
}
```

### Annotations ที่ใช้บ่อย

| Attribute | ตรวจสอบ |
|-----------|---------|
| `[Required]` | ไม่เป็น null หรือ empty |
| `[StringLength(max, Min=n)]` | ความยาว string |
| `[Range(min, max)]` | ช่วงของตัวเลข / วันที่ |
| `[EmailAddress]` | รูปแบบ email |
| `[Phone]` | รูปแบบเบอร์โทร |
| `[Url]` | รูปแบบ URL |
| `[RegularExpression(pattern)]` | match regex |
| `[Compare("OtherProp")]` | เท่ากับ property อื่น |
| `[MinLength(n)]` / `[MaxLength(n)]` | ความยาว array / string |
| `[CreditCard]` | รูปแบบหมายเลขบัตร |
| `[EnumDataType(typeof(T))]` | ค่าอยู่ใน enum |

### ทำงานอัตโนมัติกับ [ApiController]

```csharp
// [ApiController] จัดการ validation ให้อัตโนมัติ
// ถ้า ModelState invalid → return 400 ProblemDetails ก่อนเข้า action

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateUserDto dto)
    {
        // ถึงตรงนี้ได้ = dto ผ่าน validation แล้ว
        return Ok();
    }
}
```

ทดสอบส่ง invalid data:

```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"A","email":"notanemail","age":5}'
```

Response 400:
```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Name": ["Name must be between 2 and 100 characters"],
    "Email": ["Invalid email format"],
    "Age": ["Age must be between 18 and 120"]
  }
}
```

---

## 3. Custom Validation Attribute

เมื่อ built-in annotations ไม่พอ สร้าง attribute เองได้:

```csharp
// ตรวจสอบว่าวันที่ต้องอยู่ในอนาคต
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value, ValidationContext context)
    {
        if (value is DateTime date && date <= DateTime.UtcNow)
        {
            return new ValidationResult(
                ErrorMessage ?? $"{context.DisplayName} must be a future date");
        }
        return ValidationResult.Success;
    }
}

// ตรวจสอบว่า list ต้องไม่ว่าง
public class NotEmptyListAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value, ValidationContext context)
    {
        if (value is ICollection { Count: 0 })
            return new ValidationResult(
                ErrorMessage ?? $"{context.DisplayName} must not be empty");

        return ValidationResult.Success;
    }
}
```

```csharp
// ใช้งาน
public class CreateEventDto
{
    [Required]
    public string Title { get; set; } = "";

    [FutureDate(ErrorMessage = "Event must be scheduled in the future")]
    public DateTime ScheduledAt { get; set; }

    [NotEmptyList(ErrorMessage = "At least one attendee is required")]
    public List<string> Attendees { get; set; } = new();
}
```

### IValidatableObject — Validate ข้าม Property

เมื่อต้องการ validation ที่ดูหลาย property พร้อมกัน:

```csharp
public class BookingDto : IValidatableObject
{
    [Required] public DateTime CheckIn  { get; set; }
    [Required] public DateTime CheckOut { get; set; }
    public int Guests { get; set; }
    public bool IsGroupBooking { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        // CheckOut ต้องหลัง CheckIn
        if (CheckOut <= CheckIn)
            yield return new ValidationResult(
                "Check-out must be after check-in",
                new[] { nameof(CheckOut) });

        // ต้องจองล่วงหน้าอย่างน้อย 1 วัน
        if (CheckIn < DateTime.UtcNow.AddDays(1))
            yield return new ValidationResult(
                "Booking must be at least 1 day in advance",
                new[] { nameof(CheckIn) });

        // Group booking ต้องมีแขกอย่างน้อย 10 คน
        if (IsGroupBooking && Guests < 10)
            yield return new ValidationResult(
                "Group booking requires at least 10 guests",
                new[] { nameof(Guests), nameof(IsGroupBooking) });
    }
}
```

---

## 4. FluentValidation — Validation ที่ยืดหยุ่นกว่า

Data Annotations มีข้อจำกัด: logic ซับซ้อนเขียนยาก, ทดสอบยาก, อ่านยากเมื่อมีหลาย rule

**FluentValidation** แก้ปัญหานี้ด้วย fluent API ที่อ่านง่ายและทดสอบได้:

```bash
dotnet add package FluentValidation.AspNetCore
```

```csharp
// CreateUserValidator.cs
using FluentValidation;

public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    private readonly IUserRepository _repo;

    public CreateUserValidator(IUserRepository repo)
    {
        _repo = repo;  // inject dependency ได้เลย — ทำกับ Attribute ไม่ได้!

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .Length(2, 100).WithMessage("Name must be between 2 and 100 characters")
            .Matches(@"^[a-zA-Z\s]+$").WithMessage("Name must contain only letters");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("Invalid email format")
            .MustAsync(BeUniqueEmail)
                .WithMessage("Email is already registered");
            // ↑ async rule — ไป query DB ได้เลย! Attribute ทำไม่ได้

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120)
            .WithMessage("Age must be between 18 and 120");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Password must contain uppercase")
            .Matches("[0-9]").WithMessage("Password must contain a number")
            .Matches("[^a-zA-Z0-9]").WithMessage("Password must contain special char");

        // conditional rule — validate เฉพาะเมื่อมีค่า
        When(x => x.PhoneNumber is not null, () =>
        {
            RuleFor(x => x.PhoneNumber!)
                .Matches(@"^\+?[1-9]\d{7,14}$")
                .WithMessage("Invalid phone number format");
        });
    }

    private async Task<bool> BeUniqueEmail(
        string email, CancellationToken ct)
    {
        return !await _repo.ExistsAsync(email, ct);
    }
}
```

```csharp
// Program.cs — register FluentValidation
using FluentValidation;
using FluentValidation.AspNetCore;

builder.Services.AddControllers();
builder.Services.AddFluentValidationAutoValidation();
// ↑ integrate กับ ModelState อัตโนมัติ — [ApiController] จะดัก error เหมือนเดิม

// register validators — สามารถ scan assembly ได้
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserValidator>();
```

### FluentValidation Rules ที่ใช้บ่อย

```csharp
RuleFor(x => x.Name)
    .NotNull()
    .NotEmpty()
    .Length(2, 100)
    .MinimumLength(2)
    .MaximumLength(100)
    .Matches(@"pattern")
    .EmailAddress()
    .Must(name => name != "admin")
        .WithMessage("Name cannot be 'admin'")
    .MustAsync(async (name, ct) => await IsUnique(name, ct));

RuleFor(x => x.Amount)
    .GreaterThan(0)
    .GreaterThanOrEqualTo(10)
    .LessThan(10_000)
    .ExclusiveBetween(0, 10_000)
    .InclusiveBetween(1, 9_999)
    .PrecisionScale(10, 2, false);  // decimal precision

RuleFor(x => x.Items)
    .NotEmpty()
    .Must(items => items.Count <= 100)
        .WithMessage("Maximum 100 items");

// nested object
RuleFor(x => x.Address).SetValidator(new AddressValidator());

// collection
RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());
```

---

## 5. Validation ใน Service Layer

บางครั้ง validation ที่ต้องใช้ business context ไม่เหมาะจะทำใน DTO — ควรทำใน service:

```csharp
public class OrderService
{
    public async Task<Order> CreateAsync(CreateOrderDto dto, CancellationToken ct)
    {
        // Business rule validation — ไม่ใช่ input format validation
        var user = await _userRepo.GetByIdAsync(dto.UserId, ct)
            ?? throw new NotFoundException($"User {dto.UserId} not found");

        if (!user.IsActive)
            throw new BusinessRuleException("Inactive users cannot place orders");

        var product = await _productRepo.GetByIdAsync(dto.ProductId, ct)
            ?? throw new NotFoundException($"Product {dto.ProductId} not found");

        if (product.Stock < dto.Quantity)
            throw new BusinessRuleException(
                $"Insufficient stock. Available: {product.Stock}");

        // ผ่านทุก rule แล้ว → create order
        var order = new Order(dto.UserId, dto.ProductId, dto.Quantity);
        await _orderRepo.SaveAsync(order, ct);
        return order;
    }
}
```

```csharp
// Custom exceptions สำหรับ business rules
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class BusinessRuleException : Exception
{
    public BusinessRuleException(string message) : base(message) { }
}
// Error handling middleware จะ catch เหล่านี้แล้ว map → HTTP status
// (จะเห็นใน Part 11)
```

---

## 6. ตัวอย่างสมบูรณ์

```csharp
// ── DTOs with Validation ─────────────────────────────────────
public class CreateProductDto
{
    [Required]
    [StringLength(200, MinimumLength = 3)]
    public string Name { get; set; } = "";

    [StringLength(2000)]
    public string? Description { get; set; }

    [Required]
    [Range(0.01, 1_000_000)]
    public decimal Price { get; set; }

    [Range(0, 100_000)]
    public int Stock { get; set; }

    [Required]
    public string Category { get; set; } = "";
}

// ── FluentValidator ──────────────────────────────────────────
public class CreateProductValidator : AbstractValidator<CreateProductDto>
{
    private static readonly string[] ValidCategories =
        ["Electronics", "Clothing", "Food", "Books"];

    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .Length(3, 200);

        RuleFor(x => x.Price)
            .GreaterThan(0)
            .WithMessage("Price must be greater than 0")
            .LessThanOrEqualTo(1_000_000)
            .WithMessage("Price cannot exceed 1,000,000");

        RuleFor(x => x.Category)
            .NotEmpty()
            .Must(c => ValidCategories.Contains(c))
            .WithMessage($"Category must be one of: {string.Join(", ", ValidCategories)}");

        // conditional — description required ถ้าราคาเกิน 10,000
        When(x => x.Price > 10_000, () =>
        {
            RuleFor(x => x.Description)
                .NotEmpty()
                .WithMessage("Description is required for products over 10,000");
        });
    }
}

// ── Controller ───────────────────────────────────────────────
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private static readonly List<Product> _products = new();
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(ILogger<ProductsController> logger)
        => _logger = logger;

    [HttpPost]
    [ProducesResponseType<Product>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public IActionResult Create(CreateProductDto dto)
    {
        // validation ผ่านแล้ว — ถึงตรงนี้ data ถูกต้องแน่นอน
        var product = new Product(
            _products.Count + 1,
            dto.Name,
            dto.Price,
            dto.Stock,
            dto.Category);

        _products.Add(product);
        _logger.LogInformation("Created product {ProductId}: {Name}", product.Id, product.Name);

        return CreatedAtAction("GetById", new { id = product.Id }, product);
    }

    [HttpGet("{id:int}")]
    public ActionResult<Product> GetById(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        return product is null ? NotFound() : Ok(product);
    }
}

public record Product(int Id, string Name, decimal Price, int Stock, string Category);

// ── Program.cs ───────────────────────────────────────────────
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductValidator>();

var app = builder.Build();
app.MapControllers();
await app.RunAsync();
```

ทดสอบ:
```bash
# valid request
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":25000,"stock":10,"category":"Electronics","description":"High-end laptop"}'

# invalid — missing required, price negative, wrong category
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"A","price":-100,"stock":0,"category":"Invalid"}'

# conditional — price > 10000 but no description
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Expensive Item","price":15000,"stock":5,"category":"Electronics"}'
```

---

## บทสรุป

```
3 ชั้น Validation:

1. Data Annotations        → format / range ของ input field
   [Required], [Range], [EmailAddress], [StringLength]
   เหมาะ: simple, quick, ไม่ต้องการ dependency

2. FluentValidation        → rules ซับซ้อน, async, dependency
   .MustAsync(), .When(), RuleForEach(), SetValidator()
   เหมาะ: business-oriented rules, ทดสอบง่าย

3. Service Layer           → business rules ที่ต้องการ context
   NotFoundException, BusinessRuleException
   เหมาะ: "user must exist", "stock must be sufficient"

Response Format:
  [ApiController] → 400 ProblemDetails อัตโนมัติ
  errors แยกตาม field ชัดเจน
```

---

## Workshop

1. **Lab 9.1** — เพิ่ม `UpdateProductDto` พร้อม FluentValidator ที่ validate ว่าถ้า `Stock` ลดลงต้องมี `Reason` อธิบาย

2. **Lab 9.2** — สร้าง `BookingValidator` ที่ใช้ `IValidatableObject` ตรวจสอบ `CheckIn` < `CheckOut` และจองล่วงหน้าอย่างน้อย 24 ชั่วโมง

3. **Lab 9.3** — ทดสอบ FluentValidator โดยตรงโดยไม่ต้องผ่าน HTTP:
   ```csharp
   var validator = new CreateProductValidator();
   var result    = await validator.ValidateAsync(new CreateProductDto { Name = "A" });
   Assert.False(result.IsValid);
   ```

---

*→ บทถัดไป: **Part 10 — Authentication & Authorization***
