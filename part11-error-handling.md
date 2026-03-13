# Part 11 — Error Handling

> **Deep Dive Series: .NET Web API Internals**  
> Exception ที่ไม่ถูกจัดการ = 500 ที่ไม่มีข้อมูล = client งง  
> บทนี้จะดูวิธีจัดการ error ทุกชั้นให้ consistent และเป็นระเบียบ

---

## 1. ปัญหาเมื่อไม่มี Error Handling

```csharp
// ❌ ไม่มี error handling — client ได้อะไร?
[HttpGet("{id}")]
public async Task<ActionResult<User>> GetById(int id)
{
    var user = await _repo.GetByIdAsync(id);  // อาจ throw SqlException
    return Ok(user);                           // อาจ NullReferenceException
}
```

ถ้า `GetByIdAsync` throw exception:
- Default ASP.NET Core → return `500 Internal Server Error`
- body มีแค่ `"An error occurred"` หรือ stack trace (dev mode)
- client ไม่รู้ว่าเกิดอะไรขึ้น
- log อาจไม่มีข้อมูลเพียงพอ

เป้าหมายของ error handling คือ:
- client ได้ response ที่ **consistent** และ **informative**
- server **log ครบ** ทุก error
- **ไม่รั่ว** sensitive information (stack trace, connection string)

---

## 2. ProblemDetails — Standard Error Format

RFC 7807 กำหนด format มาตรฐานสำหรับ HTTP API errors ที่ .NET implement ให้แล้ว:

```json
{
  "type":     "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title":    "Not Found",
  "status":   404,
  "detail":   "User with id 42 was not found",
  "instance": "/api/users/42",
  "traceId":  "00-abc123-def456-00"
}
```

| Field | ความหมาย |
|-------|----------|
| `type` | URI ที่อธิบายประเภท error (RFC link หรือ custom) |
| `title` | ชื่อสั้นๆ ของ error ที่ไม่เปลี่ยนต่อ request |
| `status` | HTTP status code |
| `detail` | คำอธิบาย error ที่ specific กับ request นี้ |
| `instance` | URI ของ request ที่เกิด error |
| `traceId` | สำหรับ correlate กับ log |

---

## 3. Exception Middleware — จัดการทุก Exception ที่กลาง

แนวทางที่ดีที่สุดคือ **catch exception ทั้งหมดที่จุดเดียว** แล้ว map ไปยัง HTTP response ที่เหมาะสม

```csharp
// ExceptionHandlingMiddleware.cs
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate             _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        try
        {
            await _next(ctx);  // เรียก pipeline ที่เหลือ
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(ctx, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext ctx, Exception ex)
    {
        // map exception type → status code + message
        var (statusCode, title, detail) = ex switch
        {
            NotFoundException e      => (404, "Not Found",            e.Message),
            BusinessRuleException e  => (422, "Business Rule Violated", e.Message),
            UnauthorizedException e  => (401, "Unauthorized",          e.Message),
            ForbiddenException e     => (403, "Forbidden",             e.Message),
            ConflictException e      => (409, "Conflict",              e.Message),
            ValidationException e    => (400, "Validation Failed",     e.Message),
            _                        => (500, "Internal Server Error",
                                            "An unexpected error occurred")
        };

        // log ตาม severity
        if (statusCode >= 500)
            _logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);
        else
            _logger.LogWarning("Handled exception [{Status}]: {Message}",
                statusCode, ex.Message);

        // สร้าง ProblemDetails response
        ctx.Response.StatusCode  = statusCode;
        ctx.Response.ContentType = "application/problem+json";

        var problem = new ProblemDetails
        {
            Type     = GetTypeUri(statusCode),
            Title    = title,
            Status   = statusCode,
            Detail   = detail,
            Instance = ctx.Request.Path
        };

        // เพิ่ม traceId เพื่อ correlate กับ log
        problem.Extensions["traceId"] = ctx.TraceIdentifier;

        await ctx.Response.WriteAsJsonAsync(problem);
    }

    private static string GetTypeUri(int statusCode) => statusCode switch
    {
        400 => "https://tools.ietf.org/html/rfc9110#section-15.5.1",
        401 => "https://tools.ietf.org/html/rfc9110#section-15.5.2",
        403 => "https://tools.ietf.org/html/rfc9110#section-15.5.4",
        404 => "https://tools.ietf.org/html/rfc9110#section-15.5.5",
        409 => "https://tools.ietf.org/html/rfc9110#section-15.5.10",
        422 => "https://tools.ietf.org/html/rfc9110#section-15.5.21",
        _   => "https://tools.ietf.org/html/rfc9110#section-15.6.1"
    };
}
```

```csharp
// Program.cs — ต้องอยู่ต้น pipeline ก่อน middleware อื่นทุกตัว
app.UseMiddleware<ExceptionHandlingMiddleware>();  // ← บรรทัดแรก!
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

---

## 4. Custom Exception Types

กำหนด exception แต่ละชนิดให้ชัดเจน แทนที่จะ throw `Exception` ทั่วไป:

```csharp
// Exceptions.cs
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }

    // factory methods ที่สื่อความหมาย
    public static NotFoundException For<T>(int id)
        => new($"{typeof(T).Name} with id {id} was not found");

    public static NotFoundException For<T>(string field, string value)
        => new($"{typeof(T).Name} with {field} '{value}' was not found");
}

public class BusinessRuleException : Exception
{
    public string? RuleCode { get; }

    public BusinessRuleException(string message, string? ruleCode = null)
        : base(message)
    {
        RuleCode = ruleCode;
    }
}

public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }

    public static ConflictException AlreadyExists<T>(string field, string value)
        => new($"{typeof(T).Name} with {field} '{value}' already exists");
}

public class UnauthorizedException : Exception
{
    public UnauthorizedException(string message = "Authentication required")
        : base(message) { }
}

public class ForbiddenException : Exception
{
    public ForbiddenException(string message = "You do not have permission")
        : base(message) { }
}
```

### ใช้งานใน Service

```csharp
public class UserService
{
    public async Task<User> GetByIdAsync(int id, CancellationToken ct)
    {
        var user = await _repo.GetByIdAsync(id, ct);

        // throw ชัดเจน — middleware จะ map ให้
        return user ?? throw NotFoundException.For<User>(id);
    }

    public async Task<User> CreateAsync(CreateUserDto dto, CancellationToken ct)
    {
        var exists = await _repo.ExistsByEmailAsync(dto.Email, ct);
        if (exists)
            throw ConflictException.AlreadyExists<User>("email", dto.Email);

        var user = new User(dto.Name, dto.Email);
        await _repo.SaveAsync(user, ct);
        return user;
    }

    public async Task DeactivateAsync(int id, int requesterId, CancellationToken ct)
    {
        var user = await GetByIdAsync(id, ct);

        if (user.Id == requesterId)
            throw new BusinessRuleException(
                "You cannot deactivate your own account",
                ruleCode: "SELF_DEACTIVATION");

        user.Deactivate();
        await _repo.SaveAsync(user, ct);
    }
}
```

### Controller ที่สะอาด

```csharp
// ✅ Controller ไม่มี try/catch เลย — middleware จัดการให้
[HttpGet("{id:int}")]
public async Task<ActionResult<User>> GetById(int id, CancellationToken ct)
{
    var user = await _userService.GetByIdAsync(id, ct);
    return Ok(user);
    // ถ้า NotFoundException throw → middleware catch → 404 ProblemDetails
}

[HttpPost]
public async Task<ActionResult<User>> Create(CreateUserDto dto, CancellationToken ct)
{
    var user = await _userService.CreateAsync(dto, ct);
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    // ถ้า ConflictException → 409, ถ้า BusinessRuleException → 422
}
```

---

## 5. IExceptionHandler — .NET 8 Pattern

.NET 8 แนะนำ `IExceptionHandler` interface ที่ register ได้หลายตัวแบบ chain:

```csharp
// NotFoundExceptionHandler.cs
public class NotFoundExceptionHandler : IExceptionHandler
{
    private readonly ILogger<NotFoundExceptionHandler> _logger;

    public NotFoundExceptionHandler(ILogger<NotFoundExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx,
        Exception   ex,
        CancellationToken ct)
    {
        if (ex is not NotFoundException notFound)
            return false;  // ← บอก chain ว่า "ฉันจัดการไม่ได้ ส่งต่อ"

        _logger.LogWarning("Resource not found: {Message}", notFound.Message);

        ctx.Response.StatusCode = 404;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Type     = "https://tools.ietf.org/html/rfc9110#section-15.5.5",
            Title    = "Not Found",
            Status   = 404,
            Detail   = notFound.Message,
            Instance = ctx.Request.Path
        }, ct);

        return true;  // ← handled แล้ว หยุด chain
    }
}

// GlobalExceptionHandler.cs — fallback สำหรับทุกอย่างที่เหลือ
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx,
        Exception   ex,
        CancellationToken ct)
    {
        _logger.LogError(ex, "Unhandled exception");

        ctx.Response.StatusCode = 500;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Type   = "https://tools.ietf.org/html/rfc9110#section-15.6.1",
            Title  = "Internal Server Error",
            Status = 500,
            Detail = "An unexpected error occurred"
        }, ct);

        return true;
    }
}
```

```csharp
// Program.cs — .NET 8 style
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();  // เปิดใช้ built-in exception handler
```

---

## 6. Environment-Specific Error Detail

ระวังไม่ให้ stack trace รั่วออก production:

```csharp
private async Task HandleExceptionAsync(HttpContext ctx, Exception ex)
{
    var env = ctx.RequestServices
        .GetRequiredService<IHostEnvironment>();

    var (statusCode, title, detail) = ex switch
    {
        NotFoundException e => (404, "Not Found", e.Message),
        _                   => (500, "Internal Server Error",
                                "An unexpected error occurred")
    };

    var problem = new ProblemDetails
    {
        Title    = title,
        Status   = statusCode,
        Detail   = detail,
        Instance = ctx.Request.Path
    };

    // เพิ่ม stack trace เฉพาะ Development
    if (env.IsDevelopment())
    {
        problem.Extensions["stackTrace"]  = ex.StackTrace;
        problem.Extensions["exceptionType"] = ex.GetType().Name;
        // ใน dev เห็น stack trace เต็ม
        // ใน prod client เห็นแค่ "An unexpected error occurred"
    }

    problem.Extensions["traceId"] = ctx.TraceIdentifier;
    ctx.Response.StatusCode  = statusCode;
    ctx.Response.ContentType = "application/problem+json";
    await ctx.Response.WriteAsJsonAsync(problem);
}
```

---

## 7. ตัวอย่างสมบูรณ์

```csharp
// ── Custom Exceptions ────────────────────────────────────────
public class NotFoundException     : Exception
{
    public NotFoundException(string msg) : base(msg) { }
    public static NotFoundException For<T>(int id) =>
        new($"{typeof(T).Name} with id {id} not found");
}
public class ConflictException     : Exception
{
    public ConflictException(string msg) : base(msg) { }
}
public class BusinessRuleException : Exception
{
    public BusinessRuleException(string msg) : base(msg) { }
}

// ── Exception Middleware ─────────────────────────────────────
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    { _next = next; _logger = logger; }

    public async Task InvokeAsync(HttpContext ctx)
    {
        try { await _next(ctx); }
        catch (Exception ex) { await HandleAsync(ctx, ex); }
    }

    private async Task HandleAsync(HttpContext ctx, Exception ex)
    {
        var (status, title, detail) = ex switch
        {
            NotFoundException e      => (404, "Not Found",             e.Message),
            ConflictException e      => (409, "Conflict",              e.Message),
            BusinessRuleException e  => (422, "Unprocessable",         e.Message),
            _                        => (500, "Internal Server Error",
                                             "An unexpected error occurred")
        };

        if (status >= 500) _logger.LogError(ex, "Unhandled: {Msg}", ex.Message);
        else               _logger.LogWarning("[{Status}] {Msg}", status, ex.Message);

        ctx.Response.StatusCode  = status;
        ctx.Response.ContentType = "application/problem+json";
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title    = title,
            Status   = status,
            Detail   = detail,
            Instance = ctx.Request.Path,
            Extensions = { ["traceId"] = ctx.TraceIdentifier }
        });
    }
}

// ── Service ──────────────────────────────────────────────────
public interface IUserService
{
    Task<User> GetByIdAsync(int id, CancellationToken ct);
    Task<User> CreateAsync(CreateUserRequest req, CancellationToken ct);
}

public class UserService : IUserService
{
    private static readonly List<User> _users = new()
    {
        new(1, "Alice", "alice@example.com")
    };
    private static int _nextId = 2;

    public Task<User> GetByIdAsync(int id, CancellationToken ct)
    {
        var user = _users.FirstOrDefault(u => u.Id == id);
        return Task.FromResult(user ?? throw NotFoundException.For<User>(id));
    }

    public Task<User> CreateAsync(CreateUserRequest req, CancellationToken ct)
    {
        if (_users.Any(u => u.Email == req.Email))
            throw new ConflictException($"Email '{req.Email}' is already registered");

        if (req.Name.ToLower() == "admin")
            throw new BusinessRuleException("Username 'admin' is reserved");

        var user = new User(_nextId++, req.Name, req.Email);
        _users.Add(user);
        return Task.FromResult(user);
    }
}

// ── Controller ───────────────────────────────────────────────
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    public UsersController(IUserService service) => _service = service;

    [HttpGet("{id:int}")]
    public async Task<ActionResult<User>> GetById(int id, CancellationToken ct)
        => Ok(await _service.GetByIdAsync(id, ct));

    [HttpPost]
    public async Task<ActionResult<User>> Create(
        CreateUserRequest req, CancellationToken ct)
    {
        var user = await _service.CreateAsync(req, ct);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
}

public record User(int Id, string Name, string Email);
public record CreateUserRequest(
    [Required][StringLength(100, MinimumLength = 2)] string Name,
    [Required][EmailAddress] string Email);

// ── Program.cs ───────────────────────────────────────────────
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();
app.UseMiddleware<ExceptionHandlingMiddleware>();  // ← ต้องก่อนสุด
app.MapControllers();
await app.RunAsync();
```

ทดสอบทุก error path:
```bash
# 404 - not found
curl http://localhost:5000/api/users/999

# 409 - conflict (email ซ้ำ)
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Bob","email":"alice@example.com"}'

# 422 - business rule (reserved name)
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"admin","email":"admin@example.com"}'

# 400 - validation (bad format)
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"A","email":"notanemail"}'
```

---

## บทสรุป

```
Error Handling Architecture:

Request
  │
  ▼
ExceptionHandlingMiddleware (try/catch ทั้ง pipeline)
  │
  ▼
[ Validation ]   → 400 ProblemDetails (อัตโนมัติจาก [ApiController])
  │
  ▼
[ Controller ]   → ไม่มี try/catch — throw exceptions ตรงๆ
  │
  ▼
[ Service ]      → throw NotFoundException / ConflictException / BusinessRuleException
  │ exception
  ▼
ExceptionHandlingMiddleware
  ├── map exception type → status code
  ├── log (Warning สำหรับ 4xx, Error สำหรับ 5xx)
  └── return ProblemDetails JSON

กฎสำคัญ:
  ✅ Exception 1 ชนิด = HTTP status 1 ชนิด — consistent เสมอ
  ✅ Log ทุก exception ก่อน respond
  ✅ ไม่รั่ว stack trace ใน production
  ✅ ใช้ traceId เชื่อม error response กับ log
  ❌ ไม่ catch Exception แบบ swallow (catch แล้วไม่ทำอะไร)
  ❌ ไม่ return 200 เมื่อเกิด error
```

---

## Workshop

1. **Lab 11.1** — เพิ่ม `PaymentDeclinedException` ที่ map ไปยัง 402 Payment Required พร้อม `errorCode` field พิเศษใน ProblemDetails

2. **Lab 11.2** — เพิ่ม `RateLimitException` ที่ map ไปยัง 429 Too Many Requests พร้อม `Retry-After` header ที่บอกว่าอีกกี่วินาทีถึงลองได้

3. **Lab 11.3** — แก้ middleware ให้แสดง stack trace ใน Development แต่ซ่อนใน Production โดยอ่าน `IHostEnvironment` จาก DI

4. **Lab 11.4 (Bonus)** — แปลง `ExceptionHandlingMiddleware` เป็น `IExceptionHandler` แบบ .NET 8 โดยแยกเป็น handler ย่อย 1 ตัวต่อ 1 exception type

---

*→ บทถัดไป: **Part 10 — Authentication & Authorization***
