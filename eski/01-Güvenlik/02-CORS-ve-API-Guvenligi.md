# ---
layout: page
title: CORS ve API Güvenliği
order: 2
bolum: 1
---

# CORS ve API Güvenliği
## ECommerce API - Güvenlik

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [CORS Nedir?](#cors-nedir)
3. [Neden CORS Gerekli?](#neden-cors-gerekli)
4. [CORS Nasıl Çalışır?](#cors-nasıl-çalışır)
5. [ECommerce API için CORS Yapılandırması](#ecommerce-api-için-cors-yapılandırması)
6. [CorsConfig Sınıfı](#corsconfig-sınıfı)
7. [Program.cs Yapılandırması](#programcs-yapılandırması)
8. [Middleware Sırası](#middleware-sırası)
9. [Test Senaryoları](#test-senaryoları)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### CORS Nedir?

**CORS (Cross-Origin Resource Sharing)**, farklı origin'lerden (domain, protokol veya port) gelen isteklere izin verme mekanizmasıdır.

**Origin Nedir?**
- **Origin:** Protokol + Domain + Port kombinasyonu
- **Örnek:** `https://www.example.com:443` bir origin'dir
- **Aynı Origin:** Protokol, domain ve port aynı ise
- **Farklı Origin:** Protokol, domain veya port farklı ise

### ECommerce API'de Neden CORS Gerekli?

ECommerce API'niz şu senaryolarda farklı origin'lerden istek alacak:

1. **Frontend Uygulaması:**
   - React, Vue, Angular gibi frontend framework'leri
   - Örnek: `http://localhost:3000` → `https://api.ecommerce.com`

2. **Mobil Uygulama:**
   - React Native, Flutter vb.
   - Farklı domain'den API çağrıları

3. **Başka Web Siteleri:**
   - Partner sitelerden ilan listeleme
   - Embed edilmiş ilan widget'ları

**CORS olmadan:** Browser, farklı origin'den gelen istekleri otomatik olarak engeller.

---

## 📚 CORS Nedir? (Detaylı)

### Same-Origin Policy (Aynı Origin Politikası)

Browser'lar varsayılan olarak **Same-Origin Policy** uygular:

**Kural:** Bir web sayfası, sadece kendi origin'inden gelen kaynaklara erişebilir.

**Örnek:**
- ✅ `https://example.com/page1` → `https://example.com/api` (Aynı origin, izin verilir)
- ❌ `https://example.com/page1` → `https://api.example.com` (Farklı domain, engellenir)
- ❌ `http://example.com/page1` → `https://example.com/api` (Farklı protokol, engellenir)
- ❌ `https://example.com:80` → `https://example.com:443` (Farklı port, engellenir)

**Neden Bu Politika Var?**
- Güvenlik: Kötü amaçlı sitelerin başka sitelerin kaynaklarına erişmesini engeller
- Cookie ve Session koruması
- CSRF (Cross-Site Request Forgery) saldırılarına karşı koruma

### CORS Nasıl Çalışır?

CORS, browser'ın bu kısıtlamayı güvenli bir şekilde aşmasını sağlar:

**1. Preflight Request (OPTIONS İsteği):**
```
Browser → API: OPTIONS /api/properties
           Origin: https://myapp.com
           Access-Control-Request-Method: POST
           Access-Control-Request-Headers: Content-Type, Authorization

API → Browser: 200 OK
              Access-Control-Allow-Origin: https://myapp.com
              Access-Control-Allow-Methods: GET, POST, PUT, DELETE
              Access-Control-Allow-Headers: Content-Type, Authorization
              Access-Control-Allow-Credentials: true
```

**2. Actual Request (Gerçek İstek):**
```
Browser → API: POST /api/properties
           Origin: https://myapp.com
           Content-Type: application/json
           Authorization: Bearer token

API → Browser: 200 OK
              Access-Control-Allow-Origin: https://myapp.com
              Data: {...}
```

### CORS Header'ları

**Request Header'ları (Browser Gönderir):**
- **Origin:** İsteği gönderen sayfanın origin'i
- **Access-Control-Request-Method:** Kullanılacak HTTP method (preflight'ta)
- **Access-Control-Request-Headers:** Kullanılacak header'lar (preflight'ta)

**Response Header'ları (API Gönderir):**
- **Access-Control-Allow-Origin:** İzin verilen origin'ler
- **Access-Control-Allow-Methods:** İzin verilen HTTP method'ları
- **Access-Control-Allow-Headers:** İzin verilen header'lar
- **Access-Control-Allow-Credentials:** Cookie/credential gönderimi izni
- **Access-Control-Max-Age:** Preflight cache süresi (saniye)

---

## 🔒 Neden CORS Gerekli?

### Senaryo 1: Frontend-Backend Ayrımı

**Modern Web Uygulaması:**
- Frontend: `http://localhost:3000` (React)
- Backend: `http://localhost:5070` (ASP.NET Core API)

**Problem:** Farklı port → Farklı origin → Browser engeller

**Çözüm:** CORS ile `localhost:3000`'den gelen isteklere izin ver

### Senaryo 2: Production Ortamı

**Production:**
- Frontend: `https://ecommerce.com`
- Backend: `https://api.ecommerce.com`

**Problem:** Farklı subdomain → Farklı origin → Browser engeller

**Çözüm:** CORS ile `ecommerce.com` origin'ine izin ver

### Senaryo 3: Güvenlik

**CORS Olmadan:**
- ❌ Herhangi bir site sizin API'nize istek atabilir
- ❌ Cookie'ler ve token'lar çalınabilir
- ❌ CSRF saldırıları yapılabilir

**CORS ile:**
- ✅ Sadece izin verdiğiniz origin'ler istek atabilir
- ✅ Güvenli bir şekilde farklı origin'lerden erişim sağlanır
- ✅ Güvenlik kontrolü yapılır

---

## 🛠️ ECommerce API için CORS Yapılandırması

### Adım 1: CorsConfig Sınıfı Oluşturma

**CorsConfig**, CORS ayarlarını configuration'dan okumak için kullanılır:

**ECommerce.Business/Configs/CorsConfig.cs:**

```csharp
namespace ECommerce.Business.Configs;

public class CorsConfig
{
    public string[] AllowedOrigins { get; set; } = [];
    public string[] AllowedMethods { get; set; } = [];
    public string[] AllowedHeaders { get; set; } = [];
    public bool AllowCredentials { get; set; } = true;
}
```

**Açıklamalar:**
- **AllowedOrigins:** İzin verilen origin'lerin listesi (array)
- **AllowedMethods:** İzin verilen HTTP method'ları (GET, POST, vb.)
- **AllowedHeaders:** İzin verilen request header'ları
- **AllowCredentials:** Cookie ve Authorization header gönderimi izni

### Adım 2: appsettings.json Yapılandırması

**ECommerce.API/appsettings.json:**

```json
{
  "CorsSettings": {
    "AllowedOrigins": [
      "http://localhost:3000",
      "http://localhost:5040",
      "https://ecommerce-frontend.com",
      "https://www.ecommerce-frontend.com"
    ],
    "AllowedMethods": [
      "GET",
      "POST",
      "PUT",
      "DELETE",
      "PATCH"
    ],
    "AllowedHeaders": [
      "Content-Type",
      "Authorization"
    ],
    "AllowCredentials": true
  }
}
```

**Açıklamalar:**

**AllowedOrigins:**
- `http://localhost:3000` - Local development (React, Vue vb.)
- `http://localhost:5040` - Alternatif local development port
- `https://ecommerce-frontend.com` - Production frontend
- `https://www.ecommerce-frontend.com` - Production frontend (www ile)

**AllowedMethods:**
- ECommerce API'de kullanılan tüm HTTP method'ları
- GET: İlan listeleme, detay getirme
- POST: Yeni ilan oluşturma, inquiry gönderme
- PUT: İlan güncelleme
- DELETE: İlan silme
- PATCH: Kısmi güncelleme (opsiyonel)

**AllowedHeaders:**
- `Content-Type`: JSON gönderimi için (`application/json`)
- `Authorization`: JWT token gönderimi için (`Bearer token`)

**AllowCredentials:**
- `true`: Cookie ve Authorization header gönderimi için gerekli
- JWT authentication kullanıyorsak genellikle `true` olmalı

### Adım 3: Development vs Production

**appsettings.Development.json:**

```json
{
  "CorsSettings": {
    "AllowedOrigins": [],
    "AllowedMethods": [
      "GET",
      "POST",
      "PUT",
      "DELETE",
      "PATCH"
    ],
    "AllowedHeaders": [
      "Content-Type",
      "Authorization"
    ],
    "AllowCredentials": true
  }
}
```

**Not:** Development'ta `AllowedOrigins` boş array ise, `AllowAnyOrigin()` kullanılır (kolaylık için).

---

## ⚙️ Program.cs Yapılandırması

### CORS Servis Kaydı

```csharp
using ECommerce.Business.Configs;

var builder = WebApplication.CreateBuilder(args);

// CORS Configuration'dan oku
var corsConfig = builder.Configuration.GetSection("CorsSettings").Get<CorsConfig>();

// CORS servislerini kaydet
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedSpecificOrigins", policy =>
    {
        // Eğer AllowedOrigins dolu ise, belirtilen origin'lere izin ver
        if (corsConfig!.AllowedOrigins.Length > 0)
        {
            policy
                .WithOrigins(corsConfig.AllowedOrigins)  // İzin verilen origin'ler
                .WithMethods(corsConfig.AllowedMethods)  // İzin verilen method'lar
                .WithHeaders(corsConfig.AllowedHeaders); // İzin verilen header'lar
        }
        else
        {
            // Development ortamı için: Her origin'e izin ver
            policy
                .AllowAnyOrigin()   // Tüm origin'lere izin
                .AllowAnyMethod()   // Tüm method'lara izin
                .AllowAnyHeader();  // Tüm header'lara izin
        }
        
        // Credentials (cookie, authorization header) izni
        if (corsConfig.AllowCredentials)
        {
            policy.AllowCredentials();  // Cookie ve Authorization header gönderimi için
        }
    });
});

var app = builder.Build();
```

### Detaylı Açıklamalar:

**1. Configuration'dan Okuma:**
```csharp
var corsConfig = builder.Configuration.GetSection("CorsSettings").Get<CorsConfig>();
```
- **GetSection():** `appsettings.json`'dan `CorsSettings` bölümünü alır
- **Get<CorsConfig>():** JSON'ı `CorsConfig` sınıfına map eder (strongly-typed)
- **Sonuç:** Configuration değerleri `corsConfig` objesine yüklenir

**2. AddCors() ve AddPolicy():**
```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedSpecificOrigins", policy => { ... });
});
```
- **AddCors():** CORS servislerini DI container'a ekler
- **AddPolicy():** Bir CORS policy'si oluşturur ve isimlendirir
- **Policy Adı:** `"AllowedSpecificOrigins"` - sonra middleware'de kullanılacak

**3. WithOrigins():**
```csharp
policy.WithOrigins(corsConfig.AllowedOrigins)
```
- **Amaç:** İzin verilen origin'leri belirtir
- **Parametre:** String array (örnek: `["http://localhost:3000", "https://example.com"]`)
- **Sonuç:** Sadece bu origin'lerden gelen isteklere izin verilir
- **Önemli:** Protokol, domain ve port tam olarak eşleşmeli

**4. WithMethods():**
```csharp
policy.WithMethods(corsConfig.AllowedMethods)
```
- **Amaç:** İzin verilen HTTP method'larını belirtir
- **Parametre:** String array (örnek: `["GET", "POST", "PUT", "DELETE"]`)
- **Sonuç:** Sadece bu method'lara izin verilir
- **Preflight:** Browser, OPTIONS isteğinde bu method'ları kontrol eder

**5. WithHeaders():**
```csharp
policy.WithHeaders(corsConfig.AllowedHeaders)
```
- **Amaç:** İzin verilen request header'larını belirtir
- **Parametre:** String array (örnek: `["Content-Type", "Authorization"]`)
- **Sonuç:** Sadece bu header'lar gönderilebilir
- **Önemli:** `Authorization` header'ı JWT token için gereklidir

**6. AllowCredentials():**
```csharp
if (corsConfig.AllowCredentials)
{
    policy.AllowCredentials();
}
```
- **Amaç:** Cookie ve Authorization header gönderimine izin verir
- **Ne Zaman Gerekli:** JWT authentication kullanıyorsak gerekli
- **Kısıtlama:** `AllowCredentials()` kullanılırsa `AllowAnyOrigin()` kullanılamaz (güvenlik)

**7. AllowAnyOrigin() (Development için):**
```csharp
else
{
    policy.AllowAnyOrigin()
          .AllowAnyMethod()
          .AllowAnyHeader();
}
```
- **Amaç:** Development ortamında kolaylık için tüm origin'lere izin verir
- **Kullanım:** `AllowedOrigins` boş array ise (development'ta)
- **Dikkat:** Production'da kullanılmamalı (güvenlik riski)

### CORS Middleware Kullanımı

```csharp
var app = builder.Build();

// Middleware pipeline
app.UseMiddleware<ExceptionHandlingMiddleware>();  // 1. Exception handling
app.UseHttpsRedirection();                         // 2. HTTPS yönlendirme
app.UseCors("AllowedSpecificOrigins");             // 3. CORS (Authentication'tan ÖNCE!)
app.UseAuthentication();                           // 4. Authentication
app.UseAuthorization();                            // 5. Authorization
app.MapControllers();                              // 6. Controllers

app.Run();
```

**Önemli:** `UseCors()` mutlaka `UseAuthentication()` ve `UseAuthorization()`'dan **ÖNCE** olmalı!

**Neden?**
- Preflight request'ler (OPTIONS) authentication gerektirmez
- CORS kontrolü, authentication'dan önce yapılmalı
- Aksi halde CORS hatası authentication hatasından önce gelmeli

---

## 🔄 Middleware Sırası

### Doğru Middleware Sırası:

```
1. ExceptionHandlingMiddleware  → Tüm hataları yakalar
2. SecurityHeadersMiddleware    → Security header'ları ekler
3. UseHttpsRedirection()        → HTTP → HTTPS yönlendirme
4. UseRateLimiter()             → Rate limiting kontrolü
5. UseCors()                    → CORS kontrolü (Authentication'dan ÖNCE!)
6. UseAuthentication()          → JWT token doğrulama
7. UseAuthorization()           → Role/Policy kontrolü
8. UseResponseCaching()         → Response caching
9. UseResponseCompression()     → Response compression
10. MapControllers()            → Route'lar
```

**CORS Neden Bu Sırada?**
- Preflight request'ler (OPTIONS) authentication gerektirmez
- Browser, CORS kontrolünü authentication'dan önce yapar
- CORS başarısız olursa, request authentication'a ulaşmaz

---

## 🧪 Test Senaryoları

### Senaryo 1: CORS Başarılı (İzin Verilen Origin)

**Frontend (http://localhost:3000):**
```javascript
fetch('http://localhost:5070/api/properties', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token123'
  }
})
```

**Beklenen Sonuç:**
- ✅ Status Code: 200 OK
- ✅ Response header'ında: `Access-Control-Allow-Origin: http://localhost:3000`
- ✅ Data döner

**Browser Console:**
```
✅ Request successful
```

### Senaryo 2: CORS Başarısız (İzin Verilmeyen Origin)

**Farklı Origin (http://localhost:8080):**
```javascript
fetch('http://localhost:5070/api/properties', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json'
  }
})
```

**Beklenen Sonuç:**
- ❌ Browser Console'da CORS hatası:
```
❌ Access to fetch at 'http://localhost:5070/api/properties' 
   from origin 'http://localhost:8080' has been blocked by CORS policy: 
   No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

**Network Tab:**
- Request gönderilir
- Response gelir (200 OK) ama browser response'u blocklar
- Response header'ında `Access-Control-Allow-Origin` yok veya farklı origin

### Senaryo 3: Preflight Request (OPTIONS)

**Complex Request (POST + Custom Headers):**
```javascript
fetch('http://localhost:5070/api/properties', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token123'
  },
  body: JSON.stringify({ title: 'Test', price: 1000000 })
})
```

**Browser İki İstek Yapar:**

**1. Preflight Request (OPTIONS):**
```
OPTIONS /api/properties HTTP/1.1
Host: localhost:5070
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization
```

**API Response:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

**2. Actual Request (POST):**
```
POST /api/properties HTTP/1.1
Host: localhost:5070
Origin: http://localhost:3000
Content-Type: application/json
Authorization: Bearer token123
```

**API Response:**
```
HTTP/1.1 201 Created
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Credentials: true
Content-Type: application/json
{ "success": true, "data": {...} }
```

### Senaryo 4: curl ile Test

**Preflight Request:**
```bash
curl -X OPTIONS http://localhost:5070/api/properties \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type, Authorization" \
  -v
```

**Beklenen Response Headers:**
```
< Access-Control-Allow-Origin: http://localhost:3000
< Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH
< Access-Control-Allow-Headers: Content-Type, Authorization
< Access-Control-Allow-Credentials: true
```

**Actual Request:**
```bash
curl -X GET http://localhost:5070/api/properties \
  -H "Origin: http://localhost:3000" \
  -H "Authorization: Bearer token123" \
  -v
```

**Beklenen Response Headers:**
```
< Access-Control-Allow-Origin: http://localhost:3000
< Access-Control-Allow-Credentials: true
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **CORS Sadece Browser'larda Çalışır:**
   - Server-to-server isteklerde CORS yoktur
   - Postman, curl gibi araçlarda CORS kontrolü yoktur
   - Sadece browser'lar CORS uygular

2. **Preflight Request:**
   - Complex request'ler için browser otomatik OPTIONS isteği gönderir
   - Simple request'ler için preflight yok (GET, HEAD, POST with simple content-type)

3. **AllowCredentials Kısıtlaması:**
   - `AllowCredentials()` kullanılırsa `AllowAnyOrigin()` kullanılamaz
   - Spesifik origin'ler belirtilmeli
   - Güvenlik nedeniyle (cookie çalınmasını önlemek için)

4. **Wildcard Origin:**
   - `Access-Control-Allow-Origin: *` → Tüm origin'lere izin
   - `AllowCredentials()` ile kullanılamaz
   - Production'da önerilmez (güvenlik)

5. **Middleware Sırası:**
   - `UseCors()` mutlaka `UseAuthentication()`'dan önce olmalı
   - Preflight request'ler authentication gerektirmez

### Sık Yapılan Hatalar

1. **CORS'u Authentication'dan Sonra Koymak:**
   - ❌ Yanlış: `UseAuthentication()` → `UseCors()`
   - ✅ Doğru: `UseCors()` → `UseAuthentication()`

2. **AllowAnyOrigin() ile AllowCredentials() Birlikte:**
   - ❌ Yanlış: `AllowAnyOrigin().AllowCredentials()`
   - ✅ Doğru: `WithOrigins([...]).AllowCredentials()`

3. **Origin Formatı:**
   - ❌ Yanlış: `"localhost:3000"` (protokol yok)
   - ✅ Doğru: `"http://localhost:3000"`

4. **Header Adları Büyük/Küçük Harf Duyarlı:**
   - Browser'lar genellikle küçük harfe çevirir
   - ASP.NET Core case-insensitive ama dikkatli olun

5. **Development'ta AllowAnyOrigin():**
   - Development'ta kullanılabilir (kolaylık için)
   - Production'da mutlaka spesifik origin'ler kullanın

### İpuçları

1. **Development için:**
   - `AllowedOrigins: []` → `AllowAnyOrigin()` kullan (kolaylık)
   - Veya localhost port'larını ekle

2. **Production için:**
   - Mutlaka spesifik origin'ler belirtin
   - Wildcard (`*`) kullanmayın (güvenlik)

3. **Debugging:**
   - Browser DevTools → Network tab → Response Headers
   - CORS header'larını kontrol edin
   - Console'da CORS hatalarını okuyun

4. **Testing:**
   - Browser'dan test edin (curl'da CORS yok)
   - Farklı origin'lerden test edin
   - Preflight request'leri kontrol edin (OPTIONS)

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **CORS Nedir:** Cross-Origin Resource Sharing, farklı origin'lerden isteklere izin verme
2. ✅ **Neden Gerekli:** Frontend-backend ayrımı, güvenlik
3. ✅ **Nasıl Çalışır:** Preflight request, CORS header'ları
4. ✅ **Yapılandırma:** CorsConfig, appsettings.json, Program.cs
5. ✅ **Middleware Sırası:** CORS, Authentication'dan önce
6. ✅ **Test:** Browser, curl, farklı origin'ler

**Sonraki Adım:** Rate Limiting dersine geçebiliriz.

---

**Başarılar! 🚀**

