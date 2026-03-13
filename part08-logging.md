# Part 8 — Logging

> **Deep Dive Series: .NET Web API Internals**  
> `Console.WriteLine` ไม่ใช่ logging — บทนี้จะดูว่า logging จริงๆ ต่างกันอย่างไร  
> และทำให้ระบบ observable ในระดับ production ได้อย่างไร

---

## 1. ทำไม Console.WriteLine ไม่พอ?

```csharp
// ❌ Console.WriteLine — ปัญหาที่ตามมาเยอะ
Console.WriteLine($"[{DateTime.Now}] Getting user {id}");
Console.WriteLine($"Error: {ex.Message}");
```

| ปัญหา | ผลที่เกิด |
|-------|----------|
| ไม่มี level | ไม่รู้ว่าอันไหนสำคัญแค่ไหน |
| ไม่มี structure | ค้นหา / filter ด้วย tool ไม่ได้ |
| ไม่มี context | ไม่รู้ว่า log นี้มาจาก request ไหน |
| ไม่มี sink | output ออก console เท่านั้น ส่ง Datadog / Seq ไม่ได้ |
| ปิด/เปิดไม่ได้ | ทุก environment เห็นทุกอย่างเหมือนกัน |

---

## 2. ILogger — Logging Abstraction ของ .NET

```csharp
// ILogger<T> คือ interface กลาง — ไม่ผูกกับ implementation ใด
public class UserService
{
    private readonly ILogger<UserService> _logger;
    //                       ↑ T คือ "category" — บอกว่า log มาจากไหน

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }

    public async Task<User?> GetByIdAsync(int id)
    {
        _logger.LogInformation("Getting user {UserId}", id);
        // output: info: UserService[0]
        //               Getting user 42
    }
}
```

### Log Levels — 6 ระดับ

```csharp
// เรียงจากละเอียดที่สุด → ร้ายแรงที่สุด
_logger.LogTrace(    "Entering method GetByIdAsync, id={Id}", id);
_logger.LogDebug(    "Cache miss for key {Key}", cacheKey);
_logger.LogInformation("User {UserId} logged in from {Ip}", userId, ip);
_logger.LogWarning(  "Retry {Count}/{Max} for operation {Op}", count, max, op);
_logger.LogError(    ex, "Failed to process order {OrderId}", orderId);
_logger.LogCritical( "Database connection pool exhausted");
```

| Level | ใช้เมื่อ | Production? |
|-------|----------|:-----------:|
| `Trace` | ทุกก้าวย่อยของ code flow | ❌ |
| `Debug` | ข้อมูล diagnosis ที่ละเอียด | ❌ |
| `Information` | flow ปกติ, events สำคัญ | ✅ |
| `Warning` | สิ่งผิดปกติแต่ไม่ทำให้ล้มเหลว | ✅ |
| `Error` | operation ล้มเหลว แต่ app ยังทำงานได้ | ✅ |
| `Critical` | ล้มเหลวรุนแรง ต้องแก้ทันที | ✅ |

### ตั้งค่า Log Level ใน appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default":                    "Information",
      "Microsoft.AspNetCore":       "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

```json
// appsettings.Development.json — เปิด verbose ใน dev
{
  "Logging": {
    "LogLevel": {
      "Default":     "Debug",
      "MyApp":       "Trace"
    }
  }
}
```

---

## 3. Structured Logging — Log ที่ Query ได้

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง `Console.WriteLine` กับ `ILogger`

```csharp
// ❌ String interpolation — ข้อมูลกลายเป็น string ธรรมดา สูญเสีย structure
_logger.LogInformation($"User {userId} placed order {orderId} worth {amount:C}");
// output: "User 42 placed order 999 worth $150.00"
// → ค้นหาด้วย "orderId = 999" ไม่ได้ เพราะไม่มี field แยก

// ✅ Message Template — เก็บ property แยกกัน
_logger.LogInformation(
    "User {UserId} placed order {OrderId} worth {Amount}",
    userId, orderId, amount);
// output text:    "User 42 placed order 999 worth 150"
// structured:     { UserId: 42, OrderId: 999, Amount: 150.00 }
// → ค้นหา OrderId=999 ได้, filter Amount>100 ได้, aggregate ได้
```

### @ Destructuring Operator

```csharp
var user = new User(42, "Alice", "alice@example.com");

// ❌ log แค่ ToString()
_logger.LogInformation("Created user {User}", user);
// output: "Created user User { Id = 42, Name = Alice, ... }"

// ✅ @ destructure เป็น object — เก็บทุก property
_logger.LogInformation("Created user {@User}", user);
// structured: { User: { Id: 42, Name: "Alice", Email: "alice@..." } }
// → query ด้วย User.Id หรือ User.Name ได้
```

---

## 4. Log Scopes — เพิ่ม Context ให้ทุก Log ใน Block

```csharp
// ปัญหา: log หลายบรรทัดในหนึ่ง operation ไม่รู้ว่าเป็น request เดียวกัน
_logger.LogInformation("Processing order");
_logger.LogInformation("Validating items");
_logger.LogInformation("Charging payment");
// output เหมือนกัน ทั้งที่มาจาก request คนละตัว

// ✅ BeginScope — แปะ context ให้ทุก log ใน block นั้น
using (_logger.BeginScope("OrderId={OrderId}, UserId={UserId}", orderId, userId))
{
    _logger.LogInformation("Processing order");
    // output: info: [OrderId=999, UserId=42] Processing order

    _logger.LogInformation("Validating items");
    // output: info: [OrderId=999, UserId=42] Validating items

    _logger.LogInformation("Charging payment");
    // output: info: [OrderId=999, UserId=42] Charging payment
}
// scope หมดเมื่อออกจาก using block
```

เปิด scope ใน `appsettings.json`:
```json
{
  "Logging": {
    "Console": {
      "IncludeScopes": true
    }
  }
}
```

---

## 5. Correlation ID — ติดตาม Request ข้าม Services

ใน web API ที่มีหลาย request พร้อมกัน log ของ request ต่างๆ จะปะปนกัน Correlation ID แก้ปัญหานี้

```csharp
// CorrelationIdMiddleware.cs
public class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        // รับจาก header (ถ้ามี) หรือสร้างใหม่
        var correlationId = ctx.Request.Headers[HeaderName].FirstOrDefault()
                            ?? Guid.NewGuid().ToString();

        ctx.Items["CorrelationId"] = correlationId;

        // ส่ง correlationId กลับใน response header ด้วย
        ctx.Response.Headers[HeaderName] = correlationId;

        // ใส่ลงใน log scope — ทุก log ใน request นี้จะมี CorrelationId
        using (_logger.BeginScope("{CorrelationId}", correlationId))
        {
            await _next(ctx);
        }
    }

    private readonly ILogger<CorrelationIdMiddleware> _logger;
    public CorrelationIdMiddleware(RequestDelegate next,
        ILogger<CorrelationIdMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }
}
```

```csharp
// Program.cs — ลงทะเบียน middleware
app.UseMiddleware<CorrelationIdMiddleware>();  // ต้องอยู่ต้น pipeline
```

---

## 6. Logging ใน Controller และ Service

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService           _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orderService,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger       = logger;
    }

    [HttpPost]
    public async Task<ActionResult<Order>> Create(
        CreateOrderDto dto, CancellationToken ct)
    {
        // log ข้อมูล request (ระวัง sensitive data!)
        _logger.LogInformation(
            "Creating order for user {UserId}, items={ItemCount}",
            dto.UserId, dto.Items.Count);

        try
        {
            var order = await _orderService.CreateAsync(dto, ct);

            _logger.LogInformation(
                "Order {OrderId} created successfully", order.Id);

            return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
        }
        catch (InsufficientStockException ex)
        {
            // Warning — ไม่ใช่ bug แต่ควรรู้
            _logger.LogWarning(
                "Insufficient stock for product {ProductId}", ex.ProductId);
            return Conflict(new { error = ex.Message });
        }
        catch (Exception ex)
        {
            // Error — operation ล้มเหลว แนบ exception object ด้วย
            _logger.LogError(ex,
                "Failed to create order for user {UserId}", dto.UserId);
            throw;  // ปล่อยให้ exception handler จัดการต่อ
        }
    }
}
```

### สิ่งที่ไม่ควร Log

```csharp
// ❌ ห้าม log sensitive data
_logger.LogInformation("User logged in with password {Password}", password);
_logger.LogInformation("Processing card {CardNumber}", cardNumber);
_logger.LogDebug("JWT token: {Token}", jwtToken);

// ✅ log เฉพาะที่จำเป็น — mask ข้อมูลสำคัญ
_logger.LogInformation("User {UserId} logged in", userId);
_logger.LogInformation("Processing card ending in {Last4}", cardNumber[^4..]);
_logger.LogDebug("JWT issued for {UserId}, expires {Expiry}", userId, expiry);
```

---

## 7. Serilog — Logging ระดับ Production

Built-in logging ของ .NET ดีพอสำหรับ console output แต่ production ต้องการมากกว่านั้น เช่น เขียนลง file, ส่งไป Seq, Elasticsearch, หรือ Datadog

**Serilog** คือ library ยอดนิยมที่เพิ่ม:
- **Sinks** — output ไปได้หลายที่พร้อมกัน
- **Enrichers** — เพิ่ม context อัตโนมัติ (machine name, thread id, request path)
- **Rolling file** — แบ่งไฟล์ log ตามวัน

### Setup Serilog

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

```csharp
// Program.cs
using Serilog;

// สร้าง logger ก่อน Host เพื่อ log ตอน startup ด้วย
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning)
    .Enrich.FromLogContext()          // เพิ่ม context จาก BeginScope
    .Enrich.WithMachineName()        // เพิ่ม MachineName ทุก log
    .Enrich.WithThreadId()           // เพิ่ม ThreadId
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} " +
                        "{Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.log",
        rollingInterval: RollingInterval.Day,     // แยกไฟล์ทุกวัน
        retainedFileCountLimit: 30,               // เก็บ 30 วัน
        outputTemplate: "[{Timestamp:yyyy-MM-dd HH:mm:ss} {Level:u3}] " +
                        "{Message:lj} {Properties:j}{NewLine}{Exception}")
    .CreateLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // แทน default logging ด้วย Serilog
    builder.Host.UseSerilog();

    builder.Services.AddControllers();

    var app = builder.Build();

    // log HTTP requests อัตโนมัติ
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate =
            "{RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
    });

    app.MapControllers();
    await app.RunAsync();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application failed to start");
}
finally
{
    await Log.CloseAndFlushAsync();  // flush log ที่ค้างอยู่ก่อนปิด
}
```

### Serilog Output Format

```
// Console output:
[10:15:32 INF] GET /api/users/42 responded 200 in 12.3456ms
               { RequestId: "abc123", RequestPath: "/api/users/42" }

[10:15:33 INF] Getting user 42 { UserId: 42, CorrelationId: "xyz789" }

[10:15:33 WRN] Retry 2/3 for operation "GetUser"
               { Count: 2, Max: 3, Op: "GetUser" }
```

---

## 8. ตัวอย่างสมบูรณ์ — Logging ใน Web API

```csharp
// Models
public record Order(int Id, int UserId, decimal Amount, DateTime CreatedAt);
public record CreateOrderDto(int UserId, decimal Amount);

// Service
public interface IOrderService
{
    Task<Order> CreateAsync(CreateOrderDto dto, CancellationToken ct = default);
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
}

public class OrderService : IOrderService
{
    private static readonly List<Order> _orders = new();
    private static int _nextId = 1;
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Order> CreateAsync(CreateOrderDto dto, CancellationToken ct)
    {
        _logger.LogDebug("Validating order for user {UserId}", dto.UserId);

        await Task.Delay(50, ct);  // จำลอง DB call

        var order = new Order(_nextId++, dto.UserId, dto.Amount, DateTime.UtcNow);
        _orders.Add(order);

        _logger.LogInformation(
            "Order {OrderId} created for user {UserId}, amount={Amount}",
            order.Id, order.UserId, order.Amount);

        return order;
    }

    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct)
    {
        await Task.Delay(20, ct);
        var order = _orders.FirstOrDefault(o => o.Id == id);

        if (order is null)
            _logger.LogWarning("Order {OrderId} not found", id);

        return order;
    }
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService           _service;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService service, ILogger<OrdersController> logger)
    {
        _service = service;
        _logger  = logger;
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<Order>> GetById(int id, CancellationToken ct)
    {
        using var scope = _logger.BeginScope("OrderId={OrderId}", id);

        var order = await _service.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    public async Task<ActionResult<Order>> Create(
        CreateOrderDto dto, CancellationToken ct)
    {
        _logger.LogInformation(
            "Received order request: UserId={UserId}, Amount={Amount}",
            dto.UserId, dto.Amount);

        try
        {
            var order = await _service.CreateAsync(dto, ct);
            return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to create order for user {UserId}", dto.UserId);
            throw;
        }
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();
app.MapControllers();
await app.RunAsync();
```

ทดสอบ:
```bash
# สร้าง order
curl -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"amount":150.00}'

# ดู order
curl http://localhost:5000/api/orders/1

# order ที่ไม่มี — ดู Warning log
curl http://localhost:5000/api/orders/999
```

---

## บทสรุป

```
ILogger<T>:
  - T คือ category ระบุว่า log มาจากไหน
  - 6 levels: Trace Debug Information Warning Error Critical
  - ใช้ Message Template ไม่ใช่ string interpolation

Structured Logging:
  - {Property} เก็บข้อมูลเป็น field แยก — query ได้
  - {@Object} destructure object ลงทุก property

Log Scope:
  - BeginScope เพิ่ม context ให้ทุก log ใน block
  - ใช้ Correlation ID ติดตาม request ข้าม services

Serilog:
  - Sinks: Console, File, Seq, Elasticsearch, Datadog
  - Enrichers: MachineName, ThreadId, Environment
  - UseSerilogRequestLogging() log HTTP request อัตโนมัติ

Best Practices:
  ✅ Message Template แทน string interpolation
  ✅ Log ระดับที่เหมาะสม ไม่ verbose เกินใน production
  ✅ ใช้ scope เพื่อ group log ที่เกี่ยวข้องกัน
  ❌ ห้าม log password, token, card number
  ❌ ห้าม log ข้อมูลมากเกินจนเปลือง I/O
```

---

## Workshop

1. **Lab 8.1** — เพิ่ม logging ให้ `UsersController` จาก Part 6 โดยให้ log ทุก operation: GetAll, GetById, Create, Update, Delete พร้อมข้อมูลที่เหมาะสมในแต่ละ level

2. **Lab 8.2** — สร้าง `CorrelationIdMiddleware` ที่สมบูรณ์ แล้วทดสอบโดยยิง request พร้อมกัน 5 ตัว ดูว่า log แต่ละบรรทัดมี `CorrelationId` ที่ตรงกันหรือไม่

3. **Lab 8.3** — ติดตั้ง Serilog และตั้งค่าให้ write ทั้ง console และ file `logs/app-{date}.log` แล้วดูว่าไฟล์ถูกสร้างและมีข้อมูลครบ

4. **Lab 8.4 (Bonus)** — เพิ่ม log filter ให้แสดงเฉพาะ `Warning` ขึ้นไปจาก `Microsoft.AspNetCore` แต่แสดง `Debug` ขึ้นไปจาก namespace ของ app เอง แล้วสังเกตว่า noise ลดลงแค่ไหน

---

*→ บทถัดไป: **Part 9 — Validation** — Data Annotations, FluentValidation, Custom Validators และ ProblemDetails response format*
