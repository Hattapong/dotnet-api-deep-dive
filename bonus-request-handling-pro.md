# บทอ่านเสริม — การจัดการ Request แบบ Pro

> **Deep Dive Series: .NET Web API Internals**  
> Model Binding พื้นฐานเราเห็นในบท Controller แล้ว  
> บทนี้จะดูทุก edge case และ pattern ที่เจอใน production จริงๆ

---

## 1. Query Parameters — มากกว่าแค่ `?key=value`

### พื้นฐาน — Auto Binding

```csharp
// simple types bind จาก query string อัตโนมัติ
// GET /api/products?page=2&size=20&search=laptop&inStock=true

[HttpGet]
public IActionResult GetAll(
    int     page    = 1,
    int     size    = 20,
    string? search  = null,
    bool    inStock = false)
{
    // page=2, size=20, search="laptop", inStock=true
}
```

### Array และ List จาก Query

```csharp
// GET /api/products?categoryId=1&categoryId=2&categoryId=3
// หรือ
// GET /api/products?categoryId=1,2,3   (comma-separated ต้องระบุเอง)

[HttpGet]
public IActionResult GetByCategories(
    [FromQuery] int[] categoryId,        // bind แบบ repeated key
    [FromQuery] string? tags = null)     // "dotnet,webapi,csharp"
{
    var tagList = tags?.Split(',') ?? [];
    // categoryId = [1, 2, 3]
    // tagList    = ["dotnet", "webapi", "csharp"]
}
```

### Complex Object จาก Query

```csharp
// แทนที่จะมี 10 parameters แยก — รวมเป็น object เดียว
public class ProductFilter
{
    public string?   Search     { get; set; }
    public decimal?  MinPrice   { get; set; }
    public decimal?  MaxPrice   { get; set; }
    public int[]     CategoryId { get; set; } = [];
    public bool      InStock    { get; set; } = false;
    public string    SortBy     { get; set; } = "name";
    public SortOrder Order      { get; set; } = SortOrder.Asc;
}

public enum SortOrder { Asc, Desc }

// GET /api/products?search=laptop&minPrice=1000&categoryId=1&categoryId=2&sortBy=price&order=desc
[HttpGet]
public IActionResult GetAll([FromQuery] ProductFilter filter)
{
    // filter.Search   = "laptop"
    // filter.MinPrice = 1000
    // filter.CategoryId = [1, 2]
    // filter.SortBy   = "price"
    // filter.Order    = SortOrder.Desc
}
```

### Pagination Pattern ที่ใช้บ่อย

```csharp
// Standard pagination object — reuse ได้ทุก endpoint
public class PaginationParams
{
    private int _page = 1;
    private int _size = 20;

    public int Page
    {
        get => _page;
        set => _page = value < 1 ? 1 : value;      // min = 1
    }

    public int Size
    {
        get => _size;
        set => _size = value > 100 ? 100 : value < 1 ? 1 : value;  // 1-100
    }

    public int Skip => (Page - 1) * Size;
}

// Paginated response — client รู้ว่ามีกี่หน้าทั้งหมด
public class PagedResult<T>
{
    public IReadOnlyList<T> Items      { get; init; } = [];
    public int              TotalCount { get; init; }
    public int              Page       { get; init; }
    public int              PageSize   { get; init; }
    public int              TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool             HasNext    => Page < TotalPages;
    public bool             HasPrev    => Page > 1;
}

[HttpGet]
public ActionResult<PagedResult<Product>> GetAll(
    [FromQuery] PaginationParams pagination,
    [FromQuery] ProductFilter    filter)
{
    var query   = _repo.Query(filter);
    var total   = query.Count();
    var items   = query.Skip(pagination.Skip).Take(pagination.Size).ToList();

    return Ok(new PagedResult<Product>
    {
        Items      = items,
        TotalCount = total,
        Page       = pagination.Page,
        PageSize   = pagination.Size
    });
}
```

### Query String Validation

```csharp
public class ProductFilter : IValidatableObject
{
    [Range(0, double.MaxValue)]
    public decimal? MinPrice { get; set; }

    [Range(0, double.MaxValue)]
    public decimal? MaxPrice { get; set; }

    [RegularExpression("^(name|price|created)$",
        ErrorMessage = "SortBy must be 'name', 'price', or 'created'")]
    public string SortBy { get; set; } = "name";

    public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
    {
        if (MinPrice.HasValue && MaxPrice.HasValue && MinPrice > MaxPrice)
            yield return new ValidationResult(
                "MinPrice cannot be greater than MaxPrice",
                new[] { nameof(MinPrice), nameof(MaxPrice) });
    }
}
```

---

## 2. Route Path Parameters — ทุก Pattern

### Basic

```csharp
// GET /api/users/42
[HttpGet("{id:int}")]
public IActionResult GetById(int id) { }

// GET /api/files/documents/report.pdf
[HttpGet("{**filePath}")]   // catch-all — match ทุกอย่างรวม /
public IActionResult GetFile(string filePath)
{
    // filePath = "documents/report.pdf"
}
```

### Constraints ที่ใช้บ่อย

```csharp
[HttpGet("{id:int}")]            // ต้องเป็น int
[HttpGet("{id:int:min(1)}")]     // int และ >= 1
[HttpGet("{id:guid}")]           // ต้อง parse เป็น Guid ได้
[HttpGet("{date:datetime}")]     // ต้อง parse เป็น DateTime ได้
[HttpGet("{name:alpha}")]        // ตัวอักษรเท่านั้น
[HttpGet("{name:minlength(3)}")] // string ยาวอย่างน้อย 3
[HttpGet("{code:length(6)}")]    // string ยาวพอดี 6 ตัว
[HttpGet("{slug:regex(^[[a-z0-9-]]+$)}")]  // match regex (double [[ เพราะ escape])
```

### Optional Parameters

```csharp
// GET /api/categories/5          → parentId = 5
// GET /api/categories             → parentId = null
[HttpGet("{parentId:int?}")]
[HttpGet]
public IActionResult GetByParent(int? parentId = null)
{
    var categories = parentId.HasValue
        ? _repo.GetChildren(parentId.Value)
        : _repo.GetRoots();
    return Ok(categories);
}
```

### Multiple Route Parameters

```csharp
// GET /api/users/42/orders/99
[HttpGet("{userId:int}/orders/{orderId:int}")]
public async Task<IActionResult> GetUserOrder(
    int userId, int orderId, CancellationToken ct)
{
    var order = await _orderService.GetAsync(userId, orderId, ct);
    return order is null ? NotFound() : Ok(order);
}

// GET /api/orgs/acme/repos/my-project/commits/abc123
[HttpGet("{org}/{repo}/commits/{sha}")]
public IActionResult GetCommit(string org, string repo, string sha)
{
    return Ok(new { org, repo, sha });
}
```

### Custom Route Constraint

```csharp
// สร้าง constraint ที่ validate slug format
public class SlugRouteConstraint : IRouteConstraint
{
    private static readonly Regex SlugRegex =
        new(@"^[a-z0-9]+(?:-[a-z0-9]+)*$", RegexOptions.Compiled);

    public bool Match(HttpContext? httpContext, IRouter? route,
        string routeKey, RouteValueDictionary values, RouteDirection direction)
    {
        if (values.TryGetValue(routeKey, out var value) && value is string str)
            return SlugRegex.IsMatch(str);
        return false;
    }
}

// Register
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("slug", typeof(SlugRouteConstraint));
});

// ใช้งาน
[HttpGet("{slug:slug}")]
public IActionResult GetBySlug(string slug) { }
// GET /api/posts/my-first-post  ✅
// GET /api/posts/My_Post!        ❌ → 404
```

---

## 3. Request Body — ทุกรูปแบบ

### JSON — Default

```csharp
// POST /api/orders
// Content-Type: application/json
// Body: {"userId":1,"items":[{"productId":5,"qty":2}]}

[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] CreateOrderDto dto,    // อ่าน JSON body
    CancellationToken ct)
{
    // dto ถูก deserialize แล้ว
}

// DTO
public class CreateOrderDto
{
    [Required]
    public int UserId { get; set; }

    [Required]
    [MinLength(1, ErrorMessage = "At least one item required")]
    public List<OrderItemDto> Items { get; set; } = [];
}

public class OrderItemDto
{
    [Required] public int ProductId { get; set; }
    [Range(1, 1000)] public int Qty { get; set; }
}
```

### JSON Serialization Options

```csharp
// Program.cs — ปรับ JSON behavior
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        // camelCase ↔ PascalCase (default ใน .NET)
        options.JsonSerializerOptions.PropertyNamingPolicy =
            JsonNamingPolicy.CamelCase;

        // ไม่ serialize property ที่เป็น null
        options.JsonSerializerOptions.DefaultIgnoreCondition =
            JsonIgnoreCondition.WhenWritingNull;

        // อ่าน enum เป็น string ("Active" แทน 1)
        options.JsonSerializerOptions.Converters.Add(
            new JsonStringEnumConverter());

        // tolerant ต่อ trailing commas และ comments ใน JSON
        options.JsonSerializerOptions.ReadCommentHandling =
            JsonCommentHandling.Skip;
        options.JsonSerializerOptions.AllowTrailingCommas = true;
    });
```

### Form Data

```csharp
// HTML form หรือ API ที่ส่ง application/x-www-form-urlencoded
[HttpPost("login")]
[Consumes("application/x-www-form-urlencoded")]
public IActionResult Login([FromForm] LoginRequest req)
{
    return Ok();
}

public class LoginRequest
{
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
}
```

### File Upload — Single File

```csharp
// Content-Type: multipart/form-data
[HttpPost("avatar")]
[RequestSizeLimit(5 * 1024 * 1024)]           // จำกัด 5MB
[RequestFormLimits(MultipartBodyLengthLimit = 5 * 1024 * 1024)]
public async Task<IActionResult> UploadAvatar(
    int userId,
    IFormFile file,
    CancellationToken ct)
{
    // validate
    if (file.Length == 0)
        return BadRequest("File is empty");

    var allowedTypes = new[] { "image/jpeg", "image/png", "image/webp" };
    if (!allowedTypes.Contains(file.ContentType))
        return BadRequest("Only JPEG, PNG, WebP allowed");

    // อ่าน stream
    using var stream = file.OpenReadStream();
    var bytes = new byte[file.Length];
    await stream.ReadExactlyAsync(bytes, ct);

    // หรือ save ลง disk
    var path = Path.Combine("uploads", $"{userId}_{file.FileName}");
    using var dest = File.Create(path);
    await file.CopyToAsync(dest, ct);

    return Ok(new { fileName = file.FileName, size = file.Length });
}
```

### File Upload — Multiple Files + Form Fields

```csharp
// DTO สำหรับ multipart form ที่มีทั้งไฟล์และ fields
public class ProductCreateRequest
{
    [Required]
    public string Name { get; set; } = "";

    [Range(0, double.MaxValue)]
    public decimal Price { get; set; }

    // ไฟล์ภาพหลัก
    public IFormFile?       MainImage  { get; set; }

    // ไฟล์หลายไฟล์
    public IFormFileCollection? Gallery { get; set; }
}

[HttpPost]
[Consumes("multipart/form-data")]
public async Task<IActionResult> CreateProduct(
    [FromForm] ProductCreateRequest req,
    CancellationToken ct)
{
    var imageUrls = new List<string>();

    if (req.Gallery is not null)
    {
        foreach (var file in req.Gallery)
        {
            if (file.Length > 0)
            {
                var url = await _storage.UploadAsync(file, ct);
                imageUrls.Add(url);
            }
        }
    }

    return Ok(new { req.Name, req.Price, imageUrls });
}
```

### Raw Body — อ่าน Stream ตรงๆ

```csharp
// สำหรับ webhook หรือ payload ที่ไม่ใช่ JSON มาตรฐาน
[HttpPost("webhook")]
public async Task<IActionResult> ReceiveWebhook(CancellationToken ct)
{
    // เปิด buffering ก่อนอ่าน body ซ้ำ
    Request.EnableBuffering();

    using var reader = new StreamReader(
        Request.Body,
        Encoding.UTF8,
        leaveOpen: true);

    var rawBody = await reader.ReadToEndAsync(ct);

    // verify signature (เช่น Stripe, GitHub webhook)
    var signature = Request.Headers["X-Signature"].ToString();
    if (!VerifySignature(rawBody, signature))
        return Unauthorized();

    // process
    var payload = JsonSerializer.Deserialize<WebhookPayload>(rawBody);
    await _webhookService.ProcessAsync(payload!, ct);

    return Ok();
}

// อ่านเป็น byte array (เช่น binary payload)
[HttpPost("binary")]
public async Task<IActionResult> ReceiveBinary()
{
    using var ms = new MemoryStream();
    await Request.Body.CopyToAsync(ms);
    var bytes = ms.ToArray();

    return Ok(new { size = bytes.Length });
}
```

### Content Negotiation

```csharp
// บอก action ว่า accept content type อะไร
[HttpPost]
[Consumes("application/json", "application/xml")]
public IActionResult Create([FromBody] CreateProductDto dto) { }

// บอกว่า produce อะไร
[HttpGet("{id}")]
[Produces("application/json")]
public IActionResult GetById(int id) { }
```

---

## 4. Headers — อ่านและเขียน

### อ่าน Request Headers

```csharp
[HttpGet]
public IActionResult Get()
{
    // วิธีที่ 1: จาก Request.Headers โดยตรง
    string? auth       = Request.Headers.Authorization;
    string? userAgent  = Request.Headers.UserAgent;
    string? acceptLang = Request.Headers.AcceptLanguage;
    string? customHdr  = Request.Headers["X-Custom-Header"];

    // วิธีที่ 2: [FromHeader] binding
    return Ok();
}

// วิธีที่ 2: [FromHeader] — bind ตรงเข้า parameter
[HttpGet("localized")]
public IActionResult GetLocalized(
    [FromHeader(Name = "Accept-Language")] string? language = "en",
    [FromHeader(Name = "X-Api-Version")]   string? version  = "1")
{
    return Ok(new { language, version });
}
```

### เขียน Response Headers

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    var product = _repo.GetById(id);
    if (product is null) return NotFound();

    // เพิ่ม headers ก่อน return เสมอ
    Response.Headers["X-Resource-Id"]   = id.ToString();
    Response.Headers["X-Last-Modified"] = product.UpdatedAt.ToString("R");

    // Cache-Control
    Response.Headers["Cache-Control"] = "public, max-age=300";
    Response.Headers["ETag"]          = $"\"{product.Version}\"";

    return Ok(product);
}
```

### Custom Header Middleware

```csharp
// เพิ่ม security headers ทุก response อัตโนมัติ
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"]        = "DENY";
    ctx.Response.Headers["X-XSS-Protection"]       = "1; mode=block";
    ctx.Response.Headers["Referrer-Policy"]        = "strict-origin-when-cross-origin";

    // ลบ header ที่เปิดเผยข้อมูล server
    ctx.Response.Headers.Remove("Server");
    ctx.Response.Headers.Remove("X-Powered-By");

    await next();
});
```

### Versioning ด้วย Header

```csharp
// ติดตั้ง
dotnet add package Asp.Versioning.Http

// Program.cs
builder.Services
    .AddApiVersioning(options =>
    {
        options.DefaultApiVersion                = new ApiVersion(1, 0);
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.ApiVersionReader = ApiVersionReader.Combine(
            new HeaderApiVersionReader("X-Api-Version"),  // จาก header
            new QueryStringApiVersionReader("version"),    // หรือจาก query
            new MediaTypeApiVersionReader("ver"));         // หรือจาก Accept header
    })
    .AddMvc();

// Controller
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new { version = "v1", users = new[] { "Alice" } });

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new
    {
        version = "v2",
        users   = new[] { new { id = 1, name = "Alice", email = "alice@example.com" } }
    });
}
```

---

## 5. Model Binding ขั้นสูง

### Custom Model Binder

เมื่อ default binding ไม่พอ สร้าง binder เองได้:

```csharp
// รับ comma-separated string แล้วแปลงเป็น List<int>
// GET /api/products?ids=1,2,3,4,5
public class CommaDelimitedIntsBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext ctx)
    {
        var value = ctx.ValueProvider.GetValue(ctx.ModelName);

        if (value == ValueProviderResult.None)
        {
            ctx.Result = ModelBindingResult.Success(new List<int>());
            return Task.CompletedTask;
        }

        var raw  = value.FirstValue ?? "";
        var ids  = raw.Split(',', StringSplitOptions.RemoveEmptyEntries)
                      .Select(s => int.TryParse(s.Trim(), out var n) ? (int?)n : null)
                      .Where(n => n.HasValue)
                      .Select(n => n!.Value)
                      .ToList();

        ctx.Result = ModelBindingResult.Success(ids);
        return Task.CompletedTask;
    }
}

// บอก ASP.NET Core ว่าใช้ binder นี้กับ parameter ไหน
[HttpGet]
public IActionResult GetByIds(
    [ModelBinder(typeof(CommaDelimitedIntsBinder))] List<int> ids)
{
    // GET /api/products?ids=1,2,3  →  ids = [1, 2, 3]
}
```

### IActionFilter — ทำงานก่อน/หลัง Action

```csharp
// Validate + transform ก่อน action รัน
public class SanitizeQueryFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext ctx)
    {
        // trim whitespace จาก string parameters ทุกตัว
        foreach (var param in ctx.ActionArguments.ToList())
        {
            if (param.Value is string str)
                ctx.ActionArguments[param.Key] = str.Trim();
        }
    }

    public void OnActionExecuted(ActionExecutedContext ctx) { }
}

// ใช้งาน
[ServiceFilter(typeof(SanitizeQueryFilter))]
[HttpGet]
public IActionResult Search(string query)
{
    // query ถูก trim แล้วก่อนถึงตรงนี้
}
```

---

## 6. Response Shaping

### เลือก Fields ที่ต้องการ (Sparse Fieldset)

```csharp
// GET /api/users?fields=id,name,email
[HttpGet("{id}")]
public IActionResult GetById(int id, [FromQuery] string? fields = null)
{
    var user = _repo.GetById(id);
    if (user is null) return NotFound();

    if (fields is null) return Ok(user);

    // return เฉพาะ fields ที่ขอ
    var fieldSet = fields.Split(',').ToHashSet(StringComparer.OrdinalIgnoreCase);

    var result = new Dictionary<string, object?>();
    foreach (var prop in typeof(User).GetProperties())
    {
        if (fieldSet.Contains(prop.Name))
            result[prop.Name.ToLower()] = prop.GetValue(user);
    }

    return Ok(result);
}
```

### Conditional Response — ETag

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    var product = _repo.GetById(id);
    if (product is null) return NotFound();

    // สร้าง ETag จาก version หรือ hash
    var etag = $"\"{product.Version}\"";

    // ถ้า client ส่ง If-None-Match ที่ตรงกัน → 304 Not Modified
    if (Request.Headers.IfNoneMatch == etag)
        return StatusCode(304);

    Response.Headers.ETag = etag;
    return Ok(product);
}
```

---

## 7. Request Context Helpers

### Extension Methods ที่ควรมีในทุก Project

```csharp
public static class HttpRequestExtensions
{
    // ดึง IP จริงของ client (รองรับ reverse proxy)
    public static string GetClientIp(this HttpRequest request)
    {
        var forwarded = request.Headers["X-Forwarded-For"].FirstOrDefault();
        if (!string.IsNullOrEmpty(forwarded))
            return forwarded.Split(',')[0].Trim();

        return request.HttpContext.Connection.RemoteIpAddress?.ToString()
               ?? "unknown";
    }

    // ตรวจว่า request ต้องการ JSON
    public static bool IsJsonRequest(this HttpRequest request)
        => request.ContentType?.Contains("application/json") ?? false;

    // ดึง correlation id
    public static string GetCorrelationId(this HttpRequest request)
        => request.Headers["X-Correlation-Id"].FirstOrDefault()
           ?? request.HttpContext.TraceIdentifier;

    // ตรวจว่า request มาจาก mobile
    public static bool IsMobileRequest(this HttpRequest request)
    {
        var ua = request.Headers.UserAgent.ToString();
        return ua.Contains("Mobile") || ua.Contains("Android")
            || ua.Contains("iPhone");
    }
}

public static class HttpContextExtensions
{
    // ดึง userId จาก claims อย่างปลอดภัย
    public static int? GetUserId(this HttpContext ctx)
    {
        var value = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
        return int.TryParse(value, out var id) ? id : null;
    }

    // เช็คว่า response เริ่ม write แล้วหรือยัง
    public static bool HasStarted(this HttpContext ctx)
        => ctx.Response.HasStarted;
}
```

---

## 8. ตัวอย่างสมบูรณ์ — Product API ที่ใช้ทุก Technique

```csharp
// Models
public record Product(int Id, string Name, decimal Price,
    int Stock, string Category, DateTime CreatedAt);

public class ProductFilter
{
    public string?   Search     { get; set; }
    public decimal?  MinPrice   { get; set; }
    public decimal?  MaxPrice   { get; set; }
    public string[]  Category   { get; set; } = [];
    public bool?     InStock    { get; set; }
    public string    SortBy     { get; set; } = "name";
    public string    Order      { get; set; } = "asc";

    public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
    {
        if (MinPrice.HasValue && MaxPrice.HasValue && MinPrice > MaxPrice)
            yield return new ValidationResult("MinPrice cannot exceed MaxPrice");
    }
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private static readonly List<Product> _db = new()
    {
        new(1, "Laptop Pro",  45000, 10, "Electronics", DateTime.UtcNow),
        new(2, "Wireless Mouse", 890, 50, "Electronics", DateTime.UtcNow),
        new(3, "Design Book",  350, 100, "Books",       DateTime.UtcNow),
    };

    // ── GET /api/products?search=laptop&minPrice=500&category=Electronics&page=1 ──
    [HttpGet]
    public ActionResult<PagedResult<Product>> GetAll(
        [FromQuery] ProductFilter    filter,
        [FromQuery] PaginationParams pagination,
        [FromHeader(Name = "Accept-Language")] string? lang = "en")
    {
        var query = _db.AsQueryable();

        if (!string.IsNullOrWhiteSpace(filter.Search))
            query = query.Where(p =>
                p.Name.Contains(filter.Search, StringComparison.OrdinalIgnoreCase));

        if (filter.MinPrice.HasValue)
            query = query.Where(p => p.Price >= filter.MinPrice.Value);

        if (filter.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);

        if (filter.Category.Length > 0)
            query = query.Where(p => filter.Category.Contains(p.Category));

        if (filter.InStock.HasValue)
            query = query.Where(p =>
                filter.InStock.Value ? p.Stock > 0 : p.Stock == 0);

        query = (filter.SortBy.ToLower(), filter.Order.ToLower()) switch
        {
            ("price", "desc") => query.OrderByDescending(p => p.Price),
            ("price", _)      => query.OrderBy(p => p.Price),
            ("created", "desc") => query.OrderByDescending(p => p.CreatedAt),
            _                 => query.OrderBy(p => p.Name)
        };

        var total = query.Count();
        var items = query.Skip(pagination.Skip).Take(pagination.Size).ToList();

        Response.Headers["X-Total-Count"] = total.ToString();
        Response.Headers["Content-Language"] = lang ?? "en";

        return Ok(new PagedResult<Product>
        {
            Items = items, TotalCount = total,
            Page  = pagination.Page, PageSize = pagination.Size
        });
    }

    // ── GET /api/products/1 ──────────────────────────────────
    [HttpGet("{id:int:min(1)}")]
    public ActionResult<Product> GetById(int id)
    {
        var product = _db.FirstOrDefault(p => p.Id == id);
        if (product is null) return NotFound();

        var etag = $"\"{product.CreatedAt.Ticks}\"";
        if (Request.Headers.IfNoneMatch == etag)
            return StatusCode(304);

        Response.Headers.ETag     = etag;
        Response.Headers["Cache-Control"] = "public, max-age=60";
        return Ok(product);
    }

    // ── POST /api/products — multipart/form-data ─────────────
    [HttpPost]
    [Consumes("multipart/form-data")]
    public async Task<ActionResult<Product>> Create(
        [FromForm] string  name,
        [FromForm] decimal price,
        [FromForm] int     stock,
        [FromForm] string  category,
        IFormFile?         image,
        CancellationToken  ct)
    {
        string? imageUrl = null;
        if (image is not null && image.Length > 0)
        {
            var allowedTypes = new[] { "image/jpeg", "image/png" };
            if (!allowedTypes.Contains(image.ContentType))
                return BadRequest(new { error = "Only JPEG/PNG images allowed" });

            imageUrl = $"/uploads/{Guid.NewGuid()}{Path.GetExtension(image.FileName)}";
            // await _storage.UploadAsync(image, imageUrl, ct);
        }

        var product = new Product(
            _db.Max(p => p.Id) + 1, name, price, stock, category, DateTime.UtcNow);
        _db.Add(product);

        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    // ── POST /api/products/webhook — raw body ────────────────
    [HttpPost("webhook")]
    public async Task<IActionResult> Webhook(CancellationToken ct)
    {
        var signature = Request.Headers["X-Webhook-Signature"].ToString();
        if (string.IsNullOrEmpty(signature))
            return Unauthorized();

        Request.EnableBuffering();
        using var reader = new StreamReader(Request.Body, Encoding.UTF8, leaveOpen: true);
        var body = await reader.ReadToEndAsync(ct);

        // verify HMAC signature
        // if (!HmacHelper.Verify(body, signature, _secret)) return Unauthorized();

        return Ok(new { received = true, length = body.Length });
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers()
    .AddJsonOptions(o =>
    {
        o.JsonSerializerOptions.PropertyNamingPolicy         = JsonNamingPolicy.CamelCase;
        o.JsonSerializerOptions.DefaultIgnoreCondition       = JsonIgnoreCondition.WhenWritingNull;
        o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    });

var app = builder.Build();

// Security headers
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"]        = "DENY";
    ctx.Response.Headers.Remove("Server");
    await next();
});

app.MapControllers();
await app.RunAsync();
```

ทดสอบ:
```bash
# filter + pagination
curl "http://localhost:5000/api/products?search=&minPrice=500&category=Electronics&page=1&size=10"

# language header
curl http://localhost:5000/api/products \
  -H "Accept-Language: th"

# ETag caching
curl -v http://localhost:5000/api/products/1
# เอา ETag ที่ได้ใส่ใน If-None-Match → ได้ 304
curl http://localhost:5000/api/products/1 \
  -H 'If-None-Match: "..."'

# multipart upload
curl -X POST http://localhost:5000/api/products \
  -F "name=New Product" \
  -F "price=999" \
  -F "stock=50" \
  -F "category=Electronics" \
  -F "image=@photo.jpg"
```

---

## บทสรุป

```
Query Parameters:
  ✅ Complex object ด้วย [FromQuery] SomeClass filter
  ✅ Array ด้วย repeated key หรือ comma-separated
  ✅ Pagination object แยก reusable
  ✅ Validate query object ด้วย IValidatableObject

Route Parameters:
  ✅ Constraints: :int :guid :min(1) :regex(pattern)
  ✅ Optional: {id:int?}
  ✅ Catch-all: {**path}
  ✅ Custom constraint ด้วย IRouteConstraint

Request Body:
  ✅ JSON — [FromBody]
  ✅ Form data — [FromForm] + [Consumes("multipart/form-data")]
  ✅ File upload — IFormFile / IFormFileCollection
  ✅ Raw body — Request.EnableBuffering() + StreamReader

Headers:
  ✅ อ่าน: Request.Headers["X-Key"] หรือ [FromHeader]
  ✅ เขียน: Response.Headers["X-Key"] = value
  ✅ Security headers ผ่าน middleware
  ✅ API Versioning ผ่าน header

Advanced:
  ✅ ETag + If-None-Match สำหรับ cache
  ✅ Custom Model Binder สำหรับ format พิเศษ
  ✅ Extension methods สำหรับ HttpRequest / HttpContext
```

---

## Workshop

1. **Lab B.1** — เพิ่ม `GET /api/products/search` ที่รับ `ids` เป็น comma-separated string (`?ids=1,2,3`) โดยใช้ Custom Model Binder แล้ว return เฉพาะ products ที่ match

2. **Lab B.2** — Implement ETag caching ให้ครบ cycle: GET → ได้ ETag → GET พร้อม `If-None-Match` → ได้ 304 → แก้ข้อมูล → GET อีกครั้ง → ได้ 200 พร้อม ETag ใหม่

3. **Lab B.3** — สร้าง webhook endpoint ที่ verify HMAC-SHA256 signature จาก header `X-Signature` โดยใช้ `HMACSHA256` และ `System.Security.Cryptography`

4. **Lab B.4 (Bonus)** — Implement sparse fieldset: `GET /api/products/1?fields=id,name,price` ที่ return เฉพาะ fields ที่ขอ โดยใช้ Reflection อ่าน property ตามชื่อ
```
