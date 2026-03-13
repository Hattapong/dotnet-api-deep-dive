# Part 3 — C# Features สำหรับ Web API

> **Deep Dive Series: .NET Web API Internals**  
> ก่อนที่จะเข้าใจว่า ASP.NET Core ทำ magic ได้ยังไง  
> เราต้องเข้าใจ C# features ที่มันใช้ก่อน — เพราะ framework ทั้งหมดสร้างจากสิ่งเหล่านี้

---

## ภาพรวม — ทำไม Features เหล่านี้ถึงสำคัญ?

```
สิ่งที่คุณเขียนใน ASP.NET Core:         สิ่งที่อยู่ข้างใต้:
─────────────────────────────────────    ──────────────────────────────────────
app.UseAuthentication()              ←   Extension Method
services.AddScoped<T, TImpl>()       ←   Generic + Reflection
app.MapGet("/users/{id}", ...)       ←   Delegate / Lambda
[HttpGet], [Authorize]               ←   Attribute + Reflection
await client.GetAsync(url)           ←   Async/Await + Task
builder.Services.AddControllers()   ←   Extension Method + Reflection
```

ทุก API ที่คุณเคยใช้ใน .NET มี C# feature เหล่านี้อยู่เบื้องหลัง ถ้าเข้าใจ feature พวกนี้จะอ่าน source code ของ framework ได้ทันที

---

## 1. Generic — "Type เป็น Parameter"

### ปัญหาที่ Generic แก้

```csharp
// ❌ ก่อนมี Generic — ต้องเขียน method ซ้ำสำหรับทุก type
static int[]    Filter(int[]    items, int    target) { ... }
static string[] Filter(string[] items, string target) { ... }
static User[]   Filter(User[]   items, User   target) { ... }
// ถ้ามี 10 types ต้องเขียน 10 method

// หรือใช้ object แทน — แต่สูญเสีย type safety
static object[] Filter(object[] items, object target) { ... }
// compile ผ่าน แต่ runtime error ได้ทุกเมื่อ
```

```csharp
// ✅ Generic — เขียนครั้งเดียว ใช้ได้กับทุก type
static T[] Filter<T>(T[] items, T target)
{
    return items.Where(x => x!.Equals(target)).ToArray();
}

// ใช้ได้กับทุก type โดยไม่ต้อง cast
int[]    numbers = Filter(new[] { 1, 2, 3, 2 }, 2);    // [2, 2]
string[] names   = Filter(new[] { "a", "b", "a" }, "a"); // ["a", "a"]
```

### Generic ใน .NET ที่ใช้บ่อยมาก

```csharp
// Collection types
List<User>              users    = new();
Dictionary<string, int> map      = new();
IEnumerable<Product>    products = new List<Product>();
IReadOnlyList<Order>    orders   = Array.Empty<Order>();

// Task<T> — ผลลัพธ์ async
Task<User>         getUser    = GetUserAsync(id);
Task<List<string>> getItems   = GetItemsAsync();
Task<IActionResult> action    = HandleRequestAsync();

// Nullable<T> — alias ของ T?
Nullable<int> maybeId  = null;   // หรือเขียนว่า int?
int?          userId   = null;
```

### Generic Constraints — จำกัดว่า T ต้องเป็นอะไร

```csharp
// where T : class    → T ต้องเป็น reference type
// where T : struct   → T ต้องเป็น value type
// where T : new()    → T ต้องมี parameterless constructor
// where T : IEntity  → T ต้อง implement IEntity

public class Repository<T> where T : class, IEntity, new()
{
    private readonly List<T> _store = new();

    public void Add(T entity) => _store.Add(entity);

    public T? FindById(int id) => _store.FirstOrDefault(e => e.Id == id);

    // เพราะมี where T : new() จึง new T() ได้
    public T CreateEmpty() => new T();
}

// Generic ใน ASP.NET Core ที่เห็นบ่อย:
services.AddScoped<IUserService, UserService>();
//                 ↑ interface   ↑ implementation
// signature จริงๆ: AddScoped<TService, TImplementation>()

app.Services.GetRequiredService<IConfiguration>();
//                               ↑ บอก container ว่าต้องการ type อะไร

IOptions<MySettings> opts = serviceProvider.GetRequiredService<IOptions<MySettings>>();
```

### Generic Method ใน Extension Methods (เดี๋ยวจะอธิบาย)

```csharp
// นี่คือ pattern ที่ ASP.NET Core ใช้ตลอด
public static IServiceCollection AddScoped<TService, TImplementation>(
    this IServiceCollection services)
    where TService : class
    where TImplementation : class, TService
{
    services.AddScoped(typeof(TService), typeof(TImplementation));
    return services;
}
```

---

## 2. Extension Method — "เพิ่ม Method ให้ Type ที่แก้ไม่ได้"

### ปัญหาที่ Extension Method แก้

```csharp
// สมมติต้องการ method IsNullOrEmpty บน string
// แต่ string เป็น sealed class ใน .NET — แก้ไม่ได้

// ❌ วิธีเก่า — static helper method ที่น่าเกลียด
StringHelper.IsNullOrEmpty(myString);
Validator.IsValidEmail(email);
CollectionUtils.IsNullOrEmpty(list);
// ต้องจำชื่อ Helper class ทุกตัว

// ✅ Extension Method — เรียกเหมือน method ของ object เลย
myString.IsNullOrEmpty();  // เรียบร้อย อ่านง่าย
email.IsValidEmail();
list.IsNullOrEmpty();
```

### วิธีสร้าง Extension Method

```csharp
// กฎ: ต้องอยู่ใน static class, method ต้อง static, parameter แรกต้องมี this
public static class StringExtensions
{
    // this string value  ← "this" บอกว่า extend type string
    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);

    public static bool IsValidEmail(this string value)
        => value.Contains('@') && value.Contains('.');

    public static string Truncate(this string value, int maxLength)
        => value.Length <= maxLength ? value : value[..maxLength] + "...";
}

// ใช้งาน — เรียกเหมือน method ปกติ
string email = "user@example.com";
bool valid   = email.IsValidEmail();     // true
string short = "Hello World".Truncate(5); // "Hello..."

// สิ่งที่ compiler ทำจริงๆ (เบื้องหลัง):
bool valid2  = StringExtensions.IsValidEmail(email);  // เหมือนกันทุกอย่าง
```

### Extension Method ใน ASP.NET Core — ทั้งหมดคือสิ่งนี้

```csharp
// ทุก .UseXxx() และ .AddXxx() คือ extension method ทั้งนั้น

// Microsoft.AspNetCore.Builder
public static IApplicationBuilder UseAuthentication(this IApplicationBuilder app)
{
    return app.UseMiddleware<AuthenticationMiddleware>();
}

public static IApplicationBuilder UseAuthorization(this IApplicationBuilder app)
{
    return app.UseMiddleware<AuthorizationMiddleware>();
}

// Microsoft.Extensions.DependencyInjection
public static IServiceCollection AddControllers(this IServiceCollection services)
{
    services.AddMvcCore();
    services.AddApiExplorer();
    services.AddCors();
    // ... อีกเยอะ
    return services;
}

// Microsoft.AspNetCore.Routing
public static RouteHandlerBuilder MapGet(
    this IEndpointRouteBuilder endpoints,
    string pattern,
    Delegate handler)
{ ... }
```

```csharp
// ดังนั้นโค้ดนี้:
app.UseAuthentication();
app.UseAuthorization();
builder.Services.AddControllers();

// ความจริงคือ:
ApplicationBuilderExtensions.UseAuthentication(app);
ApplicationBuilderExtensions.UseAuthorization(app);
ServiceCollectionExtensions.AddControllers(builder.Services);
```

### Builder Pattern ด้วย Extension Method

Extension method ที่ return `this` ทำให้ chain ต่อกันได้:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddUserModule(this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IEmailService, SmtpEmailService>();
        return services;  // ← return ตัวเอง เพื่อ chain ต่อได้
    }

    public static IServiceCollection AddProductModule(this IServiceCollection services)
    {
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IProductService, ProductService>();
        return services;
    }
}

// ใช้งาน — chain ได้สวยงาม
builder.Services
    .AddUserModule()
    .AddProductModule()
    .AddControllers()
    .AddEndpointsApiExplorer();
```

---

## 3. Delegate & Lambda — "Function เป็น Object"

### Delegate คืออะไร?

**Delegate** คือ type ที่แทน "ลายเซ็น" ของ function — ทำให้ส่ง function เป็น parameter, เก็บใน variable, หรือ return ออกจาก method ได้

```csharp
// นิยาม delegate type — บอกว่า function ที่รับได้ต้องหน้าตาแบบนี้
delegate int  MathOperation(int a, int b);
delegate bool Predicate<T>(T item);
delegate void Action();

// สร้าง method ที่ match signature
int Add(int a, int b)      => a + b;
int Multiply(int a, int b) => a * b;

// ใส่ method ลงใน delegate variable
MathOperation op = Add;
Console.WriteLine(op(3, 4));  // 7

op = Multiply;
Console.WriteLine(op(3, 4));  // 12

// ส่ง function เป็น parameter
static int Calculate(int x, int y, MathOperation operation)
    => operation(x, y);

Calculate(3, 4, Add);       // 7
Calculate(3, 4, Multiply);  // 12
```

### Built-in Delegate Types — ไม่ต้องนิยามเอง

```csharp
// .NET มี delegate types สำเร็จรูปให้ใช้:

// Func<TResult>                 — ไม่รับ param, return TResult
// Func<T, TResult>              — รับ T, return TResult
// Func<T1, T2, TResult>         — รับ 2 params, return TResult
// ... ไปจนถึง Func<T1..T16, TResult>

Func<int>           getNumber  = () => 42;
Func<int, string>   toString   = n => n.ToString();
Func<int, int, int> add        = (a, b) => a + b;

// Action<T>  — รับ T, ไม่ return (void)
Action          sayHello  = () => Console.WriteLine("Hello");
Action<string>  sayTo     = name => Console.WriteLine($"Hello, {name}");
Action<int,int> printSum  = (a, b) => Console.WriteLine(a + b);

// Predicate<T> — รับ T, return bool (เฉพาะใช้กับ filtering)
Predicate<int>    isEven  = n => n % 2 == 0;
Predicate<string> hasAt   = s => s.Contains('@');
```

### Lambda Expression — Shorthand สำหรับ Anonymous Function

```csharp
// Lambda syntax:
// (parameters) => expression
// (parameters) => { statements; }

// ก่อนมี lambda — ต้องเขียน method แยก
int[] numbers = { 1, 2, 3, 4, 5, 6 };

static bool IsEven(int n) => n % 2 == 0;  // แยก method

var evens = numbers.Where(IsEven).ToArray();

// หลังมี lambda — เขียนตรงเลย
var evens2 = numbers.Where(n => n % 2 == 0).ToArray();
//                          ↑ anonymous function ที่ไม่ต้องตั้งชื่อ
```

### Delegate + Lambda ใน ASP.NET Core

ทุกครั้งที่เห็น `=>` ใน ASP.NET Core นั่นคือ lambda (delegate) ทั้งนั้น:

```csharp
// Minimal API — handler เป็น delegate
app.MapGet("/users", () => userService.GetAll());
//                    ↑ Func<IResult>

app.MapGet("/users/{id}", (int id) => userService.GetById(id));
//                         ↑ Func<int, IResult>

app.MapPost("/users", async (UserDto dto, IUserService svc) =>
{
    var user = await svc.CreateAsync(dto);
    return Results.Created($"/users/{user.Id}", user);
});
// ↑ Func<UserDto, IUserService, Task<IResult>>

// Middleware
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next();              // next เป็น Func<Task>
    Console.WriteLine("After");
});

// app.Use signature จริงๆ:
// public static IApplicationBuilder Use(
//     this IApplicationBuilder app,
//     Func<HttpContext, Func<Task>, Task> middleware)
```

```csharp
// LINQ ทั้งหมดใช้ delegate:
var users = dbContext.Users
    .Where(u => u.IsActive)               // Func<User, bool>
    .OrderBy(u => u.CreatedAt)            // Func<User, DateTime>
    .Select(u => new UserDto(u.Id, u.Name)) // Func<User, UserDto>
    .ToList();

// ConfigureServices — Action<T>
builder.Services.Configure<JwtOptions>(options =>
{
    options.Issuer   = "my-app";          // Action<JwtOptions>
    options.Audience = "my-clients";
});
```

### Delegate เป็นรากฐานของ Middleware Pipeline

```csharp
// RequestDelegate คือ delegate type หลักของ ASP.NET Core
public delegate Task RequestDelegate(HttpContext context);

// Middleware แต่ละตัวคือ function ที่รับ RequestDelegate แล้ว return RequestDelegate ใหม่
Func<RequestDelegate, RequestDelegate> middleware =
    next => async context =>
    {
        Console.WriteLine("Before");
        await next(context);   // เรียก middleware ถัดไป
        Console.WriteLine("After");
    };

// นี่คือสิ่งที่ app.Use() ทำจริงๆ (เราจะลงลึกใน Part 5)
```

---

## 4. Reflection & Attribute — "โค้ดที่อ่านตัวเอง"

### Reflection คืออะไร?

**Reflection** คือความสามารถของโปรแกรมในการ **ตรวจสอบและใช้งาน type ในระหว่าง runtime** โดยไม่รู้ type ณ compile time

```csharp
// ปกติ — รู้ type ตอน compile
User user = new User { Name = "Alice" };
string name = user.Name;  // compiler รู้ว่า User มี property Name

// Reflection — ค้นหา type ตอน runtime
Type type = typeof(User);
// หรือ
Type type2 = user.GetType();

// ดู properties ทั้งหมด
PropertyInfo[] props = type.GetProperties();
foreach (var prop in props)
    Console.WriteLine($"{prop.Name}: {prop.GetValue(user)}");
// output: Name: Alice
//         Email: (null)
//         Id: 0

// อ่าน/เขียน property ตามชื่อ (string)
PropertyInfo nameProp = type.GetProperty("Name")!;
nameProp.SetValue(user, "Bob");
Console.WriteLine(user.Name);  // Bob

// เรียก method ตามชื่อ (string)
MethodInfo method = type.GetMethod("GetDisplayName")!;
string result = (string)method.Invoke(user, null)!;
```

### Attribute — "Metadata บนโค้ด"

**Attribute** คือ tag ที่แปะบน class, method, property เพื่อเพิ่ม metadata ที่ Reflection อ่านได้ตอน runtime

```csharp
// สร้าง Attribute เอง — ต้อง inherit จาก Attribute
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class RequirePermissionAttribute : Attribute
{
    public string Permission { get; }

    public RequirePermissionAttribute(string permission)
    {
        Permission = permission;
    }
}

// แปะบน method/class
[RequirePermission("admin:users:read")]
public IActionResult GetAllUsers() { ... }

[RequirePermission("admin:users:write")]
public IActionResult CreateUser(UserDto dto) { ... }
```

```csharp
// อ่าน Attribute ด้วย Reflection
var method     = typeof(UsersController).GetMethod("GetAllUsers")!;
var attribute  = method.GetCustomAttribute<RequirePermissionAttribute>();

if (attribute != null)
{
    Console.WriteLine(attribute.Permission);  // "admin:users:read"
    // ใช้ตรวจสอบ permission ก่อน execute method
}
```

### Attribute ที่ใช้บ่อยใน ASP.NET Core

```csharp
// Routing
[Route("api/[controller]")]       // กำหนด route prefix
[HttpGet("{id}")]                  // GET /api/users/{id}
[HttpPost]                         // POST /api/users

// Model Binding
[FromRoute]   int id              // อ่านจาก URL segment
[FromQuery]   string? search      // อ่านจาก ?search=xxx
[FromBody]    CreateUserDto dto   // อ่านจาก request body (JSON)
[FromHeader]  string authorization // อ่านจาก header

// Validation
[Required]
[StringLength(100, MinimumLength = 2)]
[EmailAddress]
[Range(1, int.MaxValue)]

// Authorization
[Authorize]
[Authorize(Roles = "Admin")]
[AllowAnonymous]

// Response type (สำหรับ Swagger)
[ProducesResponseType<UserDto>(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
```

### Activator — สร้าง Object จาก Type ตอน Runtime

```csharp
// Activator.CreateInstance สร้าง object จาก Type โดยไม่รู้ type ตอน compile

Type type    = typeof(UserService);
object instance = Activator.CreateInstance(type)!;
// เทียบเท่ากับ new UserService() แต่ทำได้ตอน runtime

// ใช้กับ generic
UserService service = (UserService)Activator.CreateInstance(typeof(UserService))!;

// ส่ง constructor arguments
var serviceWithDeps = Activator.CreateInstance(
    typeof(UserService),
    new object[] { repository, logger }  // constructor params
)!;

// Dynamic type จาก string (ใช้ใน plugin systems)
Type? dynamicType = Type.GetType("MyApp.Services.UserService, MyApp");
if (dynamicType != null)
{
    object dynamicInstance = Activator.CreateInstance(dynamicType)!;
}
```

### Reflection + Attribute + Activator ทำงานร่วมกันใน Framework

นี่คือสิ่งที่ ASP.NET Core ทำตอน startup เพื่อ register controllers อัตโนมัติ:

```csharp
// สิ่งที่ AddControllers() ทำข้างใน (simplified):

// 1. scan assembly หา class ที่ inherit จาก ControllerBase
var assembly    = Assembly.GetExecutingAssembly();
var controllers = assembly.GetTypes()
    .Where(t => t.IsSubclassOf(typeof(ControllerBase)) && !t.IsAbstract);

foreach (var controllerType in controllers)
{
    // 2. หา methods ที่มี [HttpGet], [HttpPost] etc.
    var methods = controllerType.GetMethods(BindingFlags.Public | BindingFlags.Instance);

    foreach (var method in methods)
    {
        var httpGet  = method.GetCustomAttribute<HttpGetAttribute>();
        var httpPost = method.GetCustomAttribute<HttpPostAttribute>();
        var route    = method.GetCustomAttribute<RouteAttribute>();

        if (httpGet != null)
        {
            // 3. register route → method mapping
            RegisterRoute("GET", route?.Template ?? httpGet.Template, controllerType, method);
        }
    }

    // 4. ตอน request เข้ามา สร้าง controller instance ด้วย Activator
    //    (จริงๆ ใช้ DI container แต่ concept เดียวกัน)
    var instance = Activator.CreateInstance(controllerType, /* inject deps */);
    method.Invoke(instance, /* bind parameters */);
}
```

```csharp
// Model Validation ก็ใช้ Reflection อ่าน [Required], [StringLength] etc.
public class ModelValidator
{
    public List<string> Validate(object model)
    {
        var errors = new List<string>();
        var type   = model.GetType();

        foreach (var property in type.GetProperties())
        {
            var value      = property.GetValue(model);
            var attributes = property.GetCustomAttributes<ValidationAttribute>();

            foreach (var attr in attributes)
            {
                if (!attr.IsValid(value))
                    errors.Add($"{property.Name}: {attr.ErrorMessage}");
            }
        }

        return errors;
    }
}
```

---

## 5. Thread, Task, Async/Await — "ทำหลายอย่างพร้อมกัน"

นี่คือ feature ที่ซับซ้อนที่สุดในบทนี้ แต่สำคัญมากสำหรับ web server เพราะมัน handle requests พร้อมกันหลายร้อย/หลายพัน

### ปัญหาที่ต้องแก้ — Single-threaded Blocking

```csharp
// ❌ ถ้า web server ทำแบบนี้:
while (true)
{
    var request = AcceptConnection();   // รอ connection
    var data    = ReadFromDatabase();   // รอ 200ms (blocking!)
    var result  = ProcessData(data);    // ประมวลผล
    SendResponse(result);               // ส่งกลับ
}
// request ที่ 2 ต้องรอจนกว่า request ที่ 1 จะเสร็จทั้งหมด
// ถ้า 1000 requests, request สุดท้ายรอ 200 วินาที!
```

### Thread — หน่วยงานอิสระ

**Thread** คือ execution path อิสระที่ OS จัดการ — หลาย Thread รันพร้อมกันได้ (บน multi-core CPU)

```csharp
// สร้าง Thread โดยตรง
var thread = new Thread(() =>
{
    Console.WriteLine($"Running on thread {Thread.CurrentThread.ManagedThreadId}");
    Thread.Sleep(1000);  // จำลอง work
    Console.WriteLine("Done");
});

thread.Start();
Console.WriteLine("Main thread continues...");
thread.Join();  // รอให้ thread เสร็จ
```

**ปัญหาของ Thread:**
- สร้างใหม่แต่ละครั้งแพงมาก (OS ต้องจัดสรร stack memory ~1MB)
- ถ้า thread แค่ "รอ" I/O อยู่ — มันยึด thread นั้นไว้เฉยๆ เปลือง
- 1000 concurrent requests = 1000 threads = ~1GB RAM สำหรับ stack เฉยๆ

### ThreadPool — Pool ของ Thread สำเร็จรูป

```csharp
// แทนที่จะสร้าง thread ใหม่ทุกครั้ง ใช้ pool ที่มีอยู่แล้ว
ThreadPool.QueueUserWorkItem(_ =>
{
    Console.WriteLine($"Work on thread {Thread.CurrentThread.ManagedThreadId}");
    DoSomeWork();
});
// ThreadPool จัดการ reuse thread ให้ แต่ยังมีปัญหา blocking อยู่
```

### Task — Abstraction ของ "งานที่จะทำ"

**Task** คือ abstraction ที่ represent งานที่กำลังทำ หรือจะทำ โดยไม่ผูกติดกับ thread ใดโดยตรง

```csharp
// Task.Run — รัน CPU-bound work บน ThreadPool
Task<int> task = Task.Run(() =>
{
    int result = HeavyCalculation();  // CPU-bound
    return result;
});

// ยังไม่รอ — ทำงานอื่นไปก่อนได้
DoOtherWork();

// ค่อยรอผลตอนต้องการ
int value = await task;
```

```csharp
// Task.WhenAll — รอหลาย Task พร้อมกัน
var task1 = GetUserAsync(1);
var task2 = GetOrdersAsync(userId);
var task3 = GetRecommendationsAsync(userId);

// รันทั้ง 3 พร้อมกัน แล้วรอทุกตัวเสร็จ
var (user, orders, recs) = await (task1, task2, task3);
// ถ้าแต่ละอันใช้ 100ms ก็รอแค่ ~100ms ไม่ใช่ 300ms

// Task.WhenAny — รอตัวใดตัวหนึ่งเสร็จก่อน
var fastestResult = await Task.WhenAny(task1, task2, task3);
```

### Async / Await — "รอโดยไม่บล็อก Thread"

นี่คือ feature ที่สำคัญที่สุดสำหรับ web:

```csharp
// ❌ Synchronous — บล็อก thread ระหว่างรอ DB
public User GetUser(int id)
{
    var user = database.Query($"SELECT * FROM Users WHERE Id = {id}");
    // Thread นี้ถูก block อยู่นาน 50ms ระหว่างรอ network/disk
    // ทำอะไรอื่นไม่ได้เลย
    return user;
}

// ✅ Asynchronous — คืน thread ระหว่างรอ
public async Task<User> GetUserAsync(int id)
{
    var user = await database.QueryAsync($"SELECT * FROM Users WHERE Id = {id}");
    // ตรง await: thread คืนกลับไปที่ ThreadPool
    // รอ network/disk โดยไม่ใช้ thread
    // เมื่อ DB ตอบกลับ ดึง thread ใหม่มาทำงานต่อ
    return user;
}
```

### ภาพรวมว่า async/await ทำงานอย่างไร

```
Thread 1: รับ Request A ──► await DB query ──────────────────► return Response A
                                 │ คืน thread                   ↑ thread ใหม่รับ
                                 │                              │
Thread 1: (ว่าง)    ─────────────► รับ Request B ──► await DB ──►...
                                                          │ คืน thread
Thread 1: (ว่าง)    ──────────────────────────────────────► รับ Request C ──►...

= 1 Thread handle ได้หลาย requests พร้อมกัน เพราะ "await" คืน thread ระหว่างรอ I/O
```

### State Machine ที่ Compiler สร้าง

เมื่อเราเขียน `async/await` compiler แปลงมันเป็น state machine อัตโนมัติ:

```csharp
// โค้ดที่เราเขียน:
public async Task<string> GetDataAsync()
{
    var a = await FetchAAsync();   // await ที่ 1
    var b = await FetchBAsync(a);  // await ที่ 2
    return $"{a}-{b}";
}

// compiler แปลงเป็น (simplified):
public Task<string> GetDataAsync()
{
    var stateMachine = new GetDataAsyncStateMachine { _this = this };
    stateMachine.MoveNext();
    return stateMachine.Task;
}

// State Machine ที่ compiler สร้าง:
struct GetDataAsyncStateMachine : IAsyncStateMachine
{
    public int _state = -1;  // -1 = ยังไม่เริ่ม
    public AsyncTaskMethodBuilder<string> _builder;
    private string _a, _b;
    private TaskAwaiter<string> _awaiter;

    public void MoveNext()
    {
        switch (_state)
        {
            case -1:  // ครั้งแรก
                _awaiter = FetchAAsync().GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _state = 0;
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;  // คืน thread!
                }
                goto case 0;

            case 0:   // FetchA เสร็จแล้ว
                _a       = _awaiter.GetResult();
                _awaiter = FetchBAsync(_a).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _state = 1;
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;  // คืน thread อีกครั้ง!
                }
                goto case 1;

            case 1:   // FetchB เสร็จแล้ว
                _b = _awaiter.GetResult();
                _builder.SetResult($"{_a}-{_b}");  // complete task
                break;
        }
    }
}
```

ไม่จำเป็นต้องจำโครงสร้าง state machine นี้ แต่การเข้าใจว่า **compiler แปลง await เป็น callback chain** ช่วยให้เข้าใจว่าทำไม:
- `async void` อันตราย (exception หายไป)
- ไม่ควร `.Result` หรือ `.Wait()` (deadlock)
- `ConfigureAwait(false)` หมายความว่าอะไร

### Async ใน ASP.NET Core

```csharp
// Controller method ส่วนมากควรเป็น async
[HttpGet("{id}")]
public async Task<ActionResult<UserDto>> GetUser(int id)
{
    // await ทุกที่ที่มี I/O
    var user = await _userRepository.GetByIdAsync(id);

    if (user is null)
        return NotFound();

    return Ok(new UserDto(user.Id, user.Name, user.Email));
}

// Minimal API
app.MapGet("/users/{id}", async (int id, IUserRepository repo) =>
{
    var user = await repo.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
});
```

### กฎการใช้ Async/Await ใน Web API

```csharp
// ✅ DO: async ตลอดสาย
public async Task<UserDto> GetUserAsync(int id)
{
    var user = await _repo.GetByIdAsync(id);  // async repo
    return MapToDto(user);
}

// ❌ DON'T: .Result หรือ .Wait() — อาจ deadlock
public UserDto GetUser(int id)
{
    var user = _repo.GetByIdAsync(id).Result;  // DANGER!
    return MapToDto(user);
}

// ❌ DON'T: async void — exception หายไป ไม่มีใน catch ได้
public async void HandleRequest()   // BAD — ยกเว้น event handler
{
    await DoWorkAsync();  // ถ้า throw exception จะ crash process
}

// ✅ DO: async Task แทน
public async Task HandleRequestAsync()  // GOOD
{
    await DoWorkAsync();
}

// ✅ DO: CancellationToken — ยกเลิกได้เมื่อ client disconnect
[HttpGet("{id}")]
public async Task<ActionResult<User>> GetUser(int id, CancellationToken ct)
{
    var user = await _repo.GetByIdAsync(id, ct);  // pass token ลงไป
    return Ok(user);
}
// ถ้า client ปิด browser ระหว่างรอ ct จะถูก cancel → DB query หยุดได้
```

---

## 6. Top-level Statements & Program Args

### Top-level Statements (C# 9+)

```csharp
// ❌ ก่อน C# 9 — ต้องมี class และ Main method
namespace MyApp
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello");
        }
    }
}

// ✅ C# 9+ — เขียนตรงได้เลย
Console.WriteLine("Hello");

// compiler สร้าง Main ให้อัตโนมัติ เป็น synthesized entry point
// args ยังใช้ได้โดยไม่ต้องประกาศ
if (args.Length > 0)
    Console.WriteLine($"First arg: {args[0]}");
```

### Program.cs ใน ASP.NET Core

```csharp
// Program.cs ที่เราเห็นคือ top-level statements ทั้งนั้น
var builder = WebApplication.CreateBuilder(args);
//                                          ↑ args มาจากไหน?
// compiler สร้าง: static async Task Main(string[] args) { /* โค้ดทั้งหมดข้างบน */ }

builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
await app.RunAsync();
```

### args ใช้ทำอะไรใน Web App?

```csharp
// Command-line args ส่งตรงไป IConfiguration
// dotnet run --urls http://localhost:3000
// dotnet run --environment Production
// dotnet run --ConnectionStrings:Default "Server=..."

var builder = WebApplication.CreateBuilder(args);
// builder.Configuration จะมีค่าจาก args โดยอัตโนมัติ

// อ่าน environment
var env = builder.Environment.EnvironmentName;
// "Development" / "Production" / "Staging"

// override urls จาก args
// dotnet run --urls "http://*:8080;https://*:8443"
```

```csharp
// ใช้ args เพื่อเปลี่ยน behavior ตาม environment:
var builder = WebApplication.CreateBuilder(args);
var app     = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();       // แสดง Swagger ใน dev เท่านั้น
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

await app.RunAsync();
```

---

## 7. ทุก Feature ทำงานร่วมกัน — ตัวอย่างจริง

เพื่อให้เห็นว่าทุกอย่าง connect กัน ดูโค้ดนี้ที่ใช้ทุก feature พร้อมกัน:

```csharp
// ─── Attribute (metadata) ──────────────────────────────────
[AttributeUsage(AttributeTargets.Method)]
public class LogExecutionTimeAttribute : Attribute
{
    public string? Tag { get; init; }
}

// ─── Generic + Delegate ────────────────────────────────────
public class Pipeline<TIn, TOut>
{
    private readonly List<Func<TIn, TIn>> _middlewares = new();

    public Pipeline<TIn, TOut> Use(Func<TIn, TIn> middleware)  // extension-like pattern
    {
        _middlewares.Add(middleware);
        return this;
    }

    public async Task<TOut> ExecuteAsync(TIn input, Func<TIn, Task<TOut>> handler)
    {
        var current = input;

        foreach (var middleware in _middlewares)
            current = middleware(current);

        return await handler(current);
    }
}

// ─── Extension Method ──────────────────────────────────────
public static class PipelineExtensions
{
    public static Pipeline<TIn, TOut> UseLogging<TIn, TOut>(
        this Pipeline<TIn, TOut> pipeline,
        ILogger logger)
    {
        return pipeline.Use(input =>
        {
            logger.LogInformation("Processing: {Input}", input);
            return input;
        });
    }
}

// ─── Reflection + Attribute ────────────────────────────────
public class TimingMiddleware
{
    // อ่าน attribute ด้วย reflection เพื่อตัดสินใจว่าจะ log หรือเปล่า
    public static bool ShouldLog(MethodInfo method)
    {
        return method.GetCustomAttribute<LogExecutionTimeAttribute>() is not null;
    }
}

// ─── Async + Task + Generic ────────────────────────────────
public class UserService
{
    private readonly ILogger<UserService> _logger;
    private readonly Pipeline<int, User?> _pipeline;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
        _pipeline = new Pipeline<int, User?>()
            .UseLogging(_logger)       // Extension Method
            .Use(id => id > 0 ? id : throw new ArgumentException("Invalid id"));
    }

    [LogExecutionTime(Tag = "GetUser")]  // Attribute
    public async Task<User?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return await _pipeline.ExecuteAsync(id, async userId =>
        {
            await Task.Delay(50, ct);  // จำลอง DB query
            return userId == 1 ? new User(1, "Alice") : null;
        });
    }
}

// ─── Top-level Statements + WebApp ────────────────────────
var builder = WebApplication.CreateBuilder(args);  // args (top-level)
builder.Services.AddScoped<UserService>();          // Generic DI
builder.Logging.AddConsole();

var app = builder.Build();

// Lambda as route handler (Delegate)
app.MapGet("/users/{id}", async (int id, UserService svc, CancellationToken ct) =>
{
    var user = await svc.GetByIdAsync(id, ct);  // Async/Await
    return user is null ? Results.NotFound() : Results.Ok(user);
});

await app.RunAsync();

record User(int Id, string Name);
```

---

## บทสรุป

| Feature | แก้ปัญหา | ใช้ใน ASP.NET Core |
|---|---|---|
| **Generic** | Code ซ้ำสำหรับหลาย type | `AddScoped<T>()`, `IOptions<T>`, `Task<T>` |
| **Extension Method** | เพิ่ม method ให้ type ที่แก้ไม่ได้, Builder pattern | `UseAuthentication()`, `AddControllers()`, `MapGet()` |
| **Delegate / Lambda** | Function เป็น first-class value | Middleware pipeline, Route handlers, LINQ |
| **Reflection + Attribute** | Framework อ่าน metadata โดยไม่รู้ type ล่วงหน้า | Route discovery, Model binding, Validation |
| **Activator** | สร้าง object จาก type ตอน runtime | DI Container, Controller instantiation |
| **Async / Await / Task** | ไม่บล็อก thread ระหว่างรอ I/O | ทุก I/O operation: DB, HTTP, File |
| **Top-level Statements** | ลด boilerplate code | Program.cs ทั้งหมด |

---

## Workshop

1. **Lab 3.1** — สร้าง extension method `ToSlug(this string text)` ที่แปลง `"Hello World"` เป็น `"hello-world"` แล้วใช้ใน minimal API endpoint ที่รับ title แล้ว return slug

2. **Lab 3.2** — สร้าง generic `Result<T>` class ที่มี `bool IsSuccess`, `T? Value`, `string? Error` แล้วใช้ wrap response จาก service layer

3. **Lab 3.3** — สร้าง `[RequireApiKey]` attribute แล้วใช้ Reflection เขียน middleware ที่อ่าน attribute บน endpoint และตรวจสอบ `X-Api-Key` header อัตโนมัติ

4. **Lab 3.4** — เปรียบเทียบ performance ระหว่าง synchronous และ async endpoint โดยสร้าง 2 endpoints ที่ทั้งคู่ delay 100ms แต่หนึ่งใช้ `Thread.Sleep` และอีกอันใช้ `await Task.Delay` แล้วยิง 50 requests พร้อมกันด้วย:
   ```bash
   for i in {1..50}; do curl http://localhost:5000/sync & done
   for i in {1..50}; do curl http://localhost:5000/async & done
   ```

---

*→ บทถัดไป: **Part 4 — HttpContext & Request Pipeline** — ตอนที่ request เข้ามาถึง ASP.NET Core มันกลายเป็น HttpContext ได้ยังไง และทุก property บน HttpContext หมายความว่าอะไร*
