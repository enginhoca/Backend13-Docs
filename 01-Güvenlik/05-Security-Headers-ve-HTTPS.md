# Security Headers ve HTTPS
## ECommerce API - Güvenlik

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Security Headers Nedir?](#security-headers-nedir)
3. [HTTPS Nedir?](#https-nedir)
4. [ECommerce API için Security Headers](#ecommerce-api-için-security-headers)
5. [SecurityHeadersConfig Sınıfı](#securityheadersconfig-sınıfı)
6. [SecurityHeadersMiddleware](#securityheadersmiddleware)
7. [Program.cs Yapılandırması](#programcs-yapılandırması)
8. [HTTPS Yapılandırması](#https-yapılandırması)
9. [Test Senaryoları](#test-senaryoları)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Security Headers Nedir?

**Security Headers**, HTTP response'larına eklenen güvenlik header'larıdır. Browser'lara güvenlik politikalarını bildirir.

**Amaç:**
- XSS (Cross-Site Scripting) saldırılarını önlemek
- Clickjacking saldırılarını önlemek
- MIME type sniffing'i önlemek
- HTTPS zorunluluğu

### HTTPS Nedir?

**HTTPS (HyperText Transfer Protocol Secure)**, HTTP'nin şifreli versiyonudur.

**Özellikler:**
- ✅ Veri şifreleme (SSL/TLS)
- ✅ Sunucu kimlik doğrulama
- ✅ Veri bütünlüğü koruması

**Production'da mutlaka kullanılmalı!**

---

## 📚 Security Headers Nedir? (Detaylı)

### Önemli Security Header'ları

**1. X-Content-Type-Options: nosniff**
- Browser'ın MIME type'ı tahmin etmesini engeller
- XSS saldırılarına karşı koruma

**2. X-Frame-Options: DENY / SAMEORIGIN**
- Clickjacking saldırılarını önler
- Sayfanın iframe içinde gösterilmesini engeller

**3. Content-Security-Policy (CSP)**
- Hangi kaynaklardan script, style, image yüklenebileceğini belirtir
- XSS saldırılarına karşı güçlü koruma

**4. Strict-Transport-Security (HSTS)**
- Browser'a HTTPS kullanmasını zorunlu kılar
- HTTP → HTTPS otomatik yönlendirme

**5. Referrer-Policy**
- Referrer bilgisinin nasıl paylaşılacağını belirtir
- Hassas bilgi sızıntısını önler

**6. Permissions-Policy**
- Browser özelliklerinin kullanımını kontrol eder
- Geolocation, camera, microphone vb.

---

## 🔒 HTTPS Nedir?

### HTTP vs HTTPS

**HTTP:**
- ❌ Şifrelenmemiş veri aktarımı
- ❌ Man-in-the-middle saldırılarına açık
- ❌ Veri çalınabilir

**HTTPS:**
- ✅ SSL/TLS ile şifreli veri aktarımı
- ✅ Güvenli bağlantı
- ✅ Veri korunur

### HTTPS Nasıl Çalışır?

1. **Client → Server:** HTTPS isteği
2. **Server → Client:** SSL sertifikası gönderir
3. **Client:** Sertifikayı doğrular
4. **Şifreli Bağlantı:** Veri şifreli aktarılır

---

## 🛠️ ECommerce API için Security Headers

### Security Headers Yapılandırması

**ECommerce.Business/Configs/SecurityHeadersConfig.cs:**

```csharp
namespace ECommerce.Business.Configs;

public class SecurityHeadersConfig
{
    public string XContentTypeOptions { get; set; } = "nosniff";
    public string XFrameOptions { get; set; } = "DENY";
    public string ContentSecurityPolicy { get; set; } = "default-src 'self'";
    public string PermissionsPolicy { get; set; } = "geolocation=(), microphone=(), camera=()";
    public string ReferrerPolicy { get; set; } = "strict-origin-when-cross-origin";
    public int HstsMaxAge { get; set; } = 31536000; // 1 yıl (saniye)
    public bool HstsIncludeSubDomains { get; set; } = true;
    public bool HstsPreload { get; set; } = false;
}
```

**appsettings.json:**

```json
{
  "SecurityHeaders": {
    "XContentTypeOptions": "nosniff",
    "XFrameOptions": "DENY",
    "ContentSecurityPolicy": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'",
    "PermissionsPolicy": "geolocation=(), microphone=(), camera=()",
    "ReferrerPolicy": "strict-origin-when-cross-origin",
    "HstsMaxAge": 31536000,
    "HstsIncludeSubDomains": true,
    "HstsPreload": false
  }
}
```

---

## ⚙️ SecurityHeadersMiddleware

**ECommerce.API/Middleware/SecurityHeadersMiddleware.cs:**

```csharp
using ECommerce.Business.Configs;
using Microsoft.Extensions.Options;

namespace ECommerce.API.Middleware;

public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SecurityHeadersConfig _config;

    public SecurityHeadersMiddleware(RequestDelegate next, IOptions<SecurityHeadersConfig> config)
    {
        _next = next;
        _config = config.Value;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Security Header'ları ekle
        context.Response.Headers.Append("X-Content-Type-Options", _config.XContentTypeOptions);
        context.Response.Headers.Append("X-Frame-Options", _config.XFrameOptions);
        context.Response.Headers.Append("Content-Security-Policy", _config.ContentSecurityPolicy);
        context.Response.Headers.Append("Permissions-Policy", _config.PermissionsPolicy);
        context.Response.Headers.Append("Referrer-Policy", _config.ReferrerPolicy);

        // HSTS (sadece HTTPS'te)
        if (context.Request.IsHttps)
        {
            var hstsValue = $"max-age={_config.HstsMaxAge}";
            if (_config.HstsIncludeSubDomains)
            {
                hstsValue += "; includeSubDomains";
            }
            if (_config.HstsPreload)
            {
                hstsValue += "; preload";
            }
            context.Response.Headers.Append("Strict-Transport-Security", hstsValue);
        }

        await _next(context);
    }
}
```

---

## ⚙️ Program.cs Yapılandırması

```csharp
using ECommerce.Business.Configs;
using ECommerce.API.Middleware;

var builder = WebApplication.CreateBuilder(args);

// Security Headers Configuration
builder.Services.Configure<SecurityHeadersConfig>(
    builder.Configuration.GetSection("SecurityHeaders")
);

var app = builder.Build();

// Middleware Pipeline
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseMiddleware<SecurityHeadersMiddleware>();  // Security Headers
app.UseHttpsRedirection();  // HTTP → HTTPS yönlendirme
app.UseCors("AllowedSpecificOrigins");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## 🔒 HTTPS Yapılandırması

### UseHttpsRedirection()

```csharp
app.UseHttpsRedirection();
```

**Amaç:** HTTP isteklerini HTTPS'e yönlendirir.

**Nasıl Çalışır:**
- HTTP isteği gelir → 307 Temporary Redirect → HTTPS URL'ine yönlendirir

### Production'da HTTPS

**Render.com, Azure, AWS:** Otomatik HTTPS (SSL sertifikası sağlanır)

**Kendi Sunucunuzda:**
- Nginx/Apache reverse proxy
- Let's Encrypt SSL sertifikası
- 443 portunu dinle

---

## 🧪 Test Senaryoları

### Senaryo 1: Security Headers Kontrolü

```bash
curl -I http://localhost:5070/api/properties
```

**Beklenen Response Headers:**
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Permissions-Policy: geolocation=(), microphone=(), camera=()
Referrer-Policy: strict-origin-when-cross-origin
```

### Senaryo 2: HTTPS Yönlendirme

```bash
curl -I http://localhost:5070/api/properties
```

**Beklenen:**
```
HTTP/1.1 307 Temporary Redirect
Location: https://localhost:5071/api/properties
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Security Headers:** Her response'a eklenir
2. **HSTS:** Sadece HTTPS'te çalışır
3. **CSP:** XSS koruması için güçlü
4. **HTTPS:** Production'da zorunlu

### Sık Yapılan Hatalar

1. **Security Headers Unutmak:** ❌ Güvenlik açığı
2. **HTTPS Kullanmamak:** ❌ Veri çalınabilir
3. **CSP Çok Sıkı:** ❌ Uygulama çalışmayabilir

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Security Headers:** Güvenlik header'ları
2. ✅ **HTTPS:** Şifreli bağlantı
3. ✅ **Middleware:** SecurityHeadersMiddleware
4. ✅ **Configuration:** SecurityHeadersConfig
5. ✅ **Test:** Header kontrolü

**Sonraki Adım:** Global Exception Handling dersine geçebiliriz.

---

**Başarılar! 🚀**

