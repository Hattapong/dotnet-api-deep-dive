# Part 4 — HttpContext, Pipeline และ Middleware

> **Deep Dive Series: .NET Web API Internals**  
> เราเข้าใจ C# features แล้ว — บทนี้จะดูว่าเมื่อ HTTP request เข้ามา 1 ครั้ง  
> ASP.NET Core ทำอะไรกับมัน ตั้งแต่ bytes ใน TCP จนถึง JSON response ที่ส่งกลับ

---

## จุดเริ่มต้น — Web API ง่ายๆ 1 Endpoint

เราจะใช้โปรแกรมเล็กๆ นี้เป็นหลักตลอดบท แล้วค่อยๆ ขยายให้เห็นว่าแต่ละชั้นทำงานยังไง

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app     = builder.Build();

app.MapGet("/greet/{name}", (string name) =>
{
    return Results.Ok(new { message = $"Hello, {name}!", time = DateTime.UtcNow });
});

await app.RunAsync();
```

รันแล้วยิง request:

```bash
curl http://localhost:5000/greet/Alice
# {"message":"Hello, Alice!","time":"2026-03-13T10:00:00Z"}
```

ดูเรียบง่ายมาก — แต่ระหว่างที่ request เดินทางเข้ามา จนถึงตอน response ออกไป ASP.NET Core ทำงานหลายชั้นมาก บทนี้จะแกะทีละชั้น

---

## ภาพรวม — ทุกอย่างที่เกิดขึ้นใน 1 Request

```
Browser / curl
      │
      │  GET /greet/Alice HTTP/1.1\r\n
      │  Host: localhost:5000\r\n
      │  \r\n
      │
      ▼ (TCP bytes)
┌─────────────────────────────────────────────────────────────────┐
│  Kestrel (Web Server)                                           │
│  - รับ TCP connection บน port 5000                              │
│  - parse bytes → HttpContext                                    │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HttpContext
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  Middleware Pipeline                                            │
│                                                                 │
│  ┌──────────────┐                                               │
│  │ Middleware 1 │  ExceptionHandler   (ดัก error ทั้งหมด)       │
│  └──────┬───────┘                                               │
│         │ next()                                                │
│  ┌──────▼───────┐                                               │
│  │ Middleware 2 │  Routing            (จับคู่ URL → endpoint)   │
│  └──────┬───────┘                                               │
│         │ next()                                                │
│  ┌──────▼───────┐                                               │
│  │ Middleware 3 │  Authorization      (ตรวจสอบสิทธิ์)           │
│  └──────┬───────┘                                               │
│         │ next()                                                │
│  ┌──────▼───────┐                                               │
│  │ Middleware 4 │  EndpointMiddleware (เรียก handler จริง)      │
│  └──────┬───────┘                                               │
│         │                                                       │
│    [ handler: (string name) => Results.Ok(...) ]                │
│         │                                                       │
│  ◄──────┘ response เดินกลับผ่าน middleware เดิม (reverse)      │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HttpContext (with response)
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  Kestrel                                                        │
│  - serialize HttpContext.Response → bytes                       │
│  - ส่งกลับผ่าน TCP                                             │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (TCP bytes)
      │  HTTP/1.1 200 OK\r\n
      │  Content-Type: application/json\r\n
      │  \r\n
      │  {"message":"Hello, Alice!","time":"..."}
      │
Browser / curl
```

มี 3 concept หลักที่ต้องเข้าใจ: **HttpContext**, **Pipeline**, และ **Middleware** — มาดูทีละอัน

---

## 1. HttpContext — "ตัวแทน" ของ Request ทั้งหมด

### HttpContext คืออะไร?

`HttpContext` คือ object ที่ ASP.NET Core สร้างขึ้น **1 ครั้งต่อ 1 request** เพื่อเก็บทุกอย่างที่เกี่ยวกับการสนทนาครั้งนั้น ทั้ง request ที่เข้ามา, response ที่จะส่งออก, ข้อมูล user, และ services ต่างๆ

```
HttpContext
├── Request          ← ทุกอย่างเกี่ยวกับ request ที่เข้ามา
│   ├── Method       "GET"
│   ├── Path         "/greet/Alice"
│   ├── Query        ?foo=bar
│   ├── Headers      Host, Accept, Authorization ...
│   ├── Body         (stream ของ request body)
│   └── ...
│
├── Response         ← ทุกอย่างเกี่ยวกับ response ที่จะส่งออก
│   ├── StatusCode   200, 404, 500 ...
│   ├── Headers      Content-Type, Location ...
│   ├── Body         (stream ที่ write ลงไปเป็น response body)
│   └── ...
│
├── User             ← identity ของ user (ถ้า authenticate แล้ว)
│   └── ClaimsPrincipal
│       ├── Identity.Name  "Alice"
│       ├── Claims         [{ "role": "admin" }, ...]
│       └── IsAuthenticated true/false
│
├── Items            ← Dictionary สำหรับแชร์ข้อมูลระหว่าง middleware
│
├── RequestServices  ← DI Container scoped to this request
│
├── Connection       ← ข้อมูล TCP connection
│   ├── RemoteIpAddress  "203.0.113.1"
│   ├── LocalPort        5000
│   └── Id               unique connection id
│
└── TraceIdentifier  ← unique id ของ request นี้ (สำหรับ logging)
```

### HttpContext.Request — ดู Request ทีละส่วน

```csharp
app.MapGet("/debug", (HttpContext ctx) =>
{
    var req = ctx.Request;

    return Results.Ok(new
    {
        // ─── Request Line ───────────────────────────────────
        method      = req.Method,          // "GET"
        scheme      = req.Scheme,          // "http" หรือ "https"
        host        = req.Host.Value,      // "localhost:5000"
        path        = req.Path.Value,      // "/debug"
        queryString = req.QueryString.Value, // "?foo=bar"

        // ─── Headers ────────────────────────────────────────
        contentType = req.ContentType,     // "application/json"
        userAgent   = req.Headers.UserAgent.ToString(),
        allHeaders  = req.Headers
                        .ToDictionary(h => h.Key, h => h.Value.ToString()),

        // ─── Route Values ────────────────────────────────────
        // (ถ้ามี route pattern เช่น /users/{id})
        routeValues = req.RouteValues
                        .ToDictionary(r => r.Key, r => r.Value?.ToString()),

        // ─── Body ────────────────────────────────────────────
        hasBody     = req.ContentLength > 0,
        bodyLength  = req.ContentLength,

        // ─── Connection ──────────────────────────────────────
        clientIp    = ctx.Connection.RemoteIpAddress?.ToString(),
        isHttps     = req.IsHttps,
    });
});
```

ยิง request เพื่อดูผล:

```bash
curl "http://localhost:5000/debug?foo=bar" \
  -H "X-Custom-Header: hello"
```

### HttpContext.Response — กำหนด Response

```csharp
app.MapGet("/manual-response", async (HttpContext ctx) =>
{
    var res = ctx.Response;

    // ─── Status Code ─────────────────────────────────────────
    res.StatusCode = 200;

    // ─── Headers ─────────────────────────────────────────────
    res.ContentType = "application/json; charset=utf-8";
    res.Headers["X-Request-Id"] = ctx.TraceIdentifier;
    res.Headers["X-Powered-By"] = "MyApp/1.0";

    // ⚠️ headers ต้อง set ก่อน write body
    //    หลัง write แล้ว header ส่งไปแล้ว แก้ไม่ได้

    // ─── Body ────────────────────────────────────────────────
    var json = """{"status":"ok"}""";
    await res.WriteAsync(json);  // write ลง stream ตรงๆ
});
```

> **กฎสำคัญ:** HTTP ส่ง status line + headers ก่อน แล้วค่อยส่ง body ดังนั้น เมื่อ `WriteAsync` ครั้งแรกถูกเรียก headers จะถูก "flush" ออกไปทันที และ **แก้ไขไม่ได้อีกแล้ว**

```csharp
// ❌ Error นี้เกิดขึ้นบ่อยมาก
app.MapGet("/wrong", async (HttpContext ctx) =>
{
    await ctx.Response.WriteAsync("Hello");  // headers ถูก flush แล้ว!
    ctx.Response.StatusCode = 404;           // 💥 InvalidOperationException
    // "Headers are read-only, response has already started."
});

// ✅ กำหนด status + headers ก่อนเสมอ
app.MapGet("/correct", async (HttpContext ctx) =>
{
    ctx.Response.StatusCode = 404;
    ctx.Response.ContentType = "application/json";
    await ctx.Response.WriteAsync("""{"error":"not found"}""");
});
```

### HttpContext.Items — กระเป๋าใส่ข้อมูลระหว่าง Middleware

```csharp
// Middleware A ใส่ข้อมูลลงไป
app.Use(async (ctx, next) =>
{
    ctx.Items["StartTime"] = DateTime.UtcNow;
    ctx.Items["RequestId"] = Guid.NewGuid().ToString();

    await next();
});

// Middleware B (หรือ handler) อ่านข้อมูลออกมาได้
app.MapGet("/info", (HttpContext ctx) =>
{
    var startTime = (DateTime)ctx.Items["StartTime"]!;
    var requestId = (string)ctx.Items["RequestId"]!;
    var elapsed   = DateTime.UtcNow - startTime;

    return Results.Ok(new { requestId, elapsedMs = elapsed.TotalMilliseconds });
});
```

`Items` มีชีวิตอยู่แค่ใน request เดียว เมื่อ request จบ ข้อมูลใน `Items` หายไปด้วย

### HttpContext.RequestServices — DI Container ระดับ Request

```csharp
app.MapGet("/from-di", (HttpContext ctx) =>
{
    // ดึง service จาก scoped container ของ request นี้
    var logger = ctx.RequestServices.GetRequiredService<ILogger<Program>>();
    var config = ctx.RequestServices.GetRequiredService<IConfiguration>();

    logger.LogInformation("Handling request {Id}", ctx.TraceIdentifier);

    return Results.Ok(new { traceId = ctx.TraceIdentifier });
});

// ปกติ inject ตรงๆ ดีกว่า (ASP.NET Core ทำให้)
app.MapGet("/from-di-clean", (ILogger<Program> logger, IConfiguration config) =>
{
    // ↑ ASP.NET Core resolve จาก RequestServices ให้อัตโนมัติ
    return Results.Ok("clean injection");
});
```

---

## 2. Middleware — "ด่านตรวจ" แต่ละชั้น

### Middleware คืออะไร?

**Middleware** คือ function ที่อยู่กลางทางระหว่าง Kestrel รับ request จนถึง handler ตอบ response — มันสามารถ:

- ดู / แก้ไข request ก่อนส่งต่อ
- ดู / แก้ไข response ก่อนส่งกลับ
- ตัดสินใจว่าจะส่งต่อหรือหยุดตรงนี้เลย (short-circuit)
- รัน code หลัง handler ตอบกลับแล้ว (เพราะ pipeline เดินสองทิศ)

### RequestDelegate — Type จริงๆ ของ Middleware

```csharp
// RequestDelegate คือ delegate ที่ middleware ทุกตัว share กัน
public delegate Task RequestDelegate(HttpContext context);

// Middleware คือ function ที่รับ "next" แล้ว return RequestDelegate ใหม่
Func<RequestDelegate, RequestDelegate> middleware =
    (RequestDelegate next) =>
        async (HttpContext context) =>
        {
            // ── โค้ดที่รันก่อน handler ──
            Console.WriteLine("BEFORE");

            await next(context);   // ส่งต่อไปยัง middleware ถัดไป

            // ── โค้ดที่รันหลัง handler ──
            Console.WriteLine("AFTER");
        };
```

รูปร่างนี้คือรากฐานของ middleware ทั้งหมดใน ASP.NET Core

---

## 3. Pipeline — สายพาน Middleware

### Pipeline สร้างอย่างไร?

เมื่อเราเขียน `app.Use(...)` เราไม่ได้ "รัน" middleware ทันที แต่กำลัง **ลงทะเบียน** เรียงลำดับว่าจะเชื่อมกันยังไง แล้ว `app.Build()` (ที่เกิดตอน `builder.Build()`) จะ **compile** มันทั้งหมดเป็น `RequestDelegate` เดียว

```csharp
// สมมติเรา register 3 middleware:
app.Use(middlewareA);
app.Use(middlewareB);
app.Use(middlewareC);
app.Run(terminalHandler);  // ไม่มี next
```

สิ่งที่ pipeline compiler ทำ (เบื้องหลัง):

```csharp
// Pipeline สร้าง chain แบบ inside-out:
// เริ่มจาก terminal handler (สุดท้าย) แล้วค่อยห่อออกมา

RequestDelegate terminal = terminalHandler;         // handler สุดท้าย
RequestDelegate c        = middlewareC(terminal);   // ห่อ terminal ด้วย C
RequestDelegate b        = middlewareB(c);          // ห่อ C ด้วย B
RequestDelegate a        = middlewareA(b);          // ห่อ B ด้วย A

// pipeline จริงๆ คือ:
RequestDelegate pipeline = a;  // เรียก A → B → C → terminal → C → B → A
```

เมื่อ request เข้ามา Kestrel เรียก `pipeline(httpContext)` ซึ่งก็คือเรียก `a(ctx)` ซึ่งข้างในเรียก `b(ctx)` ต่อไปเรื่อยๆ

### ภาพการเดินของ Request และ Response

```
Request เข้า ──────────────────────────────────────────────────►
                │           │           │           │
           Middleware A  Middleware B  Middleware C  Handler
                │           │           │           │
                │──before──►│──before──►│──before──►│
                │           │           │           │  execute
                │           │           │           │
                │◄──after───│◄──after───│◄──after───│
                │           │           │           │
◄────────────────────────────────────────────────────────────────
Response ออก
```

เพราะ `await next(ctx)` คือจุดที่ "ลง" ไปข้างล่าง โค้ดก่อน `await next` = รันตอน request เข้า, โค้ดหลัง `await next` = รันตอน response ออก

---

## 4. ดู Pipeline จริงๆ ใน Action

มาแปลง web app ของเราให้เห็น pipeline ทำงานชัดๆ:

```csharp
// Program.cs — pipeline แบบ verbose เพื่อการเรียนรู้
var builder = WebApplication.CreateBuilder(args);
var app     = builder.Build();

// ─── Middleware 1: Request Logger ─────────────────────────────
app.Use(async (ctx, next) =>
{
    var watch = System.Diagnostics.Stopwatch.StartNew();
    Console.WriteLine($"[M1 ↓] {ctx.Request.Method} {ctx.Request.Path}");

    await next();   // ส่งต่อ

    watch.Stop();
    Console.WriteLine($"[M1 ↑] {ctx.Response.StatusCode} ({watch.ElapsedMilliseconds}ms)");
});

// ─── Middleware 2: Request ID ─────────────────────────────────
app.Use(async (ctx, next) =>
{
    var requestId = Guid.NewGuid().ToString()[..8];
    ctx.Items["RequestId"] = requestId;

    Console.WriteLine($"[M2 ↓] Assigned RequestId: {requestId}");

    await next();   // ส่งต่อ

    // ใส่ request id กลับลงใน response header
    ctx.Response.Headers["X-Request-Id"] = requestId;
    Console.WriteLine($"[M2 ↑] Header X-Request-Id set");
});

// ─── Middleware 3: Fake Auth Check ───────────────────────────
app.Use(async (ctx, next) =>
{
    Console.WriteLine($"[M3 ↓] Checking auth...");
    var token = ctx.Request.Headers.Authorization.ToString();

    if (string.IsNullOrEmpty(token))
    {
        // short-circuit — ไม่เรียก next()
        Console.WriteLine($"[M3 ✗] No token → 401");
        ctx.Response.StatusCode = 401;
        await ctx.Response.WriteAsync("""{"error":"Unauthorized"}""");
        return;  // ← ไม่ไปต่อ, response เดินกลับผ่าน M2 → M1 แล้วออก
    }

    Console.WriteLine($"[M3 ✓] Token found, passing through");
    await next();   // ส่งต่อ
    Console.WriteLine($"[M3 ↑] After handler");
});

// ─── Handler ─────────────────────────────────────────────────
app.MapGet("/greet/{name}", (string name, HttpContext ctx) =>
{
    var requestId = ctx.Items["RequestId"] as string;
    Console.WriteLine($"[Handler] Executing for '{name}' (reqId: {requestId})");

    return Results.Ok(new
    {
        message   = $"Hello, {name}!",
        requestId,
        time      = DateTime.UtcNow
    });
});

await app.RunAsync();
```

### ทดสอบ — Request ที่ไม่มี Token

```bash
curl http://localhost:5000/greet/Alice
```

Console output:
```
[M1 ↓] GET /greet/Alice
[M2 ↓] Assigned RequestId: a1b2c3d4
[M3 ↓] Checking auth...
[M3 ✗] No token → 401
[M2 ↑] Header X-Request-Id set
[M1 ↑] 401 (2ms)
```

สังเกตว่า handler ไม่ถูกเรียก และ M2, M1 ยังรันโค้ด "after" ของตัวเองเพราะ M3 เรียก `return` หลัง write response ไม่ใช่ throw exception

### ทดสอบ — Request ที่มี Token

```bash
curl http://localhost:5000/greet/Alice \
  -H "Authorization: Bearer my-token"
```

Console output:
```
[M1 ↓] GET /greet/Alice
[M2 ↓] Assigned RequestId: e5f6g7h8
[M3 ↓] Checking auth...
[M3 ✓] Token found, passing through
[Handler] Executing for 'Alice' (reqId: e5f6g7h8)
[M3 ↑] After handler
[M2 ↑] Header X-Request-Id set
[M1 ↑] 200 (5ms)
```

เส้นทางเต็ม: M1↓ → M2↓ → M3↓ → Handler → M3↑ → M2↑ → M1↑

---

## 5. Built-in Middleware ที่ใช้งานจริง

ใน production web app middleware ที่เราลงทะเบียนไว้ข้างบน `app.UseXxx()` ทุกตัวคือสิ่งที่เราเพิ่งเรียนมา:

```csharp
var app = builder.Build();

// ─── ลำดับ Middleware มีความสำคัญมาก ───────────────────────────

app.UseExceptionHandler("/error");
// ห่อ pipeline ทั้งหมดด้วย try/catch
// ถ้า middleware ใดโยน exception จะถูกจับตรงนี้
// แล้ว redirect ไป /error endpoint

app.UseHttpsRedirection();
// ตรวจว่า request เป็น HTTP ไหม ถ้าใช่ redirect → HTTPS (301)
// ต้องอยู่ต้นๆ ก่อน middleware อื่นอ่าน body

app.UseStaticFiles();
// เช็คว่า path ตรงกับไฟล์ใน wwwroot ไหม
// ถ้าใช่ → serve file แล้ว short-circuit (ไม่ไปถึง routing)
// เช่น /favicon.ico, /css/app.css

app.UseRouting();
// อ่าน request path แล้วค้นหา endpoint ที่ match
// เก็บผล match ไว้ใน HttpContext.Features
// ยัง ไม่ ได้ execute handler

app.UseAuthentication();
// อ่าน token/cookie จาก request
// ถอดรหัส / verify
// ถ้า valid → set HttpContext.User (ClaimsPrincipal)
// ถ้าไม่ valid → ปล่อยผ่านแต่ User จะเป็น anonymous

app.UseAuthorization();
// ตรวจ HttpContext.User กับ policy ของ endpoint
// ถ้า endpoint มี [Authorize] แต่ User ไม่มีสิทธิ์ → 401/403
// ต้องมาหลัง UseAuthentication เสมอ

app.MapControllers();
// ลงทะเบียน endpoints จาก Controllers
// นี่คือ EndpointMiddleware ที่จะเรียก action method จริงๆ
```

### ทำไมลำดับถึงสำคัญ?

```
❌ ถ้าสลับ UseAuthentication กับ UseAuthorization:

UseAuthorization()   ← ตรวจ User... แต่ User ยังไม่ถูก set!
UseAuthentication()  ← set User... แต่สายเกินไปแล้ว

ผล: endpoint ที่ต้องการ [Authorize] จะ return 401 ทุกครั้ง
    แม้ส่ง token ที่ถูกต้องมา

❌ ถ้าใส่ UseStaticFiles หลัง UseRouting:

UseRouting()      ← ค้นหา route... /favicon.ico match กับ route ที่ register ไว้ไหม?
                     ถ้าไม่ match → 404  (ไฟล์ไม่ถูกเสิร์ฟ!)
UseStaticFiles()  ← ไม่ได้รับโอกาสทำงาน

ผล: static files ไม่ทำงาน
```

---

## 6. HttpContext.Features — ชั้นล่างสุดที่ Middleware คุยกัน

`HttpContext.Features` คือ collection ของ low-level interfaces ที่ middleware แต่ละตัวใช้แชร์ข้อมูลกัน เป็น "สาย backbone" ที่ทำให้ pipeline ทำงาน

```csharp
// ตัวอย่าง Features ที่สำคัญ:

// IRoutingFeature — เก็บผลการ match route
var routeFeature = ctx.Features.Get<IRoutingFeature>();
var routeData    = routeFeature?.RouteData;

// IEndpointFeature — เก็บ endpoint ที่ถูก select
var endpointFeature = ctx.Features.Get<IEndpointFeature>();
var endpoint        = endpointFeature?.Endpoint;
Console.WriteLine(endpoint?.DisplayName);  // "HTTP: GET /greet/{name}"

// IHttpConnectionFeature — ข้อมูล connection
var connFeature = ctx.Features.Get<IHttpConnectionFeature>();
Console.WriteLine(connFeature?.RemoteIpAddress);  // IP ของ client

// IHttpRequestFeature — raw request ระดับ low level
var reqFeature = ctx.Features.Get<IHttpRequestFeature>();
Console.WriteLine(reqFeature?.RawTarget);  // "/greet/Alice" raw จาก TCP

// IHttpResponseFeature — intercept response
var resFeature = ctx.Features.Get<IHttpResponseFeature>();
resFeature?.OnCompleted(state =>
{
    Console.WriteLine("Response fully sent!");
    return Task.CompletedTask;
}, null!);
```

`HttpContext.Request` และ `HttpContext.Response` ที่เราใช้ทุกวันคือ facade ที่ wrap `Features` เหล่านี้ไว้ — เพื่อ API ที่ใช้งานง่ายกว่า

---

## 7. เชื่อมทุกอย่างเข้าด้วยกัน

มาดู flow ของ request จาก web app ต้นบทอีกครั้ง แต่คราวนี้เราเห็นชัดว่าแต่ละจุดคืออะไร:

```
GET /greet/Alice
Authorization: Bearer my-token

   │
   │ TCP bytes ──────────────────────────────────────────────────
   ▼
┌──────────────────────────────────────────────────────────────────┐
│ Kestrel                                                          │
│  parse bytes → สร้าง DefaultHttpContext                         │
│  context.Request.Method = "GET"                                  │
│  context.Request.Path   = "/greet/Alice"                         │
│  context.Request.Headers["Authorization"] = "Bearer my-token"   │
│  เรียก pipeline(context)                                         │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Middleware: ExceptionHandler  [before]                           │
│  try { await next(ctx); }                                        │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Middleware: Routing  [before]                                     │
│  path "/greet/Alice" → match pattern "/greet/{name}"            │
│  ctx.Features.Set(IRoutingFeature)                               │
│  ctx.Request.RouteValues["name"] = "Alice"                       │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Middleware: Authentication  [before]                             │
│  อ่าน Authorization header → "Bearer my-token"                  │
│  verify token → valid                                            │
│  ctx.User = new ClaimsPrincipal(identity)                        │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Middleware: Authorization  [before]                              │
│  ตรวจ endpoint policy ← ไม่มี [Authorize] บน handler นี้        │
│  ผ่านไปได้                                                       │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ EndpointMiddleware  [execute]                                    │
│  ดึง endpoint จาก IEndpointFeature                               │
│  bind parameters: name = "Alice" (จาก RouteValues)              │
│  inject: HttpContext                                             │
│  เรียก handler: (string name, HttpContext ctx) => ...            │
│  handler สร้าง IResult                                           │
│  IResult.ExecuteAsync(ctx) → write JSON ลง Response.Body        │
│  ctx.Response.StatusCode = 200                                   │
└───────────────────────────┬──────────────────────────────────────┘
                            │ (เดินกลับ reverse)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Middleware: Authorization  [after]  (ส่วนใหญ่ไม่ทำอะไร)          │
│ Middleware: Authentication  [after]  (ส่วนใหญ่ไม่ทำอะไร)         │
│ Middleware: Routing  [after]  (ส่วนใหญ่ไม่ทำอะไร)                │
│ Middleware: ExceptionHandler  [after]  } catch { handle error }  │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ Kestrel                                                          │
│  Response.StatusCode = 200, Headers flushed                      │
│  Body stream → serialize เป็น bytes → ส่งออก TCP                │
└──────────────────────────────────────────────────────────────────┘
```

---

## บทสรุป

| Concept | คืออะไร | อยู่ที่ไหน |
|---|---|---|
| **HttpContext** | Object กลางที่เก็บทุกอย่างของ request/response | สร้างโดย Kestrel, ส่งผ่านทุก middleware |
| **HttpContext.Request** | ข้อมูล request ที่เข้ามา (method, path, headers, body) | อ่านได้ตลอด, ห้ามแก้ method/path |
| **HttpContext.Response** | ข้อมูล response ที่จะส่งออก (status, headers, body) | ต้อง set headers ก่อน write body |
| **HttpContext.Items** | กระเป๋าแชร์ข้อมูลระหว่าง middleware ใน request เดียว | ใช้ Items["key"] |
| **HttpContext.Features** | Low-level interfaces ที่ middleware คุยกัน | ส่วนใหญ่ไม่ต้องใช้ตรงๆ |
| **Middleware** | Function ที่นั่งกลางระหว่าง Kestrel กับ Handler | `app.Use(async (ctx, next) => {...})` |
| **Pipeline** | Chain ของ middleware ที่ถูก compile เป็น RequestDelegate เดียว | สร้างตอน `builder.Build()` |
| **Short-circuit** | Middleware ที่ไม่เรียก next() — หยุดไว้ตรงนั้นเลย | Auth, cache, rate limit |

---

## Workshop

1. **Lab 4.1** — เพิ่ม middleware ที่จับเวลา request และ log ออกมาในรูปแบบ `[GET /path] 200 OK (15ms)` พร้อม request id ที่ไม่ซ้ำกัน

2. **Lab 4.2** — สร้าง middleware ที่อ่าน header `X-Api-Version` แล้วเก็บลงใน `ctx.Items` จากนั้นใน handler อ่านค่านั้นออกมาแสดงใน response

3. **Lab 4.3** — ทดลองสลับลำดับ middleware แล้วดูว่าผลต่างกันอย่างไร เช่น ย้าย "fake auth" ไปไว้หลัง handler ทำงาน (แล้วดู console ว่า handler ยังถูกเรียกไหม)

4. **Lab 4.4 (Bonus)** — เขียน middleware ที่ดักจับ exception ทั้งหมดใน pipeline แล้ว return JSON `{"error":"message"}` พร้อม status 500 แทนที่จะ crash โดยใช้ try/catch ครอบ `await next()`

---

*→ บทถัดไป: **Part 5 — Middleware Internals** — เราจะลงไปดู ApplicationBuilder.Build() ทำงานยังไง, Use vs Run vs Map ต่างกันอย่างไร และสร้าง middleware pipeline เองตั้งแต่ต้น*
