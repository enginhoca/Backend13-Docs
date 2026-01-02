# Rate Limiting
## ECommerce API - Güvenlik Dersleri

**Seviye:** Orta  
**Hedef:** API endpoint'lerine rate limiting uygulama

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Rate Limiting Nedir?](#rate-limiting-nedir)
3. [Neden Rate Limiting?](#neden-rate-limiting)
4. [.NET 10.0 Built-in Rate Limiting](#net-100-built-in-rate-limiting)
5. [ECommerce API için Rate Limiting](#ecommerce-api-için-rate-limiting)
6. [RateLimitingConfig Sınıfı](#ratelimitingconfig-sınıfı)
7. [Program.cs Yapılandırması](#programcs-yapılandırması)
8. [Controller'da Kullanım](#controllerda-kullanım)
9. [Test Senaryoları](#test-senaryoları)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Rate Limiting Nedir?

**Rate Limiting (İstek Hızı Sınırlama)**, bir API endpoint'ine belirli bir süre içinde yapılabilecek istek sayısını sınırlayan bir güvenlik mekanizmasıdır.

**Örnek:**
- Login endpoint'i: 1 dakikada en fazla 5 istek
- Property creation: 1 saatte en fazla 10 istek
- Genel API: 1 dakikada en fazla 100 istek

### ECommerce API'de Neden Rate Limiting?

1. **Brute Force Saldırılarına Karşı:**
   - Login endpoint'ine sınırsız istek atılmasını engeller
   - Şifre tahmin saldırılarını önler

2. **DDoS (Distributed Denial of Service) Koruması:**
   - API'nizi aşırı istekle çökertmeyi engeller
   - Sistem kaynaklarını korur

3. **Kötüye Kullanımı Önleme:**
   - Spam ilan oluşturmayı engeller
   - Aşırı inquiry gönderimini önler

4. **Adil Kullanım:**
   - Tüm kullanıcılara eşit erişim sağlar
   - Bir kullanıcının tüm kaynakları tüketmesini engeller

---

## 📚 Rate Limiting Nedir? (Detaylı)

### Rate Limiting Stratejileri

**1. Fixed Window (Sabit Pencere):**
- Belirli bir zaman penceresi içinde istek sayısı sınırı
- Örnek: 1 dakikada 5 istek
- Pencere sıfırlandığında limit yenilenir

**2. Sliding Window (Kayan Pencere):**
- Son N saniyede istek sayısı sınırı
- Daha adil dağıtım
- Daha karmaşık hesaplama

**3. Token Bucket:**
- Belirli bir token sayısı
- Her istek bir token tüketir
- Zamanla token'lar yenilenir

**.NET 10.0'da:** Fixed Window Rate Limiting kullanılır (basit ve etkili).

### Rate Limiting Nasıl Çalışır?

```
1. İstek Gelir
   ↓
2. Rate Limiter Kontrol Eder
   - IP adresine göre istek sayısını kontrol et
   - Limit aşıldı mı?
   ↓
3a. Limit Aşılmadı → İsteği İşle
3b. Limit Aşıldı → 429 Too Many Requests Hatası Döner
```

**Response Header:**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json
{
  "success": false,
  "message": "Çok fazla istek yapıldı. Lütfen 60 saniye sonra tekrar deneyin."
}
```

---

## 🛠️ .NET 10.0 Built-in Rate Limiting

**.NET 7+** ile birlikte rate limiting **built-in** olarak gelir. Ek paket gerekmez!

**Namespace:**
- `Microsoft.AspNetCore.RateLimiting`
- `System.Threading.RateLimiting`

**Özellikler:**
- ✅ Fixed Window Rate Limiting
- ✅ IP-based limiting
- ✅ Custom partition key (kullanıcı ID, endpoint, vb.)
- ✅ Policy-based configuration
- ✅ Endpoint-specific policies

---

## ⚙️ ECommerce API için Rate Limiting

### Rate Limiting Politikaları

ECommerce API için şu endpoint'lere rate limiting uygulayacağız:

1. **Login Endpoint:** 1 dakikada 5 istek (brute force koruması)
2. **Register Endpoint:** 1 saatte 3 istek (spam kayıt önleme)
3. **Property Create:** 1 saatte 10 istek (spam ilan önleme)
4. **Inquiry Create:** 1 saatte 20 istek (spam inquiry önleme)
5. **Global:** 1 dakikada 100 istek (genel API koruması)

---

## 📝 RateLimitingConfig Sınıfı

**ECommerce.Business/Configs/RateLimitingConfig.cs:**

```csharp
namespace ECommerce.Business.Configs;

public class RateLimitingConfig
{
    public RateLimitPolicy Global { get; set; } = new();
    public RateLimitPolicy Login { get; set; } = new();
    public RateLimitPolicy Register { get; set; } = new();
    public RateLimitPolicy PropertyCreate { get; set; } = new();
    public RateLimitPolicy InquiryCreate { get; set; } = new();
}

public class RateLimitPolicy
{
    public int PermitLimit { get; set; } = 10;  // İzin verilen istek sayısı
    public int WindowMinutes { get; set; } = 1;  // Zaman penceresi (dakika)
}
```

**Açıklamalar:**
- **PermitLimit:** Belirli bir zaman penceresi içinde izin verilen maksimum istek sayısı
- **WindowMinutes:** Zaman penceresi süresi (dakika cinsinden)

### appsettings.json Yapılandırması

**ECommerce.API/appsettings.json:**

```json
{
  "RateLimiting": {
    "Global": {
      "PermitLimit": 100,
      "WindowMinutes": 1
    },
    "Login": {
      "PermitLimit": 5,
      "WindowMinutes": 1
    },
    "Register": {
      "PermitLimit": 3,
      "WindowMinutes": 60
    },
    "PropertyCreate": {
      "PermitLimit": 10,
      "WindowMinutes": 60
    },
    "InquiryCreate": {
      "PermitLimit": 20,
      "WindowMinutes": 60
    }
  }
}
```

**Açıklamalar:**

**Global Policy:**
- 1 dakikada 100 istek
- Tüm endpoint'ler için genel limit

**Login Policy:**
- 1 dakikada 5 istek
- Brute force saldırılarına karşı koruma
- Kısa süreli limit (kullanıcı deneme-yanılma yapabilir)

**Register Policy:**
- 1 saatte 3 istek
- Spam kayıt önleme
- Uzun süreli limit (kayıt sık yapılmaz)

**PropertyCreate Policy:**
- 1 saatte 10 istek
- Spam ilan önleme
- Makul bir limit (normal kullanıcı yeterli)

**InquiryCreate Policy:**
- 1 saatte 20 istek
- Spam inquiry önleme
- Daha esnek limit (müşteri birden fazla ilana soru sorabilir)

---

## ⚙️ Program.cs Yapılandırması

### Rate Limiting Servis Kaydı

```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;
using ECommerce.Business.Configs;

var builder = WebApplication.CreateBuilder(args);

// Rate Limiting Configuration'dan oku
var rateLimitConfig = builder.Configuration.GetSection("RateLimiting").Get<RateLimitingConfig>();

if (rateLimitConfig == null)
{
    throw new InvalidOperationException("RateLimiting configuration bulunamadı!");
}

// Rate Limiting servislerini kaydet
builder.Services.AddRateLimiter(options =>
{
    // Global Policy
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = rateLimitConfig.Global.PermitLimit,
                Window = TimeSpan.FromMinutes(rateLimitConfig.Global.WindowMinutes)
            }));

    // Login Policy
    options.AddPolicy("LoginPolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = rateLimitConfig.Login.PermitLimit,
                Window = TimeSpan.FromMinutes(rateLimitConfig.Login.WindowMinutes)
            }));

    // Register Policy
    options.AddPolicy("RegisterPolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = rateLimitConfig.Register.PermitLimit,
                Window = TimeSpan.FromMinutes(rateLimitConfig.Register.WindowMinutes)
            }));

    // PropertyCreate Policy
    options.AddPolicy("PropertyCreatePolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = rateLimitConfig.PropertyCreate.PermitLimit,
                Window = TimeSpan.FromMinutes(rateLimitConfig.PropertyCreate.WindowMinutes)
            }));

    // InquiryCreate Policy
    options.AddPolicy("InquiryCreatePolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = rateLimitConfig.InquiryCreate.PermitLimit,
                Window = TimeSpan.FromMinutes(rateLimitConfig.InquiryCreate.WindowMinutes)
            }));

    // Rate limit aşıldığında özel response
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.ContentType = "application/json";
        
        var retryAfter = context.RetryAfter?.TotalSeconds ?? 60;
        var response = ResponseDto<object>.Fail(
            $"Çok fazla istek yapıldı. Lütfen {retryAfter} saniye sonra tekrar deneyin.",
            StatusCodes.Status429TooManyRequests
        );

        var jsonResponse = JsonSerializer.Serialize(response, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });

        await context.HttpContext.Response.WriteAsync(jsonResponse, cancellationToken);
    };
});

var app = builder.Build();

// Rate Limiting Middleware
app.UseRateLimiter();

// Diğer middleware'ler...
```

### Detaylı Açıklamalar:

**1. AddRateLimiter():**
```csharp
builder.Services.AddRateLimiter(options => { ... });
```
- **Amaç:** Rate limiting servislerini DI container'a ekler
- **Options:** Rate limiting policy'lerini yapılandırır

**2. GlobalLimiter:**
```csharp
options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(...);
```
- **Amaç:** Tüm endpoint'ler için genel bir limit belirler
- **Partition Key:** IP adresi (her IP için ayrı limit)
- **Fixed Window:** Sabit zaman penceresi kullanır

**3. PartitionKey (IP Adresi):**
```csharp
partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown"
```
- **Connection.RemoteIpAddress:** İsteği gönderen kullanıcının IP adresi
- **ToString():** IP adresini string'e çevirir
- **?? "unknown":** IP adresi null ise "unknown" kullan (fallback)
- **Sonuç:** Her IP adresi için ayrı rate limit (partition)

**4. FixedWindowRateLimiterOptions:**
```csharp
new FixedWindowRateLimiterOptions
{
    AutoReplenishment = true,  // Otomatik yenileme
    PermitLimit = 5,            // İzin verilen istek sayısı
    Window = TimeSpan.FromMinutes(1)  // Zaman penceresi (1 dakika)
}
```
- **AutoReplenishment:** Zaman penceresi bittiğinde limit otomatik yenilenir
- **PermitLimit:** Pencere içinde izin verilen maksimum istek sayısı
- **Window:** Zaman penceresi süresi

**5. AddPolicy():**
```csharp
options.AddPolicy("LoginPolicy", context => { ... });
```
- **Amaç:** Endpoint-specific rate limiting policy'si oluşturur
- **Policy Adı:** "LoginPolicy" - controller'da kullanılacak
- **Factory:** Policy'nin nasıl oluşturulacağını belirtir

**6. OnRejected Callback:**
```csharp
options.OnRejected = async (context, cancellationToken) => { ... };
```
- **Amaç:** Rate limit aşıldığında özel response döner
- **Status Code:** 429 Too Many Requests
- **Retry-After:** Kaç saniye sonra tekrar deneneceği
- **Response:** ResponseDto formatında hata mesajı

**7. UseRateLimiter():**
```csharp
app.UseRateLimiter();
```
- **Amaç:** Rate limiting middleware'ini pipeline'a ekler
- **Sıra:** CORS'tan sonra, Authentication'dan önce

### Middleware Sırası

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseHttpsRedirection();
app.UseRateLimiter();  // Rate limiting (CORS'tan sonra, Auth'dan önce)
app.UseCors("AllowedSpecificOrigins");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

**Neden Bu Sıra?**
- Rate limiting, isteği erken aşamada kontrol eder
- Limit aşıldıysa, authentication'a gerek kalmaz
- Performans: Gereksiz işlemlerden kaçınır

---

## 🎮 Controller'da Kullanım

### Login Endpoint'ine Rate Limiting

**ECommerce.API/Controllers/AuthController.cs:**

```csharp
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class AuthController : CustomControllerBase
{
    [HttpPost("login")]
    [EnableRateLimiting("LoginPolicy")]  // Login policy'sini aktif et
    public async Task<IActionResult> Login([FromBody] LoginDto loginDto)
    {
        var result = await _authService.LoginAsync(loginDto);
        return CreateActionResult(result);
    }
}
```

**Açıklama:**
- **EnableRateLimiting("LoginPolicy"):** Bu endpoint'e "LoginPolicy" policy'sini uygular
- **Policy Adı:** Program.cs'de tanımlanan policy adı ile eşleşmeli

### Register Endpoint'ine Rate Limiting

```csharp
[HttpPost("register")]
[EnableRateLimiting("RegisterPolicy")]  // Register policy'sini aktif et
public async Task<IActionResult> Register([FromBody] RegisterDto registerDto)
{
    var result = await _authService.RegisterAsync(registerDto);
    return CreateActionResult(result);
}
```

### Property Create Endpoint'ine Rate Limiting

**ECommerce.API/Controllers/PropertiesController.cs:**

```csharp
[HttpPost]
[Authorize(Roles = "Agent,Admin")]  // Sadece Agent ve Admin
[EnableRateLimiting("PropertyCreatePolicy")]  // Property create policy
public async Task<IActionResult> CreateProperty([FromBody] PropertyCreateDto dto)
{
    var result = await _propertyService.CreateAsync(dto);
    return CreateActionResult(result);
}
```

### Inquiry Create Endpoint'ine Rate Limiting

**ECommerce.API/Controllers/InquiriesController.cs:**

```csharp
[HttpPost]
[EnableRateLimiting("InquiryCreatePolicy")]  // Inquiry create policy
public async Task<IActionResult> CreateInquiry([FromBody] InquiryCreateDto dto)
{
    var result = await _inquiryService.CreateAsync(dto);
    return CreateActionResult(result);
}
```

**Not:** Authentication olmayan endpoint'lerde de rate limiting kullanılabilir.

---

## 🧪 Test Senaryoları

### Senaryo 1: Normal Kullanım (Limit Aşılmadı)

**Request 1-5 (Login):**
```bash
# 1. İstek
curl -X POST http://localhost:5070/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test123!"}'

# 2-5. İstek (aynı şekilde)
```

**Beklenen Sonuç:**
- ✅ Tüm istekler başarılı (200 OK veya 401 Unauthorized)
- ✅ Rate limit aşılmadı

### Senaryo 2: Rate Limit Aşıldı (6. İstek)

**6. İstek (Login - Limit Aşıldı):**
```bash
curl -X POST http://localhost:5070/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test123!"}' \
  -v
```

**Beklenen Sonuç:**
- ❌ Status Code: 429 Too Many Requests
- ❌ Response:
```json
{
  "success": false,
  "message": "Çok fazla istek yapıldı. Lütfen 60 saniye sonra tekrar deneyin.",
  "data": null
}
```
- ❌ Response Header: `Retry-After: 60`

### Senaryo 3: Farklı IP'ler (Limit Ayrı)

**IP 1 (192.168.1.1):**
- 5 login isteği → ✅ Başarılı
- 6. istek → ❌ 429 (Limit aşıldı)

**IP 2 (192.168.1.2):**
- 5 login isteği → ✅ Başarılı (IP 1'den bağımsız)
- 6. istek → ❌ 429 (Kendi limiti aşıldı)

**Açıklama:** Her IP adresi için ayrı rate limit vardır.

### Senaryo 4: Zaman Penceresi Yenilendi

**1. Dakika:**
- İstek 1-5 → ✅ Başarılı
- İstek 6 → ❌ 429

**2. Dakika (1 dakika geçti):**
- İstek 7 → ✅ Başarılı (Limit yenilendi)

**Açıklama:** Fixed window, pencere bittiğinde limit otomatik yenilenir.

### Senaryo 5: Farklı Endpoint'ler (Farklı Limitler)

**Login Endpoint:**
- 1 dakikada 5 istek limiti
- 6. istek → ❌ 429

**Property Create Endpoint:**
- 1 saatte 10 istek limiti
- İstek 1-10 → ✅ Başarılı
- İstek 11 → ❌ 429

**Açıklama:** Her endpoint'in kendi rate limit policy'si vardır.

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **IP-Based Limiting:**
   - Rate limiting IP adresine göre yapılır
   - Her IP için ayrı limit vardır
   - VPN/Proxy kullanılırsa farklı IP görünebilir

2. **Fixed Window:**
   - Zaman penceresi bittiğinde limit otomatik yenilenir
   - Örnek: 1 dakikalık pencere → 60 saniye sonra limit sıfırlanır

3. **Policy-Based:**
   - Her endpoint için farklı policy kullanılabilir
   - Global policy tüm endpoint'ler için geçerlidir

4. **429 Status Code:**
   - Rate limit aşıldığında 429 Too Many Requests döner
   - Retry-After header ile ne zaman tekrar deneneceği belirtilir

5. **OnRejected Callback:**
   - Özel hata mesajı döndürmek için kullanılır
   - ResponseDto formatında tutarlı response

### Sık Yapılan Hatalar

1. **UseRateLimiter() Unutmak:**
   - ❌ Yanlış: Policy tanımlanır ama middleware eklenmez
   - ✅ Doğru: `app.UseRateLimiter();` eklenmeli

2. **EnableRateLimiting Attribute Unutmak:**
   - ❌ Yanlış: Policy tanımlanır ama endpoint'te kullanılmaz
   - ✅ Doğru: `[EnableRateLimiting("PolicyName")]` eklenmeli

3. **Policy Adı Yanlış:**
   - ❌ Yanlış: `[EnableRateLimiting("Login")]` (policy adı farklı)
   - ✅ Doğru: `[EnableRateLimiting("LoginPolicy")]` (policy adı eşleşmeli)

4. **Middleware Sırası:**
   - ❌ Yanlış: `UseRateLimiter()` authentication'dan sonra
   - ✅ Doğru: `UseRateLimiter()` authentication'dan önce

5. **Configuration Null Check:**
   - ❌ Yanlış: Configuration null olabilir
   - ✅ Doğru: Null check yapılmalı

### İpuçları

1. **Development vs Production:**
   - Development'ta limitler daha esnek olabilir
   - Production'da sıkı limitler kullanın

2. **Limit Değerleri:**
   - Çok sıkı → Kullanıcı deneyimi kötü
   - Çok gevşek → Güvenlik riski
   - Denge önemli!

3. **Testing:**
   - Rate limit'i test etmek için hızlı istek gönderin
   - Farklı IP'lerden test edin
   - Zaman penceresi sonrası test edin

4. **Monitoring:**
   - Rate limit aşım sayılarını loglayın
   - Hangi IP'lerin limit aştığını takip edin
   - Saldırı göstergesi olabilir

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Rate Limiting Nedir:** İstek hızı sınırlama mekanizması
2. ✅ **Neden Gerekli:** Brute force, DDoS, kötüye kullanım koruması
3. ✅ **.NET 10.0 Built-in:** Ek paket gerekmez
4. ✅ **Yapılandırma:** RateLimitingConfig, appsettings.json
5. ✅ **Program.cs:** AddRateLimiter, UseRateLimiter
6. ✅ **Controller:** EnableRateLimiting attribute
7. ✅ **Test:** Farklı senaryolar

**Sonraki Adım:** Secrets Management dersine geçebiliriz.

---

**Başarılar! 🚀**

