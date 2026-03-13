# Part 1 — Web Server & Web Application คืออะไร

> **Deep Dive Series: .NET Web API Internals**  
> ก่อนที่จะเข้าใจ ASP.NET Core ได้จริง เราต้องเข้าใจสิ่งที่มันสร้างขึ้นมาแก้ปัญหาก่อน

---

## 1. Web Server คืออะไร?

**Web Server** คือโปรแกรมที่ทำหน้าที่ **รับ HTTP Request** และ **ส่ง HTTP Response** กลับไปหา Client

เมื่อคุณพิมพ์ `https://example.com` ในเบราว์เซอร์ สิ่งที่เกิดขึ้นคือ:

1. เบราว์เซอร์สร้าง HTTP Request ขึ้นมา
2. ส่งผ่านเครือข่ายไปยัง IP address ของ server
3. Web Server ที่ port 80/443 รับ request นั้น
4. ส่ง HTTP Response กลับมา (HTML, JSON, ไฟล์ ฯลฯ)

ตัวอย่าง Web Server ที่รู้จักกันดี:

| Web Server | ผู้พัฒนา | จุดเด่น |
|---|---|---|
| **IIS** | Microsoft | ฝังใน Windows, จัดการ Windows Auth |
| **Nginx** | Igor Sysoev | High-performance reverse proxy |
| **Apache** | Apache Foundation | ยืดหยุ่น, module-based |
| **Kestrel** | Microsoft | built-in ใน ASP.NET Core, cross-platform |

Web Server แบบดั้งเดิม (เช่น IIS, Nginx, Apache) ทำได้แค่:

- เสิร์ฟไฟล์ static (HTML, CSS, JS, รูปภาพ)
- Forward request ไปยัง application อื่น
- Handle SSL/TLS termination
- Load balancing / reverse proxy

แต่ทำ **ไม่ได้** เองโดยตรง:
- รัน business logic
- เชื่อมต่อ database
- ตรวจสอบ authentication แบบ custom
- ประมวลผล JSON ตาม routing rules ที่ซับซ้อน

นั่นคือที่มาของ **Web Application**

---

## 2. Web Application คืออะไร?

**Web Application** คือโปรแกรมที่รัน logic และสร้าง response แบบ dynamic ตาม request ที่ได้รับ

ต่างจาก Web Server ตรงที่:

```
Web Server:
  GET /logo.png  →  อ่านไฟล์ logo.png  →  ส่งกลับ

Web Application:
  GET /users/42  →  รัน logic → query DB → สร้าง JSON → ส่งกลับ
```

Web Application ต้องการ Web Server เพื่อรับ request มาก่อน แล้วค่อยส่งต่อให้ application ประมวลผล ซึ่งเรียกรูปแบบนี้ว่า **reverse proxy** หรือ **application hosting**

```
Browser
   │
   │  HTTP Request
   ▼
[ Nginx / IIS ]        ← Web Server (รับ connection, handle SSL)
   │
   │  forward request
   ▼
[ .NET App / Node / Python ]  ← Web Application (รัน logic)
   │
   │  response
   ▼
[ Nginx / IIS ]
   │
   ▼
Browser
```

ใน ASP.NET Core สมัยใหม่ Kestrel สามารถทำหน้าที่ได้ทั้งสองอย่าง แต่ในระบบ production มักใช้ Nginx หรือ IIS ด้านหน้าเสมอ

---

## 3. HTTP Request / Response — ถึงระดับโครงสร้าง

HTTP (HyperText Transfer Protocol) คือ **ข้อตกลงภาษา** ระหว่าง Client และ Server ว่าจะคุยกันด้วย format อะไร

### 3.1 โครงสร้าง HTTP Request

```
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer eyJhbGci...
Content-Length: 42

{"name":"Alice","email":"alice@example.com"}
```

แบ่งออกเป็น 3 ส่วนหลัก:

#### ส่วนที่ 1 — Request Line (บรรทัดแรก)

```
POST /api/users HTTP/1.1
^^^^ ^^^^^^^^^^^ ^^^^^^^^
 │        │          └── HTTP Version (1.1, 2, 3)
 │        └───────────── Request Target (path + query string)
 └────────────────────── HTTP Method
```

**HTTP Methods ที่ต้องรู้:**

| Method | ความหมายตาม spec | Idempotent | Safe |
|--------|-----------------|:----------:|:----:|
| GET | ดึงข้อมูล | ✅ | ✅ |
| POST | สร้างข้อมูลใหม่ | ❌ | ❌ |
| PUT | แทนที่ทั้งหมด | ✅ | ❌ |
| PATCH | แก้ไขบางส่วน | ❌ | ❌ |
| DELETE | ลบ | ✅ | ❌ |
| HEAD | เหมือน GET แต่ไม่มี body | ✅ | ✅ |
| OPTIONS | ถามว่า endpoint รับ method อะไรได้บ้าง | ✅ | ✅ |

> **Idempotent** = เรียกซ้ำ n ครั้ง ผลลัพธ์เหมือนเรียกครั้งเดียว  
> **Safe** = ไม่มีผลข้างเคียง (ไม่เปลี่ยนแปลง state บน server)

#### ส่วนที่ 2 — Headers

```
Host: example.com
Content-Type: application/json
Authorization: Bearer eyJhbGci...
Content-Length: 42
```

Headers คือ **metadata** ของ request ในรูปแบบ `Key: Value` แต่ละบรรทัด

Headers สำคัญที่ต้องรู้:

| Header | หน้าที่ |
|--------|---------|
| `Host` | บอกว่าต้องการ domain ไหน (สำคัญมากใน virtual hosting) |
| `Content-Type` | บอก format ของ request body |
| `Accept` | บอกว่า client รับ format อะไรได้บ้าง |
| `Authorization` | ส่ง credentials / token |
| `Content-Length` | ขนาด body เป็น bytes |
| `Transfer-Encoding: chunked` | ส่ง body แบบแบ่ง chunk |
| `Cookie` | ส่ง cookie ที่มีอยู่กลับไป |
| `User-Agent` | บอกว่า client คือโปรแกรมอะไร |

#### ส่วนที่ 3 — Body (Optional)

```json
{"name":"Alice","email":"alice@example.com"}
```

- มีได้ใน POST, PUT, PATCH
- GET, DELETE โดย spec ไม่ควรมี body (แม้ technically ทำได้)
- แยกจาก headers ด้วย **blank line** หนึ่งบรรทัด (`\r\n\r\n`)

---

### 3.2 โครงสร้าง HTTP Response

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/99
Date: Thu, 12 Mar 2026 10:00:00 GMT
Content-Length: 31

{"id":99,"name":"Alice"}
```

#### ส่วนที่ 1 — Status Line

```
HTTP/1.1 201 Created
^^^^^^^^ ^^^ ^^^^^^^
    │     │     └── Reason Phrase (อธิบาย status code)
    │     └──────── Status Code
    └────────────── HTTP Version
```

**Status Codes กลุ่มหลัก:**

| กลุ่ม | ความหมาย | ตัวอย่าง |
|-------|----------|---------|
| `1xx` | Informational | 100 Continue |
| `2xx` | Success | 200 OK, 201 Created, 204 No Content |
| `3xx` | Redirection | 301 Moved, 302 Found, 304 Not Modified |
| `4xx` | Client Error | 400 Bad Request, 401, 403, 404, 422 |
| `5xx` | Server Error | 500 Internal Error, 502 Bad Gateway, 503 |

#### ส่วนที่ 2 — Response Headers

```
Content-Type: application/json
Location: /api/users/99
Date: Thu, 12 Mar 2026 10:00:00 GMT
```

| Header | หน้าที่ |
|--------|---------|
| `Content-Type` | บอก format ของ response body |
| `Location` | URL ของ resource ที่เพิ่งสร้าง (ใช้กับ 201) |
| `Cache-Control` | บอก client ว่า cache ได้ไหม นานแค่ไหน |
| `Set-Cookie` | ตั้งค่า cookie ใน browser |
| `WWW-Authenticate` | บอก client ว่าต้อง auth ด้วย scheme อะไร |
| `Access-Control-Allow-Origin` | CORS policy |

#### ส่วนที่ 3 — Body

```json
{"id":99,"name":"Alice"}
```

---

### 3.3 HTTP ใน TCP — ของจริงที่วิ่งบน Network

HTTP ทั้งหมดนี้ถูกส่งไปเป็น **bytes ธรรมดา** บน TCP connection

```
TCP Stream (bytes):
50 4F 53 54 20 2F 61 70 69 2F 75 73 65 72 73 20
48 54 54 50 2F 31 2E 31 0D 0A 48 6F 73 74 3A 20
...

ถอดรหัสได้:
"POST /api/users HTTP/1.1\r\n"
"Host: example.com\r\n"
"\r\n"
"{"name":"Alice"}"
```

HTTP/1.1 ใช้ `\r\n` (CRLF) เป็น line separator และใช้ `\r\n\r\n` เป็น ตัวแบ่ง header กับ body

---

## 4. PORT คืออะไร?

### Network Address = IP + Port

IP Address บอกว่า **เครื่องไหน** แต่ PORT บอกว่า **โปรแกรมไหน** บนเครื่องนั้น

```
Server มี IP: 203.0.113.10
                     │
          ┌──────────┼──────────┐
          │          │          │
        :80         :443       :5432
      (HTTP)      (HTTPS)   (PostgreSQL)
          │          │
       Nginx       Nginx
          │
        :5000
      (.NET App)
```

Port คือ **ตัวเลข 16-bit** (0–65535) ที่ OS ใช้เพื่อรู้ว่า incoming connection ควรส่งต่อไปยัง process ไหน

### Port ที่สำคัญ (Well-Known Ports)

| Port | โปรโตคอล | ใช้กับอะไร |
|------|----------|-----------|
| 80 | HTTP | Web traffic ปกติ |
| 443 | HTTPS | Web traffic เข้ารหัส |
| 22 | SSH | Remote shell |
| 21 | FTP | File transfer |
| 5432 | PostgreSQL | Database |
| 1433 | SQL Server | Database |
| 6379 | Redis | Cache / Message broker |
| 5000–5001 | (convention) | .NET dev server |

### Port กับ Web Server

เมื่อ Web Server **เปิด listen บน port** มันจะ:

1. บอก OS ว่า "port 443 เป็นของฉัน"
2. OS จะ forward ทุก TCP connection ที่เข้ามาที่ port 443 มาให้
3. Web Server รับ bytes, parse เป็น HTTP, ประมวลผล, ส่ง response กลับ

```
Client: 192.168.1.5:52341  →  203.0.113.10:443
         source port (random)    destination port
```

> **Source port** ของ client จะเป็นเลข random ที่ OS กำหนด (ephemeral port: 49152–65535)  
> **Destination port** คือ port ที่ server เปิด listen ไว้

### ทำไม Web Server ถึงมีข้อจำกัด?

Web Server ดั้งเดิมได้รับ HTTP request แล้ว mapping ไปยัง **ไฟล์บน disk** หรือ **forward ไปยัง app อื่น** แต่ทำไม่ได้เอง:

1. **Routing แบบ dynamic** — `/users/{id}` ที่ `{id}` เป็น pattern ไม่ใช่ไฟล์จริง
2. **Business Logic** — validate, transform, query database
3. **Middleware chain** — auth → logging → rate limit → handler
4. **Response generation** — สร้าง JSON/HTML แบบ runtime
5. **Stateful processing** — session, token validation, DI container

ทั้งหมดนี้ต้องการ **โปรแกรม** ไม่ใช่ web server

---

## 5. ตัวอย่าง — Web App จาก Scratch ด้วย HttpListener

เพื่อให้เห็นว่า Web Application **จัดการ request จริงๆ** อย่างไรโดยไม่พึ่ง ASP.NET Core เลย เราจะสร้าง HTTP server ด้วย `HttpListener` ซึ่งเป็น built-in class ใน .NET

`HttpListener` เป็น wrapper บน `TcpListener` ที่จัดการ HTTP parsing ให้เราแล้ว ทำให้เราโฟกัสที่ logic ได้โดยตรง

### 5.1 Server แบบง่ายที่สุด

```csharp
// SimpleHttpServer.cs
using System.Net;
using System.Text;

var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:5000/");
listener.Start();

Console.WriteLine("Server started at http://localhost:5000/");
Console.WriteLine("Press Ctrl+C to stop...");

while (true)
{
    // รอ request — blocking call
    HttpListenerContext context = await listener.GetContextAsync();

    HttpListenerRequest request   = context.Request;
    HttpListenerResponse response = context.Response;

    // อ่านข้อมูลจาก request
    Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] {request.HttpMethod} {request.Url?.PathAndQuery}");

    // สร้าง response
    string responseBody = "<h1>Hello from raw .NET!</h1>";
    byte[] buffer = Encoding.UTF8.GetBytes(responseBody);

    response.StatusCode  = 200;
    response.ContentType = "text/html; charset=utf-8";
    response.ContentLength64 = buffer.Length;

    await response.OutputStream.WriteAsync(buffer);
    response.Close();
}
```

รันแล้วเปิด browser ไปที่ `http://localhost:5000/` จะเห็น "Hello from raw .NET!"

---

### 5.2 เพิ่ม Manual Routing

ตอนนี้เราจะ implement routing เองโดยไม่ใช้ framework ใดๆ เพื่อดูว่า framework มันทำอะไรให้เราบ้าง

```csharp
// ManualRoutingServer.cs
using System.Net;
using System.Text;
using System.Text.Json;

var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:5000/");
listener.Start();

Console.WriteLine("Manual routing server started at http://localhost:5000/");

// จำลอง in-memory database
var users = new List<User>
{
    new(1, "Alice", "alice@example.com"),
    new(2, "Bob",   "bob@example.com"),
};

while (true)
{
    var context  = await listener.GetContextAsync();
    var request  = context.Request;
    var response = context.Response;

    var path   = request.Url?.AbsolutePath ?? "/";
    var method = request.HttpMethod;

    Console.WriteLine($"{method} {path}");

    // ─── Manual Router ────────────────────────────────────────
    if (method == "GET" && path == "/")
    {
        await WriteJson(response, new { message = "Welcome to Manual API" });
    }
    else if (method == "GET" && path == "/users")
    {
        await WriteJson(response, users);
    }
    else if (method == "GET" && path.StartsWith("/users/"))
    {
        // parse id จาก path เอง
        var segment = path["/users/".Length..];

        if (int.TryParse(segment, out int id))
        {
            var user = users.FirstOrDefault(u => u.Id == id);

            if (user is not null)
                await WriteJson(response, user);
            else
                await WriteJson(response, new { error = "User not found" }, statusCode: 404);
        }
        else
        {
            await WriteJson(response, new { error = "Invalid id" }, statusCode: 400);
        }
    }
    else if (method == "POST" && path == "/users")
    {
        // อ่าน request body
        using var reader     = new StreamReader(request.InputStream, request.ContentEncoding);
        string    bodyString = await reader.ReadToEndAsync();

        var newUser = JsonSerializer.Deserialize<UserCreateDto>(bodyString,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        if (newUser?.Name is null || newUser.Email is null)
        {
            await WriteJson(response, new { error = "Name and Email are required" }, statusCode: 400);
        }
        else
        {
            var created = new User(users.Max(u => u.Id) + 1, newUser.Name, newUser.Email);
            users.Add(created);

            response.Headers["Location"] = $"/users/{created.Id}";
            await WriteJson(response, created, statusCode: 201);
        }
    }
    else
    {
        await WriteJson(response, new { error = "Not Found" }, statusCode: 404);
    }
}

// ─── Helper ──────────────────────────────────────────────────
static async Task WriteJson(HttpListenerResponse response, object data, int statusCode = 200)
{
    var json   = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
    var buffer = Encoding.UTF8.GetBytes(json);

    response.StatusCode      = statusCode;
    response.ContentType     = "application/json; charset=utf-8";
    response.ContentLength64 = buffer.Length;

    await response.OutputStream.WriteAsync(buffer);
    response.Close();
}

// ─── Records ────────────────────────────────────────────────
record User(int Id, string Name, string Email);
record UserCreateDto(string? Name, string? Email);
```

ทดสอบด้วย curl:

```bash
# ดู users ทั้งหมด
curl http://localhost:5000/users

# ดู user คนเดียว
curl http://localhost:5000/users/1

# สร้าง user ใหม่
curl -X POST http://localhost:5000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'

# 404
curl http://localhost:5000/users/999
```

---

### 5.3 เพิ่ม Middleware-like Pattern

ลองเพิ่ม cross-cutting concerns เช่น logging และ timing โดยไม่มี framework ช่วย

```csharp
// MiddlewareLikeServer.cs
using System.Diagnostics;
using System.Net;
using System.Text;
using System.Text.Json;

var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:5000/");
listener.Start();

Console.WriteLine("Server with middleware-like pattern started at http://localhost:5000/");

while (true)
{
    var context  = await listener.GetContextAsync();
    var request  = context.Request;
    var response = context.Response;

    // ─── "Middleware" ที่ 1: Request Logging ─────────────────
    var requestId = Guid.NewGuid().ToString()[..8];
    var sw        = Stopwatch.StartNew();

    Console.WriteLine($"[{requestId}] --> {request.HttpMethod} {request.Url?.PathAndQuery}");

    // ─── "Middleware" ที่ 2: Simple API Key Auth ─────────────
    var apiKey = request.Headers["X-Api-Key"];

    if (string.IsNullOrEmpty(apiKey) || apiKey != "secret-key-123")
    {
        Console.WriteLine($"[{requestId}] 401 Unauthorized");
        await WriteJson(response, new { error = "Unauthorized" }, statusCode: 401);
        LogResponse(requestId, sw, 401);
        continue;  // short-circuit — ไม่ไปถึง handler
    }

    // ─── "Middleware" ที่ 3: Route to Handler ────────────────
    var path   = request.Url?.AbsolutePath ?? "/";
    var method = request.HttpMethod;

    int statusCode;

    if (method == "GET" && path == "/health")
    {
        await WriteJson(response, new
        {
            status    = "healthy",
            timestamp = DateTime.UtcNow,
            requestId
        });
        statusCode = 200;
    }
    else if (method == "GET" && path == "/api/data")
    {
        await WriteJson(response, new
        {
            items     = new[] { "item1", "item2", "item3" },
            requestId
        });
        statusCode = 200;
    }
    else
    {
        await WriteJson(response, new { error = "Not Found" }, statusCode: 404);
        statusCode = 404;
    }

    LogResponse(requestId, sw, statusCode);
}

static void LogResponse(string requestId, Stopwatch sw, int statusCode)
{
    sw.Stop();
    Console.WriteLine($"[{requestId}] <-- {statusCode} ({sw.ElapsedMilliseconds}ms)");
}

static async Task WriteJson(HttpListenerResponse response, object data, int statusCode = 200)
{
    var json   = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
    var buffer = Encoding.UTF8.GetBytes(json);

    response.StatusCode      = statusCode;
    response.ContentType     = "application/json; charset=utf-8";
    response.ContentLength64 = buffer.Length;

    await response.OutputStream.WriteAsync(buffer);
    response.Close();
}
```

ทดสอบ:

```bash
# ไม่มี API Key → 401
curl http://localhost:5000/api/data

# มี API Key → 200
curl http://localhost:5000/api/data -H "X-Api-Key: secret-key-123"

# health check
curl http://localhost:5000/health -H "X-Api-Key: secret-key-123"
```

Output ที่ควรเห็นใน console:

```
[a1b2c3d4] --> GET /api/data
[a1b2c3d4] <-- 200 (3ms)

[e5f6g7h8] --> GET /api/data
[e5f6g7h8] 401 Unauthorized
```

---

## 6. Pain Points ที่ทำให้ต้องมี Framework

จากโค้ดข้างบน เราจะเริ่มเห็นปัญหาชัดเจนเมื่อ application ใหญ่ขึ้น:

### ปัญหา 1 — Routing ยุ่งเหยิง

```csharp
// ทำแบบนี้ไปเรื่อยๆ ไม่ได้
if (method == "GET"  && path == "/users") { ... }
if (method == "POST" && path == "/users") { ... }
if (method == "GET"  && path.StartsWith("/users/")) { /* parse id manually */ }
if (method == "PUT"  && path.StartsWith("/users/")) { /* parse id manually อีกรอบ */ }
// ถ้ามี 50 endpoints ล่ะ?
```

### ปัญหา 2 — Model Binding ต้องทำเอง

```csharp
// parse + deserialize + validate ด้วยมือทุกครั้ง
using var reader = new StreamReader(request.InputStream, request.ContentEncoding);
string body = await reader.ReadToEndAsync();
var dto = JsonSerializer.Deserialize<UserCreateDto>(body, ...);
if (dto?.Name is null) { ... }
```

### ปัญหา 3 — Middleware เป็น if-else chain

```csharp
// auth check กระจัดกระจาย ไม่ reusable
if (string.IsNullOrEmpty(apiKey) || apiKey != "secret-key-123")
{
    // ต้อง copy วิธี return 401 ทุกที่
}
```

### ปัญหา 4 — ไม่ handle concurrent requests

`HttpListener.GetContextAsync()` ในโค้ดของเรา **รอทีละ request** ถ้า request แรกใช้เวลา 5 วินาที request ที่สองต้องรอ นั่นคือ throughput = 1 req/5s

```csharp
// โค้ดของเรา:
while (true)
{
    var context = await listener.GetContextAsync();  // รอ
    // handle...  ← request ที่ 2 ต้องรอตรงนี้
}

// แก้ได้ด้วย:
while (true)
{
    var context = await listener.GetContextAsync();
    _ = Task.Run(() => HandleRequest(context));  // ไม่ await → concurrent
}
```

### ปัญหา 5 — ไม่มี Dependency Injection

```csharp
// ต้อง new() เองทุกที่
var userRepo    = new UserRepository(new SqlConnection(connectionString));
var emailSender = new EmailSender(new SmtpClient(smtpConfig));
var userService = new UserService(userRepo, emailSender);
// dependency tree ซับซ้อนขึ้นเรื่อยๆ
```

---

## บทสรุป

สิ่งที่เราเรียนรู้ในบทนี้:

- **Web Server** รับ HTTP connection ระดับ network, เสิร์ฟ static content หรือ forward ต่อ
- **Web Application** รัน business logic สร้าง dynamic response ตาม request
- **HTTP Request** มี 3 ส่วน: Request Line, Headers, Body
- **HTTP Response** มี 3 ส่วน: Status Line, Headers, Body
- **PORT** คือที่อยู่ของโปรแกรมบนเครื่อง OS ใช้ forward connection ให้ถูก process
- **HttpListener** ทำให้เราสร้าง HTTP server โดยไม่ใช้ framework แต่ code ยุ่งเหยิงมากเมื่อขนาดใหญ่ขึ้น
- ปัญหาเหล่านี้คือสิ่งที่ **ASP.NET Core แก้ให้เรา** ใน Parts ต่อๆ ไป

---

## Workshop

> ทำคนเดียวหลังจากอ่านจบ

1. **Lab 1.1** — รันโค้ด `ManualRoutingServer` แล้วทดสอบทุก endpoint ด้วย curl หรือ Postman  
   ลองส่ง POST ที่ body JSON ผิด format แล้วดูว่าเกิดอะไรขึ้น

2. **Lab 1.2** — แก้ไข `ManualRoutingServer` ให้รองรับ DELETE `/users/{id}` แล้ว return 404 ถ้าหาไม่เจอ

3. **Lab 1.3 (Bonus)** — แก้ไข server ให้ handle concurrent requests ด้วย `Task.Run` แล้วทดสอบด้วย:
   ```bash
   # ยิง 10 requests พร้อมกัน
   for i in {1..10}; do curl http://localhost:5000/users & done
   ```

---

*→ บทถัดไป: **Part 2 — Hosting Model & Program.cs** — เราจะดูว่า ASP.NET Core แก้ปัญหาทั้งหมดนี้ผ่าน Generic Host อย่างไร*
