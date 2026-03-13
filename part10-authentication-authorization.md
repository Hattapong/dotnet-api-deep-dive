# Part 10 — Authentication & Authorization

> **Deep Dive Series: .NET Web API Internals**  
> Authentication = "คุณคือใคร" — Authorization = "คุณทำอะไรได้บ้าง"  
> บทนี้จะดู JWT Bearer, Claims, และ Policy-based Authorization จากรากฐาน

---

## 1. ความแตกต่างที่ต้องเข้าใจก่อน

```
Authentication (AuthN)          Authorization (AuthZ)
──────────────────────────────  ──────────────────────────────
"คุณคือใคร?"                    "คุณทำอะไรได้บ้าง?"
ตรวจสอบ identity                ตรวจสอบ permission
เกิดก่อน                        เกิดหลัง AuthN เสมอ
JWT, Cookie, API Key, ...       Role, Policy, Claim, ...

UseAuthentication()             UseAuthorization()
↑ set HttpContext.User          ↑ ตรวจ HttpContext.User กับ endpoint policy
```

### ใน Middleware Pipeline

```
Request
   │
   ▼
UseAuthentication()   → อ่าน JWT → verify → set HttpContext.User
   │
   ▼
UseAuthorization()    → ตรวจ [Authorize] policy กับ HttpContext.User
   │
   ├── ไม่ผ่าน → 401 Unauthorized (ไม่มี identity)
   │           → 403 Forbidden (มี identity แต่ไม่มีสิทธิ์)
   ▼ ผ่าน
[ Controller Action ]
```

---

## 2. JWT คืออะไร?

**JSON Web Token (JWT)** คือ token ที่มี 3 ส่วนคั่นด้วย `.`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiI0MiIsIm5hbWUiOiJBbGljZSIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcwMDAwMDAwMH0
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
 ──────────────────────────────────────────────
         Header           Payload          Signature
```

```
Header  (base64url):  { "alg": "HS256", "typ": "JWT" }
Payload (base64url):  { "sub": "42", "name": "Alice",
                        "role": "admin", "exp": 1700000000 }
Signature:            HMACSHA256(header + "." + payload, secret)
```

JWT **ไม่ได้เข้ารหัส** — ใครก็ decode payload ได้ แต่ **แก้ไขไม่ได้** เพราะ signature จะผิดทันที

```
Client                              Server
  │                                    │
  │  POST /auth/login                  │
  │  { username, password }  ─────────►│  verify credentials
  │                                    │  สร้าง JWT
  │◄──────────────── JWT token ────────│
  │                                    │
  │  GET /api/users                    │
  │  Authorization: Bearer <JWT> ─────►│  verify signature
  │                                    │  decode claims
  │◄──────────────── 200 + data ───────│
```

---

## 3. Setup JWT Authentication

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

```json
// appsettings.json
{
  "Jwt": {
    "Secret":         "your-256-bit-secret-key-here-minimum-32-chars!!",
    "Issuer":         "my-api",
    "Audience":       "my-clients",
    "ExpiryMinutes":  60
  }
}
```

```csharp
// JwtOptions.cs
public class JwtOptions
{
    [Required] public string Secret        { get; set; } = "";
    [Required] public string Issuer        { get; set; } = "";
    [Required] public string Audience      { get; set; } = "";
    public int ExpiryMinutes { get; set; } = 60;
}
```

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// ── Register JwtOptions ──────────────────────────────────────
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration("Jwt")
    .ValidateDataAnnotations()
    .ValidateOnStart();

// ── Setup Authentication ─────────────────────────────────────
var jwtOptions = builder.Configuration
    .GetSection("Jwt")
    .Get<JwtOptions>()!;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,       // ตรวจ expiry
            ValidateIssuerSigningKey = true,

            ValidIssuer      = jwtOptions.Issuer,
            ValidAudience    = jwtOptions.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtOptions.Secret)),

            ClockSkew = TimeSpan.Zero  // default คือ 5 นาที tolerance
                                       // ตั้งเป็น 0 เพื่อ expiry ตรงเป๊ะ
        };

        // event hooks สำหรับ debug หรือ customize
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                // log เมื่อ token ไม่ valid
                var logger = ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();
                logger.LogWarning("JWT auth failed: {Error}", ctx.Exception.Message);
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddControllers();

var app = builder.Build();

app.UseAuthentication();   // ← ต้องก่อน UseAuthorization เสมอ
app.UseAuthorization();
app.MapControllers();

await app.RunAsync();
```

---

## 4. TokenService — สร้างและ Verify JWT

```csharp
// ITokenService.cs
public interface ITokenService
{
    string GenerateToken(User user);
}

// TokenService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.IdentityModel.Tokens;

public class TokenService : ITokenService
{
    private readonly JwtOptions _jwt;

    public TokenService(IOptions<JwtOptions> options)
        => _jwt = options.Value;

    public string GenerateToken(User user)
    {
        // Claims = ข้อมูลที่ฝังใน token
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub,   user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Name,  user.Name),
            new(JwtRegisteredClaimNames.Jti,   Guid.NewGuid().ToString()),
            // roles — เพิ่มได้หลาย claim ชื่อเดียวกัน
            new(ClaimTypes.Role, user.Role),
        };

        // เพิ่ม custom claims ตามต้องการ
        if (user.TenantId is not null)
            claims.Add(new("tenant_id", user.TenantId));

        var key   = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwt.Secret));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer:             _jwt.Issuer,
            audience:           _jwt.Audience,
            claims:             claims,
            notBefore:          DateTime.UtcNow,
            expires:            DateTime.UtcNow.AddMinutes(_jwt.ExpiryMinutes),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## 5. AuthController — Login Endpoint

```csharp
// LoginRequest / LoginResponse
public record LoginRequest(
    [Required][EmailAddress] string Email,
    [Required] string Password);

public record LoginResponse(string Token, DateTime ExpiresAt, string Name);

// AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IUserRepository _repo;
    private readonly ITokenService   _tokenService;
    private readonly IOptions<JwtOptions> _jwt;

    public AuthController(
        IUserRepository     repo,
        ITokenService       tokenService,
        IOptions<JwtOptions> jwt)
    {
        _repo         = repo;
        _tokenService = tokenService;
        _jwt          = jwt;
    }

    [HttpPost("login")]
    [AllowAnonymous]   // endpoint นี้ไม่ต้อง authenticate ก่อน
    public async Task<ActionResult<LoginResponse>> Login(
        LoginRequest req, CancellationToken ct)
    {
        // หา user จาก email
        var user = await _repo.FindByEmailAsync(req.Email, ct);
        if (user is null)
            return Unauthorized(new { error = "Invalid credentials" });

        // verify password (ใช้ BCrypt หรือ PasswordHasher จริงๆ)
        if (!PasswordHasher.Verify(req.Password, user.PasswordHash))
            return Unauthorized(new { error = "Invalid credentials" });

        // สร้าง token
        var token     = _tokenService.GenerateToken(user);
        var expiresAt = DateTime.UtcNow.AddMinutes(_jwt.Value.ExpiryMinutes);

        return Ok(new LoginResponse(token, expiresAt, user.Name));
    }
}
```

---

## 6. Claims — ข้อมูล Identity ที่ฝังใน Token

Claims คือ key-value pairs ที่บรรจุข้อมูล identity ไว้ใน JWT เมื่อ token ผ่านการ verify แล้ว ASP.NET Core จะ populate `HttpContext.User` ด้วย claims เหล่านี้

```csharp
// อ่าน claims จาก HttpContext.User ใน Controller
[HttpGet("me")]
[Authorize]
public IActionResult GetMe()
{
    var user = HttpContext.User;

    // อ่าน claim ด้วย type
    var userId   = user.FindFirstValue(JwtRegisteredClaimNames.Sub);
    var email    = user.FindFirstValue(JwtRegisteredClaimNames.Email);
    var name     = user.FindFirstValue(JwtRegisteredClaimNames.Name);
    var role     = user.FindFirstValue(ClaimTypes.Role);
    var tenantId = user.FindFirstValue("tenant_id");

    // helper properties
    bool isAuthenticated = user.Identity?.IsAuthenticated ?? false;
    string? username     = user.Identity?.Name;

    return Ok(new { userId, email, name, role, tenantId });
}
```

### ClaimsPrincipal Extensions — อ่าน Claims สะดวกขึ้น

```csharp
// ClaimsPrincipalExtensions.cs
public static class ClaimsPrincipalExtensions
{
    public static int GetUserId(this ClaimsPrincipal user)
    {
        var value = user.FindFirstValue(JwtRegisteredClaimNames.Sub)
            ?? throw new UnauthorizedException("User ID claim missing");
        return int.Parse(value);
    }

    public static string GetEmail(this ClaimsPrincipal user)
        => user.FindFirstValue(JwtRegisteredClaimNames.Email)
            ?? throw new UnauthorizedException("Email claim missing");

    public static string? GetTenantId(this ClaimsPrincipal user)
        => user.FindFirstValue("tenant_id");
}

// ใช้งานใน Controller
[HttpGet("orders")]
[Authorize]
public async Task<IActionResult> GetMyOrders(CancellationToken ct)
{
    var userId = User.GetUserId();  // ← extension method สะอาดกว่า
    var orders = await _orderService.GetByUserIdAsync(userId, ct);
    return Ok(orders);
}
```

---

## 7. [Authorize] Attribute — ระดับต่างๆ

```csharp
// ── ระดับ Controller — ทุก action ใน controller ต้อง authenticate ──
[ApiController]
[Route("api/[controller]")]
[Authorize]                    // ← ทุก action ต้องมี JWT
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok();  // ต้อง authenticate

    [HttpGet("{id}")]
    public IActionResult GetById(int id) => Ok();  // ต้อง authenticate

    [AllowAnonymous]             // ← override ตรงนี้ — ไม่ต้อง authenticate
    [HttpGet("public")]
    public IActionResult GetPublic() => Ok("anyone can see this");
}

// ── Role-based ──────────────────────────────────────────────
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult Delete(int id) => NoContent();

[Authorize(Roles = "Admin,Manager")]  // ← OR — Admin หรือ Manager
[HttpPost]
public IActionResult Create(CreateProductDto dto) => Ok();
```

---

## 8. Policy-based Authorization — ยืดหยุ่นกว่า Role

Role-based มีข้อจำกัด — ถ้า logic ซับซ้อนขึ้น เช่น "Admin ที่มาจาก tenant เดียวกัน" Role เดียวไม่พอ Policy แก้ปัญหานี้

### 8.1 Simple Claim Policy

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // user ต้องมี claim "department" = "engineering"
    options.AddPolicy("EngineeringOnly", policy =>
        policy.RequireClaim("department", "engineering"));

    // user ต้องมี claim "subscription" ค่าใดค่าหนึ่ง
    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));

    // user ต้องมี role Admin
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    // user ต้องมี claim email ที่ลงท้ายด้วย @company.com
    options.AddPolicy("CompanyEmail", policy =>
        policy.RequireAssertion(ctx =>
            ctx.User.GetEmail().EndsWith("@company.com")));
});
```

```csharp
// ใช้งาน
[Authorize(Policy = "EngineeringOnly")]
[HttpPost("deploy")]
public IActionResult Deploy() => Ok("Deploying...");

[Authorize(Policy = "PremiumUser")]
[HttpGet("analytics")]
public IActionResult GetAnalytics() => Ok();
```

### 8.2 Custom Requirement + Handler

สำหรับ logic ที่ซับซ้อนและต้องการ dependency:

```csharp
// 1. Requirement — กำหนด "requirement" ว่าต้องการอะไร
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

// 2. Handler — logic ตรวจสอบ requirement
public class MinimumAgeHandler
    : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement       requirement)
    {
        var birthDateClaim = context.User.FindFirstValue("birthdate");

        if (birthDateClaim is null ||
            !DateTime.TryParse(birthDateClaim, out var birthDate))
        {
            context.Fail();  // ← ไม่มีข้อมูล → fail
            return Task.CompletedTask;
        }

        var age = DateTime.Today.Year - birthDate.Year;
        if (birthDate.Date > DateTime.Today.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);  // ← ผ่าน
        else
            context.Fail();

        return Task.CompletedTask;
    }
}

// 3. Register
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Adult", policy =>
        policy.AddRequirements(new MinimumAgeRequirement(18)));

    options.AddPolicy("Senior", policy =>
        policy.AddRequirements(new MinimumAgeRequirement(60)));
});
```

### 8.3 Resource-based Authorization

เมื่อต้องการ authorize ต่อ resource เช่น "แก้ไข post ได้เฉพาะ owner":

```csharp
// Requirement
public class ResourceOwnerRequirement : IAuthorizationRequirement { }

// Handler — รับ resource เป็น parameter
public class ResourceOwnerHandler
    : AuthorizationHandler<ResourceOwnerRequirement, Post>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement    requirement,
        Post                        post)          // ← resource
    {
        var userId = context.User.GetUserId();

        if (post.AuthorId == userId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// ใช้งาน — inject IAuthorizationService ใน Controller
[HttpPut("{id}")]
[Authorize]
public async Task<IActionResult> Update(
    int           id,
    UpdatePostDto dto,
    [FromServices] IAuthorizationService authService,
    CancellationToken ct)
{
    var post = await _postService.GetByIdAsync(id, ct)
        ?? throw new NotFoundException($"Post {id} not found");

    // ตรวจสอบสิทธิ์ต่อ resource นี้โดยเฉพาะ
    var result = await authService.AuthorizeAsync(
        User, post, new ResourceOwnerRequirement());

    if (!result.Succeeded)
        return Forbid();

    var updated = await _postService.UpdateAsync(id, dto, ct);
    return Ok(updated);
}
```

---

## 9. ตัวอย่างสมบูรณ์

```csharp
// ── Models ───────────────────────────────────────────────────
public record User(int Id, string Name, string Email,
    string PasswordHash, string Role);
public record LoginRequest(
    [Required][EmailAddress] string Email,
    [Required] string Password);
public record LoginResponse(string Token, string Name, string Role);
public record SecretData(string Message, int RequestedByUserId);

// ── Simple In-Memory User Store ──────────────────────────────
public class UserStore
{
    // BCrypt hash ของ "password123"
    private const string Hash =
        "$2a$11$dummy.hash.for.demo.purposesXXXXXXXXXXXXXXXXXXXXX";

    public static readonly List<User> Users = new()
    {
        new(1, "Alice Admin", "alice@example.com", Hash, "Admin"),
        new(2, "Bob User",    "bob@example.com",   Hash, "User"),
    };

    public User? FindByEmail(string email) =>
        Users.FirstOrDefault(u =>
            u.Email.Equals(email, StringComparison.OrdinalIgnoreCase));
}

// ── Token Service ────────────────────────────────────────────
public class TokenService
{
    private readonly JwtOptions _jwt;

    public TokenService(IOptions<JwtOptions> opts) => _jwt = opts.Value;

    public string Generate(User user)
    {
        var key    = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_jwt.Secret));
        var creds  = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token  = new JwtSecurityToken(
            issuer:   _jwt.Issuer,
            audience: _jwt.Audience,
            claims: new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub,   user.Id.ToString()),
                new Claim(JwtRegisteredClaimNames.Email, user.Email),
                new Claim(JwtRegisteredClaimNames.Name,  user.Name),
                new Claim(ClaimTypes.Role,               user.Role),
            },
            expires:            DateTime.UtcNow.AddMinutes(_jwt.ExpiryMinutes),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}

// ── Auth Controller ──────────────────────────────────────────
[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly UserStore   _store;
    private readonly TokenService _tokens;

    public AuthController(UserStore store, TokenService tokens)
    { _store = store; _tokens = tokens; }

    [HttpPost("login")]
    [AllowAnonymous]
    public ActionResult<LoginResponse> Login(LoginRequest req)
    {
        var user = _store.FindByEmail(req.Email);

        // ในระบบจริงใช้ BCrypt.Verify(req.Password, user.PasswordHash)
        if (user is null || req.Password != "password123")
            return Unauthorized(new { error = "Invalid credentials" });

        var token = _tokens.Generate(user);
        return Ok(new LoginResponse(token, user.Name, user.Role));
    }
}

// ── Protected Controller ─────────────────────────────────────
[ApiController]
[Route("api/[controller]")]
[Authorize]                              // ทุก action ต้อง authenticate
public class SecretController : ControllerBase
{
    // GET api/secret — ต้อง login เท่านั้น
    [HttpGet]
    public IActionResult Get()
    {
        var userId = int.Parse(
            User.FindFirstValue(JwtRegisteredClaimNames.Sub)!);
        return Ok(new SecretData("This is secret!", userId));
    }

    // GET api/secret/admin — ต้องเป็น Admin
    [HttpGet("admin")]
    [Authorize(Roles = "Admin")]
    public IActionResult AdminOnly() =>
        Ok(new { message = "Admin area", time = DateTime.UtcNow });

    // GET api/secret/me — ดูข้อมูล claims ของตัวเอง
    [HttpGet("me")]
    public IActionResult WhoAmI() => Ok(new
    {
        id    = User.FindFirstValue(JwtRegisteredClaimNames.Sub),
        name  = User.FindFirstValue(JwtRegisteredClaimNames.Name),
        email = User.FindFirstValue(JwtRegisteredClaimNames.Email),
        role  = User.FindFirstValue(ClaimTypes.Role),
    });
}

// ── Program.cs ───────────────────────────────────────────────
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration("Jwt")
    .ValidateDataAnnotations()
    .ValidateOnStart();

var jwtOpts = builder.Configuration.GetSection("Jwt").Get<JwtOptions>()!;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.TokenValidationParameters = new()
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer      = jwtOpts.Issuer,
            ValidAudience    = jwtOpts.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtOpts.Secret)),
            ClockSkew = TimeSpan.Zero
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", p => p.RequireRole("Admin"));
});

builder.Services.AddControllers();
builder.Services.AddSingleton<UserStore>();
builder.Services.AddScoped<TokenService>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

await app.RunAsync();
```

ทดสอบ:
```bash
# login เป็น Alice (Admin)
TOKEN=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}' \
  | jq -r '.token')

# ดู secret — ต้องมี token
curl http://localhost:5000/api/secret \
  -H "Authorization: Bearer $TOKEN"

# ดู admin area — Alice ทำได้
curl http://localhost:5000/api/secret/admin \
  -H "Authorization: Bearer $TOKEN"

# login เป็น Bob (User)
TOKEN_BOB=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"bob@example.com","password":"password123"}' \
  | jq -r '.token')

# ดู admin area — Bob ทำไม่ได้ → 403
curl http://localhost:5000/api/secret/admin \
  -H "Authorization: Bearer $TOKEN_BOB"

# ไม่มี token → 401
curl http://localhost:5000/api/secret
```

---

## บทสรุป

```
JWT Flow:
  Client login → server สร้าง JWT → client เก็บ token
  Client ส่ง "Authorization: Bearer <token>" ทุก request
  Server verify signature → decode claims → set HttpContext.User

Claims:
  Key-value pairs ที่ฝังใน JWT
  อ่านได้ผ่าน HttpContext.User.FindFirstValue("claim_type")
  ระวัง: อย่าเก็บ sensitive data เช่น password ใน claims

Authorization 3 ระดับ:
  Role-based     [Authorize(Roles = "Admin")]
  Policy-based   [Authorize(Policy = "PremiumUser")]
  Resource-based IAuthorizationService.AuthorizeAsync(User, resource, requirement)

กฎสำคัญ:
  ✅ UseAuthentication() ก่อน UseAuthorization() เสมอ
  ✅ JWT Secret ต้องยาวอย่างน้อย 32 chars
  ✅ ตั้ง ClockSkew = TimeSpan.Zero สำหรับ expiry เป๊ะ
  ✅ ใช้ User Secrets เก็บ JWT Secret ใน dev
  ❌ ห้ามเก็บ password หรือ sensitive data ใน JWT payload
  ❌ ห้าม hardcode JWT secret ใน appsettings.json ที่ commit
```

---

## Workshop

1. **Lab 10.1** — เพิ่ม `POST /api/auth/refresh` endpoint ที่รับ token เก่า ตรวจ claims แล้วออก token ใหม่ (ต้องยัง valid อยู่)

2. **Lab 10.2** — สร้าง Policy `"SeniorStaff"` ที่กำหนดว่าต้องมี Role เป็น Admin หรือ Manager **และ** มี claim `"department"` = `"engineering"` พร้อมทดสอบทั้ง 2 กรณี

3. **Lab 10.3** — Implement Resource-based Authorization สำหรับ `PUT /api/posts/{id}` โดยอนุญาตเฉพาะ Admin หรือ post owner เท่านั้น

4. **Lab 10.4 (Bonus)** — เพิ่ม `OnTokenValidated` event ใน `JwtBearerEvents` เพื่อ blacklist tokens ที่ถูก revoke โดยเก็บ revoked JTI ไว้ใน `IMemoryCache`

---

*→ บทถัดไป: **Part 12 — Testing** — Unit test Service layer, Integration test ผ่าน WebApplicationFactory และ test endpoints ที่ต้องการ Auth*
