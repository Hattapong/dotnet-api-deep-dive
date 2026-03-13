# .NET Web API Deep Dive

คอร์สนี้พาคุณลงลึกตั้งแต่รากฐานของ HTTP จนถึงการสร้าง Web API ระดับ production  
แต่ละบทเรียงต่อกันอย่างมีลำดับ — อ่านตามลำดับเพื่อผลลัพธ์ที่ดีที่สุด

---

## Fundamentals

### [Part 1 — Web Server & Web Application](part01-web-server-fundamentals.md)
เข้าใจว่า Web Server คืออะไร แตกต่างจาก Web Application อย่างไร HTTP request/response มีโครงสร้างยังไง และทำไม plain web server ถึงไม่พอสำหรับ modern API

- HTTP Request & Response Structure
- HTTP Methods และ Status Codes
- PORT และการทำงานของ TCP
- ข้อจำกัดของ Web Server ที่ไม่มี Framework

---

### [Part 2 — Hosting Model & Program.cs](part02-hosting-model-program-cs.md)
จาก console app ธรรมดาสู่ `IHost` — เข้าใจ lifecycle ของ ASP.NET Core ตั้งแต่ startup จนถึง graceful shutdown

- IHost และ IHostedService
- Configuration, Logging, Graceful Shutdown
- BackgroundService
- Program.cs 4 Phases: Builder → Build → Configure Pipeline → Run

---

### [Part 3 — C# Features สำหรับ Web API](part03-csharp-features.md)
ก่อนเข้าใจ framework ต้องเข้าใจ C# features ที่มันสร้างขึ้นมา — ทุก `.UseXxx()` และ `.AddXxx()` คือสิ่งในบทนี้

- Generic — Type เป็น Parameter
- Extension Method — เพิ่ม Method ให้ Type ที่แก้ไม่ได้
- Delegate & Lambda — Function เป็น Object
- Reflection, Attribute, Activator
- Thread, Task, Async/Await
- Top-level Statements & Program Args

---

## Core Concepts

### [Part 4 — HttpContext, Pipeline และ Middleware](part04-httpcontext-pipeline-middleware.md)
เมื่อ HTTP request เข้ามา 1 ครั้ง เกิดอะไรขึ้นตั้งแต่ TCP bytes จนถึง JSON response ที่ส่งกลับ

- HttpContext และทุก Property ที่ควรรู้
- Middleware คืออะไร และ RequestDelegate
- Pipeline — วิธีที่ Middleware ถูก compile เชื่อมกัน
- Short-circuit, Two-way flow
- Built-in Middleware และลำดับที่สำคัญ

---

### [Part 5 — Dependency Injection Deep Dive](part05-di-deep-dive.md)
DI Container ทำงานอย่างไรจากรากฐาน ทำไมถึงต้องมี และ Lifetime ที่เลือกผิดสร้าง bug อะไร

- ทำไมต้อง DI — ปัญหาของ `new()` ทุกอย่างเอง
- IServiceCollection vs IServiceProvider
- Recursive Resolution — วิธีที่ Container หา Dependency
- Singleton, Scoped, Transient — Lifetime จริงๆ
- Captive Dependency — Bug ที่พบบ่อยมาก
- Mini DI Container — สร้างเองเพื่อเข้าใจ

---

## Framework Features

### [Part 6 — Controller & Routing](part06-controller-routing.md)
Controller คืออะไร request ถูก route ไปหา action method ยังไง และ Model Binding ทำงานอย่างไร

- ControllerBase และ `[ApiController]`
- Route Template และ HTTP Method Attributes
- Route Constraints — `:int`, `:guid`, `:regex`
- Model Binding — `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`
- ActionResult และ Helper Methods
- Controller vs Minimal API

---

### [Part 7 — Configuration & Options Pattern](part07-configuration-options.md)
จัดการ config อย่างถูกต้องตั้งแต่ต้น — strongly-typed, validated, และปลอดภัย

- Configuration Sources และ Priority Chain
- appsettings.json และ Environment Override
- Options Pattern — `IOptions<T>`
- `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`
- Options Validation ด้วย `ValidateOnStart()`
- User Secrets สำหรับ Development

---

### [Part 8 — Logging](part08-logging.md)
ทำให้ระบบ observable — Structured Logging, Correlation ID, และ Serilog สำหรับ production

- ILogger และ 6 Log Levels
- Structured Logging — Message Template ไม่ใช่ String Interpolation
- `@` Destructuring Operator
- Log Scopes — เพิ่ม Context ให้ทุก Log ใน Block
- Correlation ID Middleware
- Serilog — Sinks, Enrichers, Rolling File

---

### [Part 9 — Validation](part09-validation.md)
ป้องกัน bad data ก่อนเข้าสู่ระบบ — ทุกชั้นตั้งแต่ input format จนถึง business rules

- Data Annotations — `[Required]`, `[Range]`, `[EmailAddress]`
- Custom Validation Attribute
- IValidatableObject — Cross-property Validation
- FluentValidation — Async Rules, Dependency Injection ใน Validator
- Business Rule Validation ใน Service Layer

---

### [Part 10 — Authentication & Authorization](part10-authentication-authorization.md)
JWT Bearer, Claims, และ Policy-based Authorization จากรากฐาน

- Authentication vs Authorization
- JWT — โครงสร้าง Header/Payload/Signature
- Setup JWT Authentication และ TokenValidationParameters
- TokenService — สร้างและ Verify JWT
- Claims และ ClaimsPrincipal Extensions
- Role-based, Policy-based, Resource-based Authorization

---

### [Part 11 — Error Handling](part11-error-handling.md)
จัดการ error ทุกชั้นให้ consistent — client ได้ข้อมูลที่ถูกต้อง server log ครบ ไม่รั่ว sensitive data

- ProblemDetails — RFC 7807 Standard Error Format
- Exception Middleware — จัดการทุก Exception ที่จุดเดียว
- Custom Exception Types — `NotFoundException`, `BusinessRuleException`
- IExceptionHandler (.NET 8) — Chain Pattern
- Environment-specific Error Detail

---

## Bonus

### [บทอ่านเสริม — การจัดการ Request แบบ Pro](bonus-request-handling-pro.md)
ทุก edge case และ pattern ที่เจอใน production จริงๆ สำหรับ request handling

- Query Parameters — Complex Object, Array, Pagination Pattern
- Route Constraints — Custom `IRouteConstraint`
- Request Body — JSON, Form Data, File Upload, Raw Body Stream
- Headers — Security Headers, API Versioning
- Custom Model Binder
- ETag และ Conditional Response
- HttpRequest / HttpContext Extension Methods

---

## สิ่งที่ต้องรู้ก่อนเริ่ม

- C# พื้นฐาน — class, interface, async/await
- ติดตั้ง [.NET 8 SDK](https://dotnet.microsoft.com/download)
- IDE — Visual Studio, VS Code + C# Dev Kit, หรือ Rider

```bash
# สร้าง project ใหม่สำหรับทดลองแต่ละบท
dotnet new webapi -n MyApi --use-controllers
cd MyApi
dotnet run
```

---

*คอร์สนี้เน้น "เข้าใจว่าทำไม" ไม่ใช่แค่ "ทำยังไง" — อ่านแล้วควรอธิบายให้คนอื่นได้ว่า ASP.NET Core ทำงานอย่างไรข้างใต้*
