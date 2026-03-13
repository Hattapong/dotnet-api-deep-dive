# Part 7 — Configuration & Options Pattern

> **Deep Dive Series: .NET Web API Internals**  
> ค่าที่ hardcode ใน code คือ technical debt ที่รอวันระเบิด  
> บทนี้จะดูวิธีที่ .NET จัดการ configuration อย่างถูกต้องตั้งแต่ต้น

---

## 1. ปัญหาของ Hardcoded Values

```csharp
// ❌ สิ่งที่ไม่ควรทำ
public class EmailService
{
    public async Task SendAsync(string to, string subject)
    {
        using var client = new SmtpClient("smtp.gmail.com");  // hardcode!
        client.Port = 587;                                     // hardcode!
        // ต้องแก้ code ทุกครั้งที่เปลี่ยน server
        // dev ใช้ server อื่น, prod ใช้อีก server — แก้ยังไง?
    }
}

public class JwtService
{
    private const string Secret = "my-super-secret-key";  // hardcode!
    // ถ้า secret หลุดออก git → security incident ทันที
}
```

ปัญหาที่ตามมา:

| ปัญหา | ผลที่เกิด |
|-------|----------|
| แต่ละ environment ต้องการค่าต่างกัน | แก้ code ทุกครั้งที่ deploy |
| Secret ติดไปกับ source code | Security risk |
| เปลี่ยน config ต้อง rebuild + redeploy | Downtime |
| ทดสอบยาก | ต้องแก้ code ก่อน test |

---

## 2. Configuration System ของ .NET

`Host.CreateDefaultBuilder` และ `WebApplication.CreateBuilder` ตั้งค่า configuration providers อัตโนมัติตามลำดับ priority:

```
Priority (สูง → ต่ำ):

1. Command-line arguments       --Smtp:Host=mail.example.com
2. Environment Variables        Smtp__Host=mail.example.com
3. User Secrets (dev only)      secrets.json (ไม่อยู่ใน git)
4. appsettings.{Environment}.json   appsettings.Production.json
5. appsettings.json             ค่า default
6. Default values in code
```

ค่าที่ priority สูงกว่าจะ **override** ค่าที่ต่ำกว่าเสมอ

```
appsettings.json          appsettings.Production.json    Environment Variable
──────────────────        ───────────────────────────    ────────────────────
Smtp:Host = localhost  →  Smtp:Host = smtp.sendgrid.com  (ไม่มี)
Smtp:Port = 25         →  (ไม่มี → ใช้จาก json)         Smtp:Port = 465

ผลลัพธ์ใน Production:
  Smtp:Host = smtp.sendgrid.com   ← จาก appsettings.Production.json
  Smtp:Port = 465                 ← override ด้วย env var
```

---

## 3. appsettings.json — รูปแบบและการอ่าน

### โครงสร้างไฟล์

```json
// appsettings.json
{
  "App": {
    "Name": "My API",
    "Version": "1.0.0",
    "AllowedOrigins": ["https://myapp.com", "https://admin.myapp.com"]
  },
  "Smtp": {
    "Host": "localhost",
    "Port": 25,
    "UseSsl": false,
    "Username": "",
    "Password": ""
  },
  "Jwt": {
    "Issuer": "my-api",
    "Audience": "my-clients",
    "ExpiryMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

```json
// appsettings.Development.json — override เฉพาะ dev
{
  "Smtp": {
    "Host": "localhost",
    "Port": 1025
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}
```

### อ่านด้วย IConfiguration โดยตรง

```csharp
// Program.cs — อ่าน config ตอน startup
var builder = WebApplication.CreateBuilder(args);

// IConfiguration พร้อมใช้ทันทีหลัง CreateBuilder
string appName   = builder.Configuration["App:Name"] ?? "Default";
int    smtpPort  = builder.Configuration.GetValue<int>("Smtp:Port");
string? secret   = builder.Configuration["Jwt:Secret"];  // null ถ้าไม่มี

// GetSection — ดึงเฉพาะ section
IConfigurationSection smtpSection = builder.Configuration.GetSection("Smtp");
string smtpHost = smtpSection["Host"] ?? "localhost";

// GetConnectionString — shorthand สำหรับ ConnectionStrings section
string connStr = builder.Configuration.GetConnectionString("Default") ?? "";
// เทียบเท่ากับ: builder.Configuration["ConnectionStrings:Default"]
```

```csharp
// Controller — inject IConfiguration
[ApiController]
[Route("api/[controller]")]
public class InfoController : ControllerBase
{
    private readonly IConfiguration _config;

    public InfoController(IConfiguration config) => _config = config;

    [HttpGet]
    public IActionResult Get() => Ok(new
    {
        name    = _config["App:Name"],
        version = _config["App:Version"],
        env     = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
    });
}
```

การอ่านด้วย `IConfiguration` โดยตรงมีข้อเสีย: ใช้ string key ที่ผิดพลาดได้ง่าย และไม่มี IntelliSense ดังนั้นเราจะใช้ **Options Pattern** แทน

---

## 4. Options Pattern — Strongly-Typed Configuration

แทนที่จะอ่าน config ด้วย string key เราสร้าง C# class แทน section นั้น

```csharp
// ── Step 1: สร้าง Options class ────────────────────────────
public class SmtpOptions
{
    // ชื่อ property ต้องตรงกับ key ใน JSON (case-insensitive)
    public string Host     { get; set; } = "localhost";
    public int    Port     { get; set; } = 25;
    public bool   UseSsl   { get; set; } = false;
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
}

public class JwtOptions
{
    public string Issuer        { get; set; } = "";
    public string Audience      { get; set; } = "";
    public string Secret        { get; set; } = "";
    public int    ExpiryMinutes { get; set; } = 60;
}
```

```csharp
// ── Step 2: Register ใน Program.cs ─────────────────────────
builder.Services.Configure<SmtpOptions>(
    builder.Configuration.GetSection("Smtp"));

builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection("Jwt"));
```

```csharp
// ── Step 3: Inject และใช้งาน ────────────────────────────────
public class EmailService
{
    private readonly SmtpOptions _smtp;

    public EmailService(IOptions<SmtpOptions> options)
    {
        _smtp = options.Value;
        // _smtp.Host, _smtp.Port, _smtp.UseSsl — strongly typed!
    }

    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_smtp.Host);
        client.Port      = _smtp.Port;
        client.EnableSsl = _smtp.UseSsl;
        // ...
    }
}
```

---

## 5. IOptions vs IOptionsSnapshot vs IOptionsMonitor

สามตัวนี้ใช้ inject Options เหมือนกัน แต่ต่างกันตรงที่อ่านค่าเมื่อไหร่:

```
IOptions<T>          — อ่านครั้งเดียวตอน startup, ไม่ reload
IOptionsSnapshot<T>  — อ่านใหม่ทุก request (Scoped)
IOptionsMonitor<T>   — อ่านใหม่ทันทีเมื่อไฟล์เปลี่ยน (Singleton)
```

### IOptions\<T\> — Simple, ไม่ reload

```csharp
// เหมาะกับ config ที่ไม่เคยเปลี่ยนระหว่าง runtime
public class JwtService
{
    private readonly JwtOptions _jwt;

    public JwtService(IOptions<JwtOptions> options)
    {
        _jwt = options.Value;  // อ่านครั้งเดียวตอนสร้าง service
    }
}
```

### IOptionsSnapshot\<T\> — Reload ต่อ Request

```csharp
// เหมาะกับ Scoped services ที่ต้องการค่าล่าสุดต่อ request
// ใช้ได้กับ Scoped และ Transient เท่านั้น (ไม่ใช่ Singleton)
public class FeatureFlagService
{
    private readonly FeatureFlags _flags;

    public FeatureFlagService(IOptionsSnapshot<FeatureFlags> options)
    {
        _flags = options.Value;
        // ถ้า appsettings.json เปลี่ยน request ถัดไปจะได้ค่าใหม่ทันที
    }
}
```

### IOptionsMonitor\<T\> — Reload แบบ Real-time

```csharp
// เหมาะกับ Singleton services ที่ต้องการตอบสนองต่อการเปลี่ยนแปลง
public class CacheService
{
    private readonly IOptionsMonitor<CacheOptions> _monitor;

    public CacheService(IOptionsMonitor<CacheOptions> monitor)
    {
        _monitor = monitor;

        // subscribe การเปลี่ยนแปลง
        _monitor.OnChange(newOptions =>
        {
            Console.WriteLine($"Cache TTL changed to {newOptions.TtlSeconds}s");
            // reset cache หรือทำ action อื่นๆ
        });
    }

    public int GetTtl() => _monitor.CurrentValue.TtlSeconds;
    //                               ↑ อ่านค่าล่าสุดเสมอ
}
```

---

## 6. Options Validation — ตรวจสอบ Config ตอน Startup

```csharp
// เพิ่ม Data Annotations บน Options class
public class SmtpOptions
{
    [Required]
    public string Host { get; set; } = "";

    [Range(1, 65535)]
    public int Port { get; set; } = 25;

    [Required]
    public string Username { get; set; } = "";
}
```

```csharp
// Register พร้อม validation
builder.Services
    .AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")           // bind จาก section "Smtp"
    .ValidateDataAnnotations()           // validate ด้วย [Required], [Range] etc.
    .ValidateOnStart();                  // validate ตอน app start ไม่รอจนใช้งาน
```

```csharp
// Custom validation logic
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration("Jwt")
    .Validate(jwt =>
    {
        if (string.IsNullOrEmpty(jwt.Secret)) return false;
        if (jwt.Secret.Length < 32) return false;  // ต้องยาวพอสำหรับ HMAC
        return true;
    }, "Jwt:Secret must be at least 32 characters")
    .ValidateOnStart();
```

ถ้า validation ไม่ผ่านตอน startup:

```
Unhandled exception. Microsoft.Extensions.Options.OptionsValidationException:
DataAnnotation validation failed for 'SmtpOptions' members:
'Username' with the error: 'The Username field is required.'.
```

app จะไม่ start — ดีกว่าให้มันล้มตอน production เมื่อส่ง email ครั้งแรก

---

## 7. Environment Variables — Override ใน Production

### รูปแบบชื่อ

Environment variable ใช้ `__` (double underscore) แทน `:` เพราะ `:` ไม่ valid บน Linux

```bash
# JSON key       Environment Variable
# ──────────     ──────────────────────────────
# Smtp:Host   →  Smtp__Host
# Smtp:Port   →  Smtp__Port
# App:Name    →  App__Name

# ตั้งค่าใน shell (Linux/Mac)
export Smtp__Host=smtp.sendgrid.com
export Smtp__Port=465
export Jwt__Secret=my-production-secret-that-is-long-enough

# ตั้งค่าใน Docker
docker run -e Smtp__Host=smtp.sendgrid.com myapp

# ตั้งค่าใน docker-compose.yml
environment:
  - Smtp__Host=smtp.sendgrid.com
  - Smtp__Port=465
```

### Prefix — ป้องกัน Conflict

```csharp
// ถ้าต้องการใช้เฉพาะ env vars ที่ขึ้นต้นด้วย prefix
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");

// ตอนตั้งค่า:
// MYAPP_Smtp__Host=smtp.sendgrid.com
// prefix จะถูกตัดออกอัตโนมัติ → Smtp:Host
```

---

## 8. User Secrets — สำหรับ Development

User Secrets เก็บ sensitive config ไว้นอก project folder ไม่ติดไปกับ git

```bash
# setup (ทำครั้งเดียวต่อ project)
dotnet user-secrets init

# เพิ่ม secret
dotnet user-secrets set "Jwt:Secret" "dev-secret-key-32-chars-minimum!!"
dotnet user-secrets set "Smtp:Password" "dev-password"

# ดู secrets ทั้งหมด
dotnet user-secrets list
```

ไฟล์จะถูกเก็บที่:
- Windows: `%APPDATA%\Microsoft\UserSecrets\{id}\secrets.json`
- Mac/Linux: `~/.microsoft/usersecrets/{id}/secrets.json`

ไม่อยู่ใน project folder → ไม่ติดไปกับ git → ปลอดภัย

```json
// secrets.json (ไม่อยู่ใน project)
{
  "Jwt:Secret": "dev-secret-key-32-chars-minimum!!",
  "Smtp:Password": "dev-password"
}
```

User Secrets ทำงานเฉพาะ `Development` environment เท่านั้น Production ใช้ environment variables หรือ secret management service (Azure Key Vault, AWS Secrets Manager)

---

## 9. ตัวอย่างสมบูรณ์

```csharp
// ── Options Classes ──────────────────────────────────────────
public class AppOptions
{
    public string Name    { get; set; } = "My API";
    public string Version { get; set; } = "1.0.0";
    public string[] AllowedOrigins { get; set; } = [];
}

public class SmtpOptions
{
    [Required] public string Host     { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; } = 25;
    public bool   UseSsl   { get; set; } = false;
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
}

// ── appsettings.json ─────────────────────────────────────────
// {
//   "App": { "Name": "My API", "Version": "1.0.0" },
//   "Smtp": { "Host": "localhost", "Port": 1025, "UseSsl": false }
// }

// ── Program.cs ───────────────────────────────────────────────
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Register + validate options
builder.Services
    .AddOptions<AppOptions>()
    .BindConfiguration("App");

builder.Services
    .AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddScoped<IEmailService, EmailService>();

var app = builder.Build();
app.MapControllers();
await app.RunAsync();

// ── EmailService.cs ──────────────────────────────────────────
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}

public class EmailService : IEmailService
{
    private readonly SmtpOptions             _smtp;
    private readonly ILogger<EmailService>   _logger;

    public EmailService(IOptions<SmtpOptions> options, ILogger<EmailService> logger)
    {
        _smtp   = options.Value;
        _logger = logger;
    }

    public async Task SendAsync(string to, string subject, string body)
    {
        _logger.LogInformation(
            "Sending email to {To} via {Host}:{Port}", to, _smtp.Host, _smtp.Port);

        // จำลอง send
        await Task.Delay(100);
        _logger.LogInformation("Email sent successfully");
    }
}

// ── InfoController.cs ────────────────────────────────────────
[ApiController]
[Route("api/[controller]")]
public class InfoController : ControllerBase
{
    private readonly AppOptions _app;

    public InfoController(IOptions<AppOptions> options)
    {
        _app = options.Value;
    }

    [HttpGet]
    public IActionResult Get() => Ok(new
    {
        name    = _app.Name,
        version = _app.Version,
        env     = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production"
    });
}

// ── TestEmailController.cs ───────────────────────────────────
[ApiController]
[Route("api/[controller]")]
public class TestEmailController : ControllerBase
{
    private readonly IEmailService _email;

    public TestEmailController(IEmailService email) => _email = email;

    [HttpPost]
    public async Task<IActionResult> Send([FromBody] SendEmailRequest req)
    {
        await _email.SendAsync(req.To, req.Subject, req.Body);
        return Ok(new { message = "Email sent" });
    }
}

public record SendEmailRequest(string To, string Subject, string Body);
```

ทดสอบ config override ด้วย environment variable:

```bash
# รันด้วย config ปกติ
dotnet run

# override smtp port ผ่าน env var
Smtp__Port=2525 dotnet run

# override ผ่าน command-line argument
dotnet run --Smtp:Port=2525 --App:Name="My Overridden API"
```

---

## บทสรุป

```
Configuration Sources (priority สูง → ต่ำ):
  Command-line args  >  Environment Variables  >  User Secrets
  >  appsettings.{Env}.json  >  appsettings.json

Options Pattern:
  IOptions<T>          → อ่านครั้งเดียว ไม่ reload  (Singleton-safe)
  IOptionsSnapshot<T>  → reload ต่อ request         (Scoped/Transient)
  IOptionsMonitor<T>   → reload real-time + OnChange (Singleton-safe)

Best Practices:
  ✅ ใช้ Options Pattern แทน IConfiguration โดยตรง
  ✅ ValidateOnStart เพื่อ fail fast ตอน startup
  ✅ User Secrets สำหรับ dev, Environment Variables สำหรับ prod
  ❌ ห้าม commit secrets ลง git
  ❌ ห้าม hardcode config ใน code
```

---

## Workshop

1. **Lab 7.1** — สร้าง `DatabaseOptions` class สำหรับ connection string, timeout, และ max retry แล้ว validate ว่า connection string ต้องไม่ว่าง และ timeout ต้องอยู่ระหว่าง 5–120 วินาที

2. **Lab 7.2** — เพิ่ม `appsettings.Development.json` ที่ override log level เป็น Debug และ smtp port เป็น 1025 แล้วดูว่า config ถูก merge ถูกต้อง

3. **Lab 7.3** — ใช้ `IOptionsMonitor<T>` สร้าง `FeatureFlagService` ที่มี flag `EnableNewFeature: true/false` แล้วแก้ไข appsettings.json ระหว่างที่ app รันอยู่ดูว่า endpoint อ่านค่าใหม่ได้ทันทีหรือไม่

4. **Lab 7.4 (Bonus)** — ตั้งค่า User Secrets สำหรับ `Jwt:Secret` แล้วตรวจสอบว่า secret ไม่ปรากฏใน appsettings.json และไม่ติดไปกับ git

---

*→ บทถัดไป: **Part 8 — Logging** — Structured logging, Log Levels, Serilog integration และ Correlation ID สำหรับ tracing requests*
