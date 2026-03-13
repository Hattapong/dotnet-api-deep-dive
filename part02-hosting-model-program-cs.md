# Part 2 — Hosting Model & Program.cs

> **Deep Dive Series: .NET Web API Internals**  
> เราสร้าง HTTP server ด้วย `HttpListener` ได้แล้ว — แต่มันมีปัญหาเยอะ  
> บทนี้จะดูว่า .NET แก้ปัญหาเหล่านั้นด้วย **Host** อย่างไร

---

## 1. ปัญหาของ "โปรแกรมธรรมดา"

ก่อนจะพูดถึง Hosting Model ลองนึกถึงโปรแกรม console ที่เราเขียนกันปกติ:

```csharp
// Program.cs — console app แบบดั้งเดิม
Console.WriteLine("App started");

while (true)
{
    var input = Console.ReadLine();
    if (input == "quit") break;
    Console.WriteLine($"You said: {input}");
}

Console.WriteLine("App stopped");
```

โปรแกรมนี้มีปัญหาอะไรบ้างถ้านำไป production?

| ปัญหา | อาการ |
|-------|-------|
| **ไม่มี Configuration** | hardcode ทุกอย่าง หรือ parse args เอง |
| **ไม่มี Logging** | `Console.WriteLine` ไม่มี level, ไม่มี structured format |
| **ไม่มี Graceful Shutdown** | Ctrl+C ฆ่าโปรแกรมทันที ไม่รอให้งานเสร็จ |
| **ไม่มี Dependency Management** | `new()` เองทุกที่ |
| **ไม่มี Background Services** | ต้องเขียน thread management เอง |
| **ไม่มี Health Check** | ไม่รู้ว่า app ยังทำงานปกติหรือไม่ |

.NET แก้ทั้งหมดนี้ด้วยสิ่งที่เรียกว่า **Generic Host**

---

## 2. แนวคิดของ IHost — "กล่องที่ดูแล App ให้"

**Host** คือ object ที่ทำหน้าที่เป็น **lifecycle manager** ของโปรแกรม

แทนที่จะเขียน `while(true)` เองและจัดการทุกอย่างเอง เราบอก Host ว่า:
- App นี้มี services อะไรบ้าง
- มี background jobs อะไรที่ต้องรัน
- Configuration มาจากไหน
- Log ไปที่ไหน

แล้ว Host จะจัดการ **start → run → stop** ให้เราเอง

```
┌─────────────────────────────────────────────────────┐
│                     IHost                           │
│                                                     │
│  ┌─────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │   Services  │  │   Config   │  │   Logging   │  │
│  │  (IService  │  │ (appset-   │  │ (ILogger<T> │  │
│  │  Collection)│  │  tings)    │  │  Console    │  │
│  └─────────────┘  └────────────┘  │  File, etc) │  │
│                                   └─────────────┘  │
│  ┌──────────────────────────────────────────────┐   │
│  │         IHostedService (Background Jobs)     │   │
│  │   WorkerService1 │ WorkerService2 │ ...      │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  Lifecycle: Start ──► Running ──► Stop (graceful)   │
└─────────────────────────────────────────────────────┘
```

### Interface สำคัญที่ต้องรู้

```csharp
// ตัว Host หลัก
public interface IHost : IDisposable
{
    IServiceProvider Services { get; }   // DI Container
    Task StartAsync(CancellationToken cancellationToken = default);
    Task StopAsync(CancellationToken cancellationToken = default);
}

// สร้าง Host
public interface IHostBuilder
{
    IHostBuilder ConfigureServices(Action<HostBuilderContext, IServiceCollection> configureDelegate);
    IHostBuilder ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate);
    IHostBuilder ConfigureLogging(Action<HostBuilderContext, ILoggingBuilder> configureDelegate);
    IHost Build();
}

// Background Service ที่รันใน Host
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

---

## 3. จาก Console App → IHost App

### 3.1 โปรแกรมตั้งต้น — Console App ธรรมดา

สมมติเรามี echo app ที่รับ input แล้ว print ออกมา:

```csharp
// ❌ Before: Console App ธรรมดา
// Program.cs

Console.WriteLine("=== Echo App Started ===");
Console.WriteLine("Type something and press Enter. Type 'quit' to exit.");

var messageCount = 0;

while (true)
{
    var input = Console.ReadLine();

    if (input == null || input.ToLower() == "quit")
        break;

    messageCount++;
    Console.WriteLine($"[{messageCount}] Echo: {input}");
}

Console.WriteLine($"=== App Stopped. Total messages: {messageCount} ===");
```

**ปัญหา:** ถ้าต้องการเพิ่ม config file, logging, หรือ background timer ต้องเขียนเองทั้งหมด

---

### 3.2 แปลงเป็น IHost App — ขั้นต่ำสุด

```csharp
// ✅ After: IHost App
// Program.cs

using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .Build();

await host.RunAsync();
```

แค่นี้ยังไม่ได้ทำอะไรพิเศษ แต่เราได้มา **ฟรี** โดยไม่ต้องเขียนเอง:
- Configuration จาก `appsettings.json` + environment variables + command-line args
- Structured logging ออก console
- Graceful shutdown เมื่อกด Ctrl+C (SIGTERM)
- DI Container พร้อมใช้

---

### 3.3 ย้าย Logic เข้า IHostedService

เพื่อให้ Host จัดการ lifecycle ให้ เราต้องใส่ logic ของเราใน `IHostedService`:

```csharp
// EchoService.cs
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class EchoService : IHostedService
{
    private readonly ILogger<EchoService> _logger;
    private          Task?               _runningTask;
    private          CancellationToken   _stoppingToken;

    public EchoService(ILogger<EchoService> logger)
    {
        _logger = logger;
    }

    // Host เรียก StartAsync เมื่อ app เริ่มต้น
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _stoppingToken = cancellationToken;
        _logger.LogInformation("EchoService starting...");

        // เริ่ม background task โดยไม่ await
        // เพื่อให้ StartAsync return เร็วๆ
        _runningTask = RunAsync();

        return Task.CompletedTask;
    }

    private async Task RunAsync()
    {
        _logger.LogInformation("=== Echo App Ready. Type something. Type 'quit' to exit ===");

        var messageCount = 0;

        while (!_stoppingToken.IsCancellationRequested)
        {
            // อ่าน input แบบ async
            var input = await Task.Run(() => Console.ReadLine(), _stoppingToken);

            if (input == null || input.ToLower() == "quit")
            {
                // ขอให้ Host หยุด app ทั้งหมด
                Environment.Exit(0);
                break;
            }

            messageCount++;
            _logger.LogInformation("[{Count}] Echo: {Input}", messageCount, input);
        }
    }

    // Host เรียก StopAsync เมื่อ app กำลังจะหยุด (Ctrl+C หรือ SIGTERM)
    public async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("EchoService stopping gracefully...");

        if (_runningTask is not null)
            await Task.WhenAny(_runningTask, Task.Delay(Timeout.Infinite, cancellationToken));

        _logger.LogInformation("EchoService stopped.");
    }
}
```

```csharp
// Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<EchoService>();
    })
    .Build();

await host.RunAsync();
```

ตอนนี้ถ้ากด Ctrl+C จะเห็น log "stopping gracefully..." ก่อน app จะปิด ไม่ใช่ตายทันที

---

### 3.4 BackgroundService — Abstract Class ที่ง่ายกว่า

เพราะ pattern `StartAsync` + `RunAsync` + `StopAsync` ซ้ำกันทุกครั้ง .NET มี abstract class `BackgroundService` ให้:

```csharp
// EchoService.cs — เวอร์ชันที่สะอาดกว่า
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class EchoService : BackgroundService  // ← ไม่ต้อง implement IHostedService เอง
{
    private readonly ILogger<EchoService> _logger;

    public EchoService(ILogger<EchoService> logger)
    {
        _logger = logger;
    }

    // implement แค่ method เดียว
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("=== Echo App Ready ===");

        var messageCount = 0;

        while (!stoppingToken.IsCancellationRequested)
        {
            string? input;

            try
            {
                input = await Task.Run(Console.ReadLine, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;  // Ctrl+C กด → loop จบ
            }

            if (input == null || input.Equals("quit", StringComparison.OrdinalIgnoreCase))
                break;

            messageCount++;
            _logger.LogInformation("[{Count}] Echo: {Input}", messageCount, input);
        }

        _logger.LogInformation("Total messages processed: {Count}", messageCount);
    }
}
```

`BackgroundService` จัดการ `StartAsync` และ `StopAsync` ให้เราแล้ว เราต้องเขียนแค่ `ExecuteAsync`

---

## 4. Feature ที่ IHost ให้มาฟรี

### 4.1 Configuration — อ่าน appsettings.json โดยอัตโนมัติ

`Host.CreateDefaultBuilder` อ่าน configuration จากหลายแหล่งตามลำดับ priority:

```
Priority (สูง → ต่ำ):
  1. Command-line args         --MyApp:Name=Alice
  2. Environment Variables     MyApp__Name=Alice
  3. appsettings.{Env}.json   appsettings.Production.json
  4. appsettings.json         appsettings.json
  5. Default values
```

ค่าที่อยู่สูงกว่าจะ **override** ค่าที่ต่ำกว่า

```json
// appsettings.json
{
  "EchoApp": {
    "Prefix": "ECHO",
    "MaxMessages": 100
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  }
}
```

```csharp
// EchoOptions.cs — strongly-typed config
public class EchoOptions
{
    public string Prefix      { get; set; } = "ECHO";
    public int    MaxMessages { get; set; } = 100;
}
```

```csharp
// Program.cs
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        // Bind section "EchoApp" → EchoOptions
        services.Configure<EchoOptions>(
            context.Configuration.GetSection("EchoApp"));

        services.AddHostedService<EchoService>();
    })
    .Build();

await host.RunAsync();
```

```csharp
// EchoService.cs — รับ config ผ่าน IOptions<T>
using Microsoft.Extensions.Options;

public class EchoService : BackgroundService
{
    private readonly ILogger<EchoService> _logger;
    private readonly EchoOptions          _options;

    public EchoService(ILogger<EchoService> logger, IOptions<EchoOptions> options)
    {
        _logger  = logger;
        _options = options.Value;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("=== Echo App Ready (Prefix: {Prefix}) ===", _options.Prefix);

        var messageCount = 0;

        while (!stoppingToken.IsCancellationRequested && messageCount < _options.MaxMessages)
        {
            string? input;
            try { input = await Task.Run(Console.ReadLine, stoppingToken); }
            catch (OperationCanceledException) { break; }

            if (input == null || input.Equals("quit", StringComparison.OrdinalIgnoreCase))
                break;

            messageCount++;
            _logger.LogInformation("[{Prefix}][{Count}/{Max}] {Input}",
                _options.Prefix, messageCount, _options.MaxMessages, input);
        }

        _logger.LogInformation("Session ended. Messages: {Count}", messageCount);
    }
}
```

ทีนี้เปลี่ยน config โดยไม่ต้องแตะโค้ดได้เลย:

```bash
# เปลี่ยน prefix ผ่าน environment variable
MyApp__EchoApp__Prefix=HELLO dotnet run

# หรือผ่าน command-line
dotnet run --EchoApp:Prefix=HI --EchoApp:MaxMessages=5
```

---

### 4.2 Logging — Structured Logging พร้อมใช้

เมื่อใช้ `ILogger<T>` แทน `Console.WriteLine` เราได้:

```csharp
// ❌ Console.WriteLine — ข้อมูลสูญหายเป็น string ธรรมดา
Console.WriteLine($"[{count}] Echo: {input}");
// output: "[5] Echo: hello"

// ✅ ILogger — structured data ที่ query ได้
_logger.LogInformation("[{Count}] Echo: {Input}", count, input);
// output: info: EchoService[0]
//              [5] Echo: hello
// แต่ถ้าส่งไป Seq/OpenTelemetry จะเห็น Count=5, Input="hello" เป็น field แยกกัน
```

```csharp
// Log Levels — ใช้ให้ถูก level
_logger.LogTrace("Very detailed debug info");      // ละเอียดสุด, dev only
_logger.LogDebug("Entering method X with id={Id}", id);
_logger.LogInformation("User {UserId} logged in", userId);  // flow ปกติ
_logger.LogWarning("Retry attempt {Count} of {Max}", count, max);
_logger.LogError(ex, "Failed to process message {MessageId}", msgId);
_logger.LogCritical("Database connection pool exhausted!");  // ร้ายแรงสุด
```

filter log level ใน `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default":        "Information",
      "EchoService":    "Debug",
      "Microsoft":      "Warning",
      "System":         "Warning"
    }
  }
}
```

---

### 4.3 Graceful Shutdown — ไม่ตายกลางทาง

เปรียบเทียบ:

```csharp
// ❌ Console App ธรรมดา
// Ctrl+C → process ถูก kill ทันที
// งานที่ค้างอยู่? หายหมด

// ✅ IHost App
// Ctrl+C → Host รับ SIGTERM → เรียก StopAsync บน services ทุกตัว
// แต่ละ service มีเวลา graceful shutdown (default 5 วินาที)
// แล้วค่อยปิด process
```

```csharp
// ดูว่า Host ทำอะไรเมื่อ Ctrl+C กด
// (ไม่ต้องเขียนเอง Host ทำให้)

// 1. OS ส่ง SIGTERM/SIGINT
// 2. Host เรียก IHostApplicationLifetime.StopApplication()
// 3. CancellationToken ใน ExecuteAsync ถูก cancel
// 4. Host รอให้ ExecuteAsync return (ภายใน timeout)
// 5. Host เรียก IHostedService.StopAsync บนทุก service
// 6. Host.Dispose()
// 7. Process exit
```

ปรับ shutdown timeout:

```csharp
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        // ให้เวลา graceful shutdown 30 วินาที
        services.Configure<HostOptions>(options =>
        {
            options.ShutdownTimeout = TimeSpan.FromSeconds(30);
        });

        services.AddHostedService<EchoService>();
    })
    .Build();
```

---

### 4.4 Multiple Background Services

Host รองรับ services หลายตัวพร้อมกัน:

```csharp
// HeartbeatService.cs — ping ทุก 10 วินาที
public class HeartbeatService : BackgroundService
{
    private readonly ILogger<HeartbeatService> _logger;

    public HeartbeatService(ILogger<HeartbeatService> logger) => _logger = logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("💓 Heartbeat — app is alive at {Time}", DateTimeOffset.Now);
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

```csharp
// Program.cs — รัน 2 services พร้อมกัน
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<EchoService>();
        services.AddHostedService<HeartbeatService>();  // รันคู่กันเลย
    })
    .Build();

await host.RunAsync();
```

Host จะ start ทุก `IHostedService` พร้อมกัน และ stop ตาม LIFO order (ตัวสุดท้ายที่ register หยุดก่อน)

---

## 5. Flow ใน Program.cs — ก่อนและหลัง Build()

นี่คือส่วนที่คนมักสับสน มาดูกันชัดๆ ว่าแต่ละ section ทำหน้าที่อะไร

```csharp
// Program.cs — annotated version

// ══════════════════════════════════════════════════════════
// PHASE 1: BUILDER PHASE — ยังไม่ได้สร้าง host จริงๆ
// ทุกอย่างที่เขียนตรงนี้คือ "การตั้งค่า" ล้วนๆ
// ไม่มี service ถูกสร้าง, ไม่มี config ถูกอ่านจริงๆ
// ══════════════════════════════════════════════════════════

var builder = Host.CreateDefaultBuilder(args);
//                 └── สร้าง HostBuilder พร้อม defaults:
//                     - appsettings.json
//                     - environment variables
//                     - console logging
//                     - DI container (empty)

builder.ConfigureServices((context, services) =>
{
    // context.Configuration → อ่าน config ได้แล้ว (build แล้ว)
    // context.HostingEnvironment → Development/Production/etc.

    // ตรงนี้แค่ "ลงทะเบียน" ว่าจะมี services อะไร
    // ยังไม่ได้ new() อะไรทั้งนั้น!
    services.AddHostedService<EchoService>();
    services.AddSingleton<IMyService, MyService>();
});

builder.ConfigureLogging((context, logging) =>
{
    logging.ClearProviders();
    logging.AddConsole();
    logging.AddDebug();
    // ยังไม่มี ILogger ที่ใช้ได้จริงตรงนี้
});

builder.ConfigureAppConfiguration((context, config) =>
{
    // เพิ่ม config source เพิ่มเติม
    config.AddJsonFile("custom-settings.json", optional: true);
    config.AddEnvironmentVariables(prefix: "MYAPP_");
});

// ══════════════════════════════════════════════════════════
// PHASE 2: BUILD — จุดที่ทุกอย่างถูก "คอมไพล์" จริงๆ
// ══════════════════════════════════════════════════════════

var host = builder.Build();
//              ↑
// ณ จุดนี้:
// ✅ IConfiguration ถูกสร้าง (อ่าน appsettings.json จริงๆ)
// ✅ IServiceProvider ถูกสร้าง (validate registrations)
// ✅ ILogger factory พร้อม
// ✅ IHostEnvironment พร้อม
// ❌ IHostedService ยัง start ไม่ได้
// ❌ โปรแกรมยังไม่รัน

// ══════════════════════════════════════════════════════════
// PHASE 3: RUN — เริ่มต้น app จริงๆ
// ══════════════════════════════════════════════════════════

await host.RunAsync();
// ↑
// 1. เรียก StartAsync บน host
// 2. Host เรียก StartAsync บนทุก IHostedService
// 3. รอจนกว่าจะได้ signal หยุด (Ctrl+C / SIGTERM)
// 4. เรียก StopAsync บนทุก IHostedService (reverse order)
// 5. Dispose host
// 6. return
```

### ลำดับการทำงานภายใน Build()

```
builder.Build() ทำ:

1. Build IConfiguration
   └── อ่าน appsettings.json
   └── อ่าน appsettings.{Environment}.json
   └── อ่าน environment variables
   └── อ่าน command-line args
   └── merge ทุกอย่างตาม priority

2. Build IServiceProvider
   └── รวบรวม registrations ทั้งหมด
   └── validate scopes (ถ้าเปิด ValidateScopes)
   └── สร้าง root container

3. สร้าง IHost
   └── inject IServiceProvider, IConfiguration, ILogger เข้าไป

4. return IHost (ยังไม่ start อะไร)
```

### ลำดับการทำงานภายใน RunAsync()

```
host.RunAsync() ทำ:

1. StartAsync(CancellationToken)
   ├── เรียก IHostedService.StartAsync ตามลำดับ registration
   │   ├── EchoService.StartAsync()
   │   └── HeartbeatService.StartAsync()
   └── log "Application started"

2. WaitForShutdownAsync()
   └── block จนกว่าจะได้ signal
       └── Ctrl+C หรือ SIGTERM หรือ Environment.Exit()

3. StopAsync(CancellationToken)
   ├── เรียก IHostedService.StopAsync แบบ reverse order
   │   ├── HeartbeatService.StopAsync()  ← หยุดก่อน
   │   └── EchoService.StopAsync()
   └── log "Application stopped"

4. host.Dispose()
```

---

## 6. เปรียบเทียบกับ WebApplicationBuilder

ถึงตรงนี้เรารู้จัก `Host.CreateDefaultBuilder` สำหรับ console/worker apps แล้ว

ใน ASP.NET Core มีอีกตัวที่เฉพาะสำหรับ Web:

### WebApplicationBuilder vs IHostBuilder

```csharp
// ── Worker / Console App ──────────────────────────────
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<MyWorker>();
    })
    .Build();

await host.RunAsync();


// ── Web API ───────────────────────────────────────────
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

await app.RunAsync();
```

ดูเผินๆ เหมือนกัน แต่ข้างในต่างกันมาก:

| | `Host.CreateDefaultBuilder` | `WebApplication.CreateBuilder` |
|---|---|---|
| **เหมาะกับ** | Worker, Console, Background jobs | Web API, MVC, Minimal API |
| **เพิ่มเติม** | — | Kestrel, HTTP pipeline, Routing |
| **Builder type** | `IHostBuilder` | `WebApplicationBuilder` |
| **Result** | `IHost` | `WebApplication` |
| **Middleware** | ❌ ไม่มี | ✅ `app.Use(...)` |
| **Routing** | ❌ ไม่มี | ✅ `app.Map(...)` |
| **IHostedService** | ✅ | ✅ (inherited) |
| **Configuration** | ✅ | ✅ (same) |
| **Logging** | ✅ | ✅ (same) |

`WebApplication` คือ `IHost` + Web Server + HTTP Pipeline รวมกัน

---

### 6.1 โครงสร้าง Program.cs แบบ Web API

```csharp
// Program.cs — Web API พร้อม annotation ครบ

// ══════════════════════════════════════════════════════════
// PHASE 1: BUILDER PHASE
// ══════════════════════════════════════════════════════════

var builder = WebApplication.CreateBuilder(args);
//                            └── CreateDefaultBuilder +
//                                AddKestrel +
//                                AddRouting +
//                                AddControllers (ถ้าเรียก)

// ─── Services ───────────────────────────────────────────
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// สามารถเพิ่ม custom services ได้เหมือนกัน
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddHostedService<CleanupWorker>();   // background job ก็ได้

// ─── Configuration ──────────────────────────────────────
// builder.Configuration ← อ่าน config ได้เลย (ถูก build ไปแล้วตั้งแต่ CreateBuilder)
var connStr = builder.Configuration.GetConnectionString("Default");

// ─── Logging ────────────────────────────────────────────
builder.Logging.ClearProviders();
builder.Logging.AddConsole();

// ─── Kestrel (Web Server) ────────────────────────────────
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenLocalhost(5000);   // HTTP
    options.ListenLocalhost(5001, o => o.UseHttps());  // HTTPS
});

// ══════════════════════════════════════════════════════════
// PHASE 2: BUILD
// ══════════════════════════════════════════════════════════

var app = builder.Build();
//           └── IConfiguration built
//               IServiceProvider built
//               Kestrel configured
//               HTTP pipeline NOT configured yet

// ══════════════════════════════════════════════════════════
// PHASE 3: CONFIGURE HTTP PIPELINE
// (เฉพาะ WebApplication — IHost ธรรมดาไม่มีตรงนี้)
// ══════════════════════════════════════════════════════════

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// ─── Endpoints ──────────────────────────────────────────
app.MapControllers();

// Minimal API
app.MapGet("/health", () => new { status = "healthy", time = DateTime.UtcNow });

// ══════════════════════════════════════════════════════════
// PHASE 4: RUN
// ══════════════════════════════════════════════════════════

await app.RunAsync();
// ↑ เหมือน IHost.RunAsync เลย แต่ยังเริ่ม Kestrel ด้วย
```

---

### 6.2 Flow เปรียบเทียบแบบ Side-by-Side

```
IHost (Worker App)              WebApplication (Web API)
─────────────────────────────   ────────────────────────────────
Host.CreateDefaultBuilder()     WebApplication.CreateBuilder()
       │                                   │
ConfigureServices()             builder.Services.Add...()
       │                        builder.Logging...
       │                        builder.WebHost...
       │                                   │
   .Build()          ══════►        .Build()
       │                                   │
       │                         ┌── Configure Pipeline ──┐
       │                         │  app.UseMiddleware()   │
       │                         │  app.UseRouting()      │
       │                         │  app.UseAuth()         │
       │                         └────────────────────────┘
       │                                   │
       │                           app.MapGet()
       │                           app.MapControllers()
       │                                   │
  host.RunAsync()    ══════►      app.RunAsync()
       │                                   │
  Start IHostedServices          Start Kestrel (port 5000)
       │                         Start IHostedServices
  Wait for SIGTERM               Accept HTTP Requests
       │                         Wait for SIGTERM
  Stop gracefully                Stop gracefully
```

---

### 6.3 ข้างในจริงๆ — WebApplication คือ IHost

`WebApplication` implement `IHost`:

```csharp
// ดูจาก ASP.NET Core source (simplified)
public sealed class WebApplication : IHost, IApplicationBuilder, IEndpointRouteBuilder
{
    private readonly IHost _host;  // ← wrapped IHost ข้างใน!

    // delegate ไปที่ _host ทุกอย่าง
    public IServiceProvider Services => _host.Services;
    public Task StartAsync(CancellationToken ct) => _host.StartAsync(ct);
    public Task StopAsync(CancellationToken ct)  => _host.StopAsync(ct);

    // เพิ่มของ web มา
    public IConfiguration Configuration { get; }
    public IWebHostEnvironment Environment { get; }
    public ILogger Logger { get; }

    // IApplicationBuilder — configure HTTP pipeline
    public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware) { ... }

    // IEndpointRouteBuilder — define endpoints
    public IEndpointRouteBuilder MapGet(string pattern, ...) { ... }
}
```

ดังนั้น `WebApplication` คือ **IHost ที่มีความสามารถ web เพิ่มมา** ไม่ใช่คนละเรื่องกัน

---

## 7. ตัวอย่างสมบูรณ์ — Echo App vs Todo API

เพื่อให้เห็นภาพชัด นี่คือ 2 โปรแกรมที่ใช้ hosting model เดียวกัน:

```csharp
// ═══ Echo Worker App (IHost) ═══════════════════════════════

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((ctx, services) =>
    {
        services.Configure<EchoOptions>(ctx.Configuration.GetSection("Echo"));
        services.AddHostedService<EchoService>();
        services.AddHostedService<HeartbeatService>();
    })
    .Build();

await host.RunAsync();

// ═══ Todo Web API (WebApplication) ═════════════════════════

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<List<Todo>>();  // in-memory store

var app = builder.Build();

var todos = app.Services.GetRequiredService<List<Todo>>();

app.MapGet("/todos",       ()        => todos);
app.MapGet("/todos/{id}", (int id)   => todos.Find(t => t.Id == id) is { } t ? Results.Ok(t) : Results.NotFound());
app.MapPost("/todos",     (Todo todo) => { todos.Add(todo); return Results.Created($"/todos/{todo.Id}", todo); });
app.MapDelete("/todos/{id}", (int id) =>
{
    var todo = todos.Find(t => t.Id == id);
    if (todo is null) return Results.NotFound();
    todos.Remove(todo);
    return Results.NoContent();
});

await app.RunAsync();

record Todo(int Id, string Title, bool IsCompleted = false);
```

ทั้งสองโปรแกรมมี lifecycle เหมือนกัน ต่างกันที่ worker ไม่มี HTTP pipeline

---

## บทสรุป

สิ่งที่เรียนรู้ในบทนี้:

- **IHost** คือ lifecycle manager ที่จัดการ Start/Stop, Configuration, Logging, DI ให้เรา
- **IHostedService** / **BackgroundService** คือวิธีใส่ logic ของเราเข้าไปใน Host
- **Program.cs มี 4 phases**: Builder → ConfigureServices → Build → Run
- หลัง `Build()` เราได้ IHost ที่พร้อมใช้ แต่ยังไม่ได้ start
- **WebApplication** คือ IHost + Kestrel + HTTP Pipeline — ไม่ใช่คนละเรื่อง
- ก่อน `Build()` คือ "ลงทะเบียน" — หลัง `Build()` คือ "configure pipeline" — หลัง `Run()` คือ "ทำงานจริง"

---

## Workshop

1. **Lab 2.1** — สร้าง Worker App ที่รัน TimerService ซึ่ง log เวลาทุก 5 วินาที พร้อมอ่าน interval จาก `appsettings.json`

2. **Lab 2.2** — แก้ให้ TimerService หยุดนับเมื่อได้รับ Ctrl+C แล้ว log สรุปว่า tick ไปกี่ครั้งก่อนหยุด (graceful shutdown)

3. **Lab 2.3** — แปลง Todo API ข้างบนให้ใช้ `appsettings.json` เก็บ `ApiTitle` และ `Version` แล้วแสดงผ่าน `GET /info`

4. **Lab 2.4 (Bonus)** — สร้าง project ที่มีทั้ง WebApplication (HTTP API) และ BackgroundService (HeartbeatService) รันอยู่ใน host เดียวกัน

---

*→ บทถัดไป: **Part 3 — Dependency Injection** — ทำไม `services.AddScoped<T>()` ถึงทำงานได้ และ DI Container สร้างและจัดการ objects ยังไงข้างใน*
