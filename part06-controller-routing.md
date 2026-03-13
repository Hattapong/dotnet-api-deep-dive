# Part 6 — Controller & Routing

> **Deep Dive Series: .NET Web API Internals**  
> เราเข้าใจ Pipeline และ DI แล้ว — บทนี้จะดูว่า Controller คืออะไร  
> และ request ถูก route ไปหา action method ยังไง

---

## 1. Controller คืออะไร?

**Controller** คือ class ที่ทำหน้าที่รับ HTTP request แล้วตอบกลับ มันคือ "handler" ที่อยู่ปลายสุดของ middleware pipeline

```
HTTP Request
     │
     ▼
[ Middleware Pipeline ]
     │
     ▼
[ EndpointMiddleware ]
     │
     ├── Minimal API Handler  →  (string name) => Results.Ok(...)
     │
     └── Controller Action   →  UsersController.GetById(int id)
```

ใน ASP.NET Core มี 2 วิธีหลักในการเขียน handler:

| | **Controller** | **Minimal API** |
|---|---|---|
| รูปแบบ | Class inherit `ControllerBase` | Lambda / method โดยตรง |
| เหมาะกับ | API ขนาดกลาง–ใหญ่, ต้องการ organize | API เล็ก, prototype, simple endpoints |
| Model Binding | อัตโนมัติผ่าน Attribute | อัตโนมัติผ่าน Parameter type |
| Filter | `[ActionFilter]`, `[ExceptionFilter]` | `IEndpointFilter` |
| Testing | `WebApplicationFactory` + direct instantiation | `WebApplicationFactory` |

---

## 2. Controller พื้นฐาน

```csharp
// UsersController.cs
using Microsoft.AspNetCore.Mvc;

[ApiController]           // ← เปิด behaviors พิเศษ (อธิบายด้านล่าง)
[Route("api/[controller]")] // ← route prefix = "api/users"
public class UsersController : ControllerBase
{
    private static readonly List<User> _db = new()
    {
        new(1, "Alice", "alice@example.com"),
        new(2, "Bob",   "bob@example.com"),
    };

    // GET api/users
    [HttpGet]
    public ActionResult<IEnumerable<User>> GetAll()
    {
        return Ok(_db);
    }

    // GET api/users/1
    [HttpGet("{id}")]
    public ActionResult<User> GetById(int id)
    {
        var user = _db.FirstOrDefault(u => u.Id == id);
        return user is null ? NotFound() : Ok(user);
    }

    // POST api/users
    [HttpPost]
    public ActionResult<User> Create(CreateUserDto dto)
    {
        var user = new User(_db.Max(u => u.Id) + 1, dto.Name, dto.Email);
        _db.Add(user);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // PUT api/users/1
    [HttpPut("{id}")]
    public ActionResult<User> Update(int id, UpdateUserDto dto)
    {
        var user = _db.FirstOrDefault(u => u.Id == id);
        if (user is null) return NotFound();

        var updated = user with { Name = dto.Name, Email = dto.Email };
        _db[_db.IndexOf(user)] = updated;
        return Ok(updated);
    }

    // DELETE api/users/1
    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        var user = _db.FirstOrDefault(u => u.Id == id);
        if (user is null) return NotFound();

        _db.Remove(user);
        return NoContent();  // 204
    }
}

record User(int Id, string Name, string Email);
record CreateUserDto(string Name, string Email);
record UpdateUserDto(string Name, string Email);
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();  // register Controller infrastructure

var app = builder.Build();
app.MapControllers();               // scan + register endpoints จาก Controllers
await app.RunAsync();
```

---

## 3. [ApiController] ทำอะไร?

`[ApiController]` เปิด behaviors 3 อย่างอัตโนมัติ:

### 3.1 Automatic Model Validation

```csharp
// ❌ ถ้าไม่มี [ApiController] ต้องเช็คเอง
[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    if (!ModelState.IsValid)           // ต้องเขียนทุก action
        return BadRequest(ModelState);

    // ... logic
}

// ✅ ถ้ามี [ApiController] — ถ้า model invalid จะ return 400 อัตโนมัติ
[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    // ถึงตรงนี้ได้ = dto valid แน่นอนแล้ว
    // ... logic
}
```

### 3.2 Binding Source Inference

```csharp
// ❌ ไม่มี [ApiController] ต้องระบุ source เองทุกครั้ง
public IActionResult Create([FromBody] CreateUserDto dto) { }
public IActionResult GetById([FromRoute] int id) { }
public IActionResult Search([FromQuery] string keyword) { }

// ✅ มี [ApiController] — infer อัตโนมัติตาม type และ convention
public IActionResult Create(CreateUserDto dto)  // complex type → [FromBody]
public IActionResult GetById(int id)            // simple type + route param → [FromRoute]
public IActionResult Search(string keyword)     // simple type ไม่อยู่ใน route → [FromQuery]
```

### 3.3 ProblemDetails Response

```csharp
// ❌ ไม่มี [ApiController]
return BadRequest(ModelState);
// {"Name":["The Name field is required."]}   ← format ไม่ standard

// ✅ มี [ApiController]
return BadRequest(ModelState);
// {
//   "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
//   "title": "One or more validation errors occurred.",
//   "status": 400,
//   "errors": { "Name": ["The Name field is required."] }
// }
// RFC 7807 ProblemDetails format — standard สำหรับ HTTP API errors
```

---

## 4. Routing — จับคู่ URL กับ Action

### Route Template

```csharp
[Route("api/[controller]")]   // [controller] = ชื่อ class ตัด "Controller" ออก
public class UsersController  // → "api/users"

[Route("api/v{version}/[controller]")]  // → "api/v2/users"
public class UsersV2Controller

// Hardcode ก็ได้
[Route("api/members")]        // → "api/members" เสมอ ไม่สนชื่อ class
public class UsersController
```

### HTTP Method Attributes

```csharp
[HttpGet]                // GET /api/users
[HttpGet("{id}")]        // GET /api/users/42
[HttpGet("{id}/orders")] // GET /api/users/42/orders
[HttpPost]               // POST /api/users
[HttpPut("{id}")]        // PUT /api/users/42
[HttpPatch("{id}")]      // PATCH /api/users/42
[HttpDelete("{id}")]     // DELETE /api/users/42
```

### Route Constraints — จำกัด type ของ parameter

```csharp
[HttpGet("{id:int}")]           // id ต้องเป็น int เท่านั้น
[HttpGet("{id:int:min(1)}")]    // int และต้อง >= 1
[HttpGet("{name:alpha}")]       // ตัวอักษรเท่านั้น
[HttpGet("{slug:regex(^[a-z-]+$)}")]  // match regex
[HttpGet("{date:datetime}")]    // ต้อง parse เป็น DateTime ได้

// ถ้า constraint ไม่ผ่าน → route นี้ไม่ match → ลองหา route อื่น
// ถ้าไม่เจอเลย → 404
```

### Route Parameter vs Query String

```csharp
// Route parameter — อยู่ใน URL path
// GET /api/users/42
[HttpGet("{id}")]
public ActionResult<User> GetById(int id) { }
//                                  ↑ bind จาก route segment

// Query string — อยู่หลัง ?
// GET /api/users?page=2&size=10&search=alice
[HttpGet]
public ActionResult<IEnumerable<User>> GetAll(
    int    page   = 1,
    int    size   = 10,
    string? search = null)
//  ↑ bind จาก query string อัตโนมัติ
{ }
```

---

## 5. Model Binding — จาก Request ไป C# Object

Model Binding คือกระบวนการที่ ASP.NET Core แปลง HTTP request data ให้เป็น C# parameter

```csharp
// Sources ทั้งหมดที่ bind ได้:

[HttpPost("{id}")]
public IActionResult Example(
    [FromRoute]  int           id,          // จาก URL path segment
    [FromQuery]  string?       search,      // จาก ?search=xxx
    [FromBody]   CreateUserDto dto,         // จาก request body (JSON)
    [FromHeader] string?       apiKey,      // จาก header X-Api-Key → apiKey
    [FromHeader(Name = "X-Api-Key")]
                 string?       key,         // ระบุชื่อ header ชัดๆ
    [FromForm]   IFormFile?    avatar,      // จาก multipart form data
    [FromServices] ILogger<UsersController> logger  // จาก DI
)
{ }
```

### Complex Object Binding

```csharp
public record CreateUserDto(
    [Required]
    [StringLength(100, MinimumLength = 2)]
    string Name,

    [Required]
    [EmailAddress]
    string Email,

    int Age = 0
);

// POST /api/users
// Body: {"name": "Alice", "email": "alice@example.com", "age": 30}
// → ASP.NET Core deserialize JSON → CreateUserDto instance
// → validate Data Annotations
// → ถ้า invalid → 400 ProblemDetails (ถ้ามี [ApiController])
// → ถ้า valid → เข้า action method
```

---

## 6. ActionResult — ประเภทของ Response

```csharp
// IActionResult — return type ทั่วไป
public IActionResult GetUser(int id)
{
    var user = FindUser(id);
    if (user is null) return NotFound();      // 404
    return Ok(user);                          // 200 + JSON body
}

// ActionResult<T> — บอก Swagger ว่า success response type คืออะไร
public ActionResult<User> GetUser(int id)
{
    var user = FindUser(id);
    if (user is null) return NotFound();
    return Ok(user);    // หรือเขียนแค่ return user; ก็ได้ (implicit conversion)
}
```

### Helper Methods ใน ControllerBase

```csharp
// 2xx Success
return Ok(data);                                        // 200 + body
return Created("/api/users/1", user);                   // 201 + Location header
return CreatedAtAction(nameof(GetById), new{id=1}, user); // 201 + Location จาก route
return NoContent();                                     // 204 (DELETE, PUT success)
return Accepted();                                      // 202 (async operation started)

// 4xx Client Error
return BadRequest();                                    // 400
return BadRequest(ModelState);                          // 400 + validation errors
return Unauthorized();                                  // 401
return Forbid();                                        // 403
return NotFound();                                      // 404
return Conflict();                                      // 409
return UnprocessableEntity(ModelState);                 // 422

// 5xx Server Error (ส่วนใหญ่ไม่ return เอง ปล่อยให้ exception handler จัดการ)
return StatusCode(500, new { error = "Something went wrong" });

// Redirect
return Redirect("https://example.com");                 // 302
return RedirectToAction("GetById", new { id = 1 });     // 302 → route
```

---

## 7. Dependency Injection ใน Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // inject ผ่าน constructor — วิธีแนะนำ
    private readonly IUserService         _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        IUserService              userService,
        ILogger<UsersController>  logger)
    {
        _userService = userService;
        _logger      = logger;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetById(
        int               id,
        CancellationToken ct)     // ← inject จาก framework ไม่ใช่ DI
    {
        _logger.LogInformation("Getting user {UserId}", id);

        var user = await _userService.GetByIdAsync(id, ct);

        return user is null ? NotFound() : Ok(user);
    }

    // inject เฉพาะ action ที่ต้องการ (ไม่ต้องใส่ใน constructor)
    [HttpPost("{id}/notify")]
    public async Task<IActionResult> SendNotification(
        int                id,
        [FromServices] IEmailService emailService)  // inject ตรง action
    {
        await emailService.SendAsync(id);
        return NoContent();
    }
}
```

---

## 8. Minimal API เปรียบเทียบกับ Controller

```csharp
// ── Controller Style ─────────────────────────────────────────
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    public UsersController(IUserService service) => _service = service;

    [HttpGet("{id}")]
    [ProducesResponseType<UserDto>(200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<UserDto>> GetById(int id, CancellationToken ct)
    {
        var user = await _service.GetByIdAsync(id, ct);
        return user is null ? NotFound() : Ok(user);
    }
}

// ── Minimal API Style ────────────────────────────────────────
app.MapGet("/api/users/{id}", async (
    int            id,
    IUserService   service,
    CancellationToken ct) =>
{
    var user = await service.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
})
.WithName("GetUserById")
.Produces<UserDto>(200)
.Produces(404);
```

เมื่อไหร่ควรใช้อะไร:

```
Controller                          Minimal API
──────────────────────────────────  ──────────────────────────────────
API มีหลาย endpoint ใน domain เดียว  endpoint ไม่กี่ตัว
ต้องการ Action Filters               endpoint เรียบง่าย
ทีมคุ้นเคยกับ MVC pattern           prototype / microservice เล็กๆ
ต้องการ Swagger โดยไม่ config มาก   ต้องการ performance สูงสุด
```

---

## 9. ตัวอย่างสมบูรณ์

```csharp
// Models
public record User(int Id, string Name, string Email);
public record CreateUserDto(
    [Required][StringLength(100, MinimumLength = 2)] string Name,
    [Required][EmailAddress] string Email);
public record UpdateUserDto(
    [Required][StringLength(100, MinimumLength = 2)] string Name,
    [Required][EmailAddress] string Email);

// Service
public interface IUserService
{
    IReadOnlyList<User> GetAll(string? search = null);
    User? GetById(int id);
    User Create(CreateUserDto dto);
    User? Update(int id, UpdateUserDto dto);
    bool Delete(int id);
}

public class InMemoryUserService : IUserService
{
    private readonly List<User> _users = new()
    {
        new(1, "Alice", "alice@example.com"),
        new(2, "Bob",   "bob@example.com"),
    };
    private int _nextId = 3;

    public IReadOnlyList<User> GetAll(string? search = null) =>
        string.IsNullOrWhiteSpace(search)
            ? _users.AsReadOnly()
            : _users.Where(u =>
                u.Name.Contains(search, StringComparison.OrdinalIgnoreCase) ||
                u.Email.Contains(search, StringComparison.OrdinalIgnoreCase))
              .ToList().AsReadOnly();

    public User? GetById(int id) =>
        _users.FirstOrDefault(u => u.Id == id);

    public User Create(CreateUserDto dto)
    {
        var user = new User(_nextId++, dto.Name, dto.Email);
        _users.Add(user);
        return user;
    }

    public User? Update(int id, UpdateUserDto dto)
    {
        var idx = _users.FindIndex(u => u.Id == id);
        if (idx < 0) return null;
        var updated = _users[idx] with { Name = dto.Name, Email = dto.Email };
        _users[idx] = updated;
        return updated;
    }

    public bool Delete(int id)
    {
        var user = _users.FirstOrDefault(u => u.Id == id);
        if (user is null) return false;
        _users.Remove(user);
        return true;
    }
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService            _service;
    private readonly ILogger<UsersController> _logger;

    public UsersController(IUserService service, ILogger<UsersController> logger)
    {
        _service = service;
        _logger  = logger;
    }

    // GET api/users?search=alice
    [HttpGet]
    [ProducesResponseType<IEnumerable<User>>(StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<User>> GetAll([FromQuery] string? search)
    {
        _logger.LogInformation("GetAll called, search={Search}", search);
        return Ok(_service.GetAll(search));
    }

    // GET api/users/1
    [HttpGet("{id:int}")]
    [ProducesResponseType<User>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<User> GetById(int id)
    {
        var user = _service.GetById(id);
        return user is null ? NotFound() : Ok(user);
    }

    // POST api/users
    [HttpPost]
    [ProducesResponseType<User>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public ActionResult<User> Create(CreateUserDto dto)
    {
        var user = _service.Create(dto);
        _logger.LogInformation("Created user {UserId}", user.Id);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // PUT api/users/1
    [HttpPut("{id:int}")]
    [ProducesResponseType<User>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<User> Update(int id, UpdateUserDto dto)
    {
        var user = _service.Update(id, dto);
        return user is null ? NotFound() : Ok(user);
    }

    // DELETE api/users/1
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult Delete(int id)
    {
        return _service.Delete(id) ? NoContent() : NotFound();
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddSingleton<IUserService, InMemoryUserService>();

var app = builder.Build();
app.MapControllers();
await app.RunAsync();
```

ทดสอบ:
```bash
curl http://localhost:5000/api/users
curl http://localhost:5000/api/users/1
curl http://localhost:5000/api/users?search=alice

curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'

curl -X PUT http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Updated","email":"alice2@example.com"}'

curl -X DELETE http://localhost:5000/api/users/1
```

---

## บทสรุป

| Concept | สรุป |
|---|---|
| **Controller** | Class ที่ inherit `ControllerBase` รับ HTTP request ปลายสุดของ pipeline |
| **[ApiController]** | เปิด Auto Validation, Source Inference, ProblemDetails |
| **[Route]** | กำหนด URL prefix ของ controller ทั้งหมด |
| **[HttpGet/Post/...]** | กำหนด HTTP method + URL ของแต่ละ action |
| **Route Constraints** | `:int`, `:alpha`, `:min(1)` จำกัด type ของ URL segment |
| **Model Binding** | `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]` |
| **ActionResult<T>** | Return type ที่บอก success type + รองรับ error responses |

---

## Workshop

1. **Lab 6.1** — เพิ่ม endpoint `GET /api/users/{id}/profile` ที่ return ข้อมูล extended ของ user (เพิ่ม field `joinedAt`, `orderCount`)

2. **Lab 6.2** — เพิ่ม query string parameter `page` และ `size` ให้ `GET /api/users` พร้อม return pagination metadata `{ items, totalCount, page, size }`

3. **Lab 6.3** — ทดสอบ Automatic Validation โดยส่ง POST ที่ไม่มี `name` หรือ `email` แล้วดู ProblemDetails response

4. **Lab 6.4 (Bonus)** — แปลง `UsersController` ทั้งหมดเป็น Minimal API แล้วเปรียบเทียบจำนวนบรรทัด และพิจารณาว่าแบบไหนอ่านง่ายกว่าสำหรับ team ของคุณ

---

*→ บทถัดไป: **Part 7 — Configuration & Options Pattern** — อ่าน `appsettings.json`, จัดการ environment-specific config, และ strongly-typed options ด้วย `IOptions<T>`*
