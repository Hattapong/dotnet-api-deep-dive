# Part 5 — Dependency Injection Deep Dive

> **Deep Dive Series: .NET Web API Internals**  
> ทุกครั้งที่เขียน `services.AddScoped<T>()` หรือ inject parameter เข้า handler  
> มี DI Container ทำงานอยู่เบื้องหลัง — บทนี้จะเข้าใจมันจากรากฐาน

---

## 1. ทำไมต้อง DI?

### ปัญหาของ "new ทุกอย่างเอง"

สมมติเราสร้าง UserService ที่ต้องการ Repository และ Logger:

```csharp
// ❌ ไม่ใช้ DI — new() ทุกอย่างเอง
public class UserController
{
    public IActionResult GetUser(int id)
    {
        // สร้าง dependency chain ด้วยมือ
        var connection  = new SqlConnection("Server=...;Database=...");
        var repo        = new UserRepository(connection);
        var emailSender = new SmtpEmailSender("smtp.gmail.com", 587);
        var service     = new UserService(repo, emailSender);

        var user = service.GetById(id);
        return Ok(user);
    }
}
```

ปัญหาที่เกิดขึ้นทันที:

| ปัญหา | อาการ |
|-------|-------|
| **Tight Coupling** | `UserController` รู้จัก `SqlConnection`, `SmtpEmailSender` ทั้งที่ไม่ควร |
| **Hard to Test** | ทดสอบ `UserController` ต้องมี DB จริง, SMTP จริง |
| **Duplication** | ทุก controller ที่ต้องการ `UserService` ต้อง new() chain เดิมซ้ำ |
| **Lifetime ยุ่งยาก** | `SqlConnection` ควร reuse ต่อ request แต่ตอนนี้สร้างใหม่ทุกครั้ง |
| **Config กระจัดกระจาย** | connection string อยู่ทุกที่ แก้ที่เดียวไม่ได้ |

### DI แก้ปัญหาอย่างไร

แนวคิดหลักคือ **"อย่าสร้าง dependency เอง ให้บอกว่าต้องการอะไร แล้วให้คนอื่นเอามาให้"**

```csharp
// ✅ ใช้ DI — บอกแค่ว่าต้องการอะไร
public class UserController
{
    private readonly IUserService _service;

    // Constructor Injection — บอกว่าต้องการ IUserService
    public UserController(IUserService service)
    {
        _service = service;
    }

    public IActionResult GetUser(int id)
    {
        var user = _service.GetById(id);
        return Ok(user);
    }
}
// UserController ไม่รู้ว่า IUserService คือ implementation อะไร
// ไม่รู้ว่า SqlConnection อยู่ที่ไหน
// ทดสอบได้ด้วยการ inject MockUserService แทน
```

```csharp
// ลงทะเบียน mapping ที่ Program.cs ที่เดียว
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<IEmailSender, SmtpEmailSender>();
```

---

## 2. กลไกการทำงานของ DI Container

DI Container ทำงาน 2 phase:

```
Phase 1: Registration (ตอน startup)
  "บันทึกว่า ถ้าใครขอ IUserService ให้สร้าง UserService"

Phase 2: Resolution (ตอน runtime)
  "มีคนขอ IUserService → สร้าง UserService → inject dependencies ของมันต่อ"
```

### IServiceCollection vs IServiceProvider

```csharp
// IServiceCollection = สมุดบันทึก registrations (ก่อน Build)
IServiceCollection services = builder.Services;
services.AddScoped<IUserService, UserService>();
services.AddScoped<IUserRepository, UserRepository>();
// แค่บันทึก ยังไม่ได้สร้างอะไร

// IServiceProvider = container ที่ใช้งานจริง (หลัง Build)
IServiceProvider provider = builder.Build().Services;
var userService = provider.GetRequiredService<IUserService>();
// ตอนนี้ container สร้าง UserService + inject deps ให้
```

```
IServiceCollection          IServiceProvider
──────────────────          ────────────────
[ IUserService               สร้างตอน Build()
  → UserService    ]──────►  และใช้ resolve ตอน runtime
[ IUserRepository
  → UserRepository ]
[ IEmailSender
  → SmtpEmailSender]
```

### วิธีที่ Container Resolve — Recursive

เมื่อขอ `IUserService` container ทำดังนี้:

```
ขอ IUserService
  → ต้องสร้าง UserService
  → UserService constructor ต้องการ (IUserRepository, IEmailSender)
      → ต้องสร้าง UserRepository
      → UserRepository constructor ต้องการ (IConfiguration)
          → IConfiguration ลงทะเบียนไว้แล้ว → resolve ได้
      → UserRepository สร้างสำเร็จ ✓
      → ต้องสร้าง SmtpEmailSender
      → SmtpEmailSender constructor ต้องการ (IOptions<SmtpSettings>)
          → IOptions<SmtpSettings> ลงทะเบียนไว้แล้ว → resolve ได้
      → SmtpEmailSender สร้างสำเร็จ ✓
  → UserService สร้างสำเร็จ ✓
```

ทำแบบ recursive จนกว่าจะ resolve ได้ทุกตัว หรือ throw exception ถ้าหา registration ไม่เจอ

---

## 3. Service Lifetime — อายุของ Object

นี่คือหัวใจสำคัญที่ต้องเข้าใจให้ถูก

### Singleton — สร้างครั้งเดียว ใช้ตลอด app

```csharp
services.AddSingleton<IEmailSender, SmtpEmailSender>();
```

```
App start
    │
    ├── Request 1: ขอ IEmailSender → สร้าง SmtpEmailSender (instance A)
    │
    ├── Request 2: ขอ IEmailSender → คืน instance A (เดิม)
    │
    ├── Request 3: ขอ IEmailSender → คืน instance A (เดิม)
    │
App stop → instance A ถูก Dispose
```

เหมาะกับ: configuration, HTTP clients, caches, stateless utilities

### Scoped — สร้างใหม่ต่อ 1 scope (= 1 HTTP request)

```csharp
services.AddScoped<IUserRepository, UserRepository>();
```

```
Request 1 เข้า → สร้าง Scope 1
    │   ขอ IUserRepository → สร้าง UserRepository (instance B)
    │   ขอ IUserRepository อีกครั้ง → คืน instance B (เดิม ใน scope เดียวกัน)
Request 1 จบ → Scope 1 Dispose → instance B ถูก Dispose

Request 2 เข้า → สร้าง Scope 2
    │   ขอ IUserRepository → สร้าง UserRepository (instance C) ← ใหม่
Request 2 จบ → Scope 2 Dispose → instance C ถูก Dispose
```

เหมาะกับ: DbContext, Unit of Work, services ที่ควร share state ภายใน request เดียว

### Transient — สร้างใหม่ทุกครั้งที่ขอ

```csharp
services.AddTransient<IOrderCalculator, OrderCalculator>();
```

```
Request 1:
    ขอ IOrderCalculator ครั้งที่ 1 → สร้าง instance D
    ขอ IOrderCalculator ครั้งที่ 2 → สร้าง instance E (ใหม่เสมอ)

Request 2:
    ขอ IOrderCalculator ครั้งที่ 1 → สร้าง instance F (ใหม่เสมอ)
```

เหมาะกับ: lightweight stateless operations, validators

### เปรียบเทียบ 3 Lifetime

```csharp
// ทดสอบด้วยโค้ดนี้:
builder.Services.AddSingleton<SingletonService>();
builder.Services.AddScoped<ScopedService>();
builder.Services.AddTransient<TransientService>();

app.MapGet("/lifetime", (
    SingletonService  s1, SingletonService  s2,
    ScopedService     sc1, ScopedService    sc2,
    TransientService  t1, TransientService  t2) =>
{
    return Results.Ok(new
    {
        singleton_same  = ReferenceEquals(s1, s2),   // true เสมอ
        scoped_same     = ReferenceEquals(sc1, sc2),  // true (ใน request เดียว)
        transient_same  = ReferenceEquals(t1, t2),    // false เสมอ
    });
});
// ยิง 2 requests:
// Request 1: singleton id = 1, scoped id = 10, transient id = 100,101
// Request 2: singleton id = 1, scoped id = 11, transient id = 102,103
//            ↑ เหมือนเดิม     ↑ ใหม่            ↑ ใหม่ทุกครั้ง
```

---

## 4. Captive Dependency — Bug ที่พบบ่อยมาก

**Captive Dependency** เกิดเมื่อ service ที่มี lifetime ยาวกว่า inject service ที่มี lifetime สั้นกว่า

```
❌ ปัญหา:

Singleton (อยู่ตลอด app)
  └── inject → Scoped (ควรอยู่แค่ per-request)

ผล: Scoped service ถูก Singleton "จับ" ไว้
    ไม่ถูก Dispose ตอนจบ request
    ข้อมูลใน request ก่อนหน้า "รั่ว" เข้ามาใน request ถัดไป
```

```csharp
// ❌ Bug — Singleton inject Scoped
public class MySingletonService
{
    private readonly IUserRepository _repo;  // Scoped!

    public MySingletonService(IUserRepository repo)  // ← ตรงนี้คือปัญหา
    {
        _repo = repo;
    }
    // _repo จะถูก "lock" ไว้ใน singleton
    // ทุก request ใช้ repo instance เดิม แทนที่จะได้อันใหม่
}
```

ASP.NET Core detect ปัญหานี้ให้ในโหมด Development:

```
InvalidOperationException:
Cannot consume scoped service 'IUserRepository'
from singleton 'MySingletonService'.
```

วิธีแก้:

```csharp
// ✅ Option 1: เปลี่ยน lifetime ให้ตรงกัน
services.AddSingleton<IUserRepository, UserRepository>();  // ถ้าเหมาะสม

// ✅ Option 2: ใช้ IServiceScopeFactory สร้าง scope เอง
public class MySingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MySingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IUserRepository>();
        // ใช้ repo ภายใน scope นี้
        await repo.SaveChangesAsync();
    }  // scope Dispose → repo Dispose ✓
}
```

---

## 5. การ Register รูปแบบต่างๆ

```csharp
// ── รูปแบบพื้นฐาน ──────────────────────────────────────────
services.AddScoped<IUserService, UserService>();
// interface → implementation

services.AddSingleton<AppSettings>(new AppSettings { Title = "My API" });
// ส่ง instance ตรงๆ เลย

services.AddTransient<IValidator, UserValidator>();

// ── Factory Registration ────────────────────────────────────
// ใช้เมื่อต้องการ logic ในการสร้าง
services.AddScoped<IUserService>(provider =>
{
    var repo    = provider.GetRequiredService<IUserRepository>();
    var logger  = provider.GetRequiredService<ILogger<UserService>>();
    var feature = Environment.GetEnvironmentVariable("FEATURE_FLAG") == "true";

    return feature
        ? new EnhancedUserService(repo, logger)
        : new UserService(repo, logger);
});

// ── Register หลาย Implementation ────────────────────────────
services.AddScoped<INotificationSender, EmailSender>();
services.AddScoped<INotificationSender, SmsSender>();
services.AddScoped<INotificationSender, PushSender>();

// inject เป็น IEnumerable<T> เพื่อรับทั้งหมด
public class NotificationService
{
    private readonly IEnumerable<INotificationSender> _senders;
    public NotificationService(IEnumerable<INotificationSender> senders)
        => _senders = senders;

    public async Task SendAllAsync(string message)
    {
        foreach (var sender in _senders)
            await sender.SendAsync(message);
    }
}

// ── Keyed Services (.NET 8) ─────────────────────────────────
// ตั้งชื่อให้แต่ละ implementation
services.AddKeyedScoped<ICache, MemoryCache>("memory");
services.AddKeyedScoped<ICache, RedisCache>("redis");

// inject โดยระบุชื่อ
public class MyService([FromKeyedServices("redis")] ICache cache) { }
```

---

## 6. DI กับ HttpContext — Scoped Container ต่อ Request

ใน ASP.NET Core ทุก HTTP request จะได้รับ **Scoped Container** ของตัวเอง นี่คือเหตุผลที่ `Scoped` service ทำงานได้

```
App start → Root Container (Singleton scope)
               │
    ┌──────────┼──────────┐
    │          │          │
Request 1   Request 2   Request 3
  Scope        Scope       Scope
    │
    ├── IUserRepository (instance ของ request 1)
    ├── IOrderService   (instance ของ request 1)
    └── DbContext       (instance ของ request 1)
```

```csharp
// ดู scoped container ของ request ปัจจุบัน
app.MapGet("/scope-demo", (HttpContext ctx) =>
{
    // RequestServices คือ scoped container ของ request นี้
    var repo1 = ctx.RequestServices.GetRequiredService<IUserRepository>();
    var repo2 = ctx.RequestServices.GetRequiredService<IUserRepository>();

    // repo1 และ repo2 คือ instance เดียวกัน (scoped)
    Console.WriteLine(ReferenceEquals(repo1, repo2));  // true
});
```

---

## 7. Mini DI Container — เข้าใจด้วยการสร้างเอง

ดู code ง่ายๆ ที่แสดงหลักการทำงานของ DI container โดยไม่ต้องพึ่ง framework:

```csharp
public class MiniContainer
{
    // สมุดบันทึก: interface type → วิธีสร้าง
    private readonly Dictionary<Type, Func<MiniContainer, object>> _registrations = new();

    // ── Registration ──────────────────────────────────────────
    public void Register<TService, TImpl>() where TImpl : TService
    {
        _registrations[typeof(TService)] = container =>
        {
            // หา constructor แรกที่มี parameter มากสุด
            var implType    = typeof(TImpl);
            var constructor = implType.GetConstructors()
                                      .OrderByDescending(c => c.GetParameters().Length)
                                      .First();

            // resolve parameter แต่ละตัวแบบ recursive
            var parameters = constructor.GetParameters()
                                        .Select(p => container.Resolve(p.ParameterType))
                                        .ToArray();

            return constructor.Invoke(parameters);
        };
    }

    public void RegisterInstance<TService>(TService instance)
    {
        _registrations[typeof(TService)] = _ => instance!;
    }

    // ── Resolution ────────────────────────────────────────────
    public object Resolve(Type serviceType)
    {
        if (!_registrations.TryGetValue(serviceType, out var factory))
            throw new InvalidOperationException($"Service '{serviceType.Name}' not registered");

        return factory(this);  // recursive: factory อาจ Resolve type อื่นต่อ
    }

    public T Resolve<T>() => (T)Resolve(typeof(T));
}

// ── ทดสอบ ─────────────────────────────────────────────────────
public interface IGreeter { string Greet(string name); }
public class Greeter : IGreeter
{
    private readonly string _prefix;
    public Greeter(AppConfig config) => _prefix = config.Prefix;
    public string Greet(string name) => $"{_prefix}, {name}!";
}
public class AppConfig { public string Prefix { get; set; } = "Hello"; }

var container = new MiniContainer();
container.RegisterInstance(new AppConfig { Prefix = "Sawasdee" });
container.Register<IGreeter, Greeter>();

var greeter = container.Resolve<IGreeter>();
Console.WriteLine(greeter.Greet("Alice"));  // "Sawasdee, Alice!"
```

Container จริงของ .NET ทำสิ่งเดียวกันนี้แต่รองรับ lifetime, scope, generic types, validation, และ performance optimization เพิ่มเติม

---

## 8. Inject เข้า Minimal API — ASP.NET Core ทำให้อัตโนมัติ

```csharp
// Minimal API — inject ผ่าน parameter โดยตรง
// ASP.NET Core resolve จาก RequestServices ให้เอง

app.MapGet("/users/{id}", async (
    int                  id,        // ← จาก route (ไม่ใช่ DI)
    IUserService         service,   // ← จาก DI (Scoped)
    ILogger<Program>     logger,    // ← จาก DI (Singleton)
    CancellationToken    ct)        // ← จาก HttpContext
{
    logger.LogInformation("Getting user {Id}", id);
    var user = await service.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

// ASP.NET Core แยกแยะ parameter โดย:
// - Type อยู่ใน DI? → inject จาก container
// - Type เป็น route value? → bind จาก URL
// - Type เป็น [FromBody]? → deserialize จาก request body
// - Type เป็น HttpContext/CancellationToken? → inject จาก framework
```

---

## บทสรุป

```
DI Container คือ:
  Registration   →   สมุดบันทึก (IServiceCollection)
  Resolution     →   factory ที่ recursive สร้าง dependency tree
  Lifetime       →   นโยบายว่าจะ reuse หรือสร้างใหม่เมื่อไหร่

Lifetime 3 แบบ:
  Singleton  → 1 instance ตลอด app lifetime
  Scoped     → 1 instance ต่อ 1 HTTP request
  Transient  → new instance ทุกครั้งที่ขอ

กฎสำคัญ:
  Singleton ห้าม inject Scoped  → Captive Dependency bug
  Scoped inject Singleton ได้   → ไม่มีปัญหา (lifetime ยาวกว่า)
  ใช้ IServiceScopeFactory ถ้า Singleton ต้องการ Scoped service
```

---

## Workshop

1. **Lab 5.1** — สร้าง `ICounter` service ที่นับจำนวนครั้งที่ถูกเรียก register ทั้ง Singleton และ Scoped แล้วยิง 3 requests ดูว่าค่าต่างกันอย่างไร

2. **Lab 5.2** — จงทำให้เกิด Captive Dependency exception โดยตั้งใจ inject `Scoped` เข้า `Singleton` แล้วดู error message แล้วแก้โดยใช้ `IServiceScopeFactory`

3. **Lab 5.3** — Register `INotificationSender` 3 แบบ (Email, SMS, Push) แล้วสร้าง endpoint ที่ส่ง notification ผ่านทุกช่องทางพร้อมกัน

---

*→ บทถัดไป: **Part 6 — Middleware Internals & Use vs Run vs Map** — ลงไปดู ApplicationBuilder.Build() ทำงานยังไง และความแตกต่างของ Use, Run, Map ในระดับ code*
