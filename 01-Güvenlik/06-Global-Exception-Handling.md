# Global Exception Handling
## ECommerce API - Güvenlik

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Global Exception Handling Nedir?](#global-exception-handling-nedir)
3. [Neden Global Exception Handling?](#neden-global-exception-handling)
4. [Custom Exception Sınıfları](#custom-exception-sınıfları)
5. [ExceptionHandlingMiddleware](#exceptionhandlingmiddleware)
6. [Program.cs Yapılandırması](#programcs-yapılandırması)
7. [Service Katmanında Kullanım](#service-katmanında-kullanım)
8. [Test Senaryoları](#test-senaryoları)
9. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Global Exception Handling Nedir?

**Global Exception Handling**, uygulamadaki tüm exception'ları merkezi bir middleware'de yakalayıp, tutarlı bir format'ta response döndürme işlemidir.

**Avantajlar:**
- ✅ Tutarlı error response formatı
- ✅ Kod tekrarını önler
- ✅ Güvenlik (detaylı hata mesajlarını gizler)
- ✅ Logging merkezi

### ECommerce API'de Neden Gerekli?

1. **Tutarlı Response Formatı:**
   - Tüm hatalar `ResponseDto<T>` formatında döner
   - Client tarafında kolay işleme

2. **Güvenlik:**
   - Production'da detaylı hata mesajları gizlenir
   - Stack trace gösterilmez

3. **Logging:**
   - Tüm exception'lar merkezi olarak loglanır
   - Monitoring ve debugging kolaylaşır

---

## 📚 Custom Exception Sınıfları

### BusinessException (Base Class)

**ECommerce.Business/Exceptions/BusinessException.cs:**

```csharp
namespace ECommerce.Business.Exceptions;

public class BusinessException : Exception
{
    public int StatusCode { get; }
    public string ErrorCode { get; }

    public BusinessException(string message, int statusCode = 400, string errorCode = "BUSINESS_ERROR") 
        : base(message)
    {
        StatusCode = statusCode;
        ErrorCode = errorCode;
    }

    public BusinessException(string message, Exception innerException, int statusCode = 400, string errorCode = "BUSINESS_ERROR") 
        : base(message, innerException)
    {
        StatusCode = statusCode;
        ErrorCode = errorCode;
    }
}
```

**Açıklama:**
- **StatusCode:** HTTP status code (400, 404, 500 vb.)
- **ErrorCode:** Hata kodu (loglama ve debugging için)

### NotFoundException

```csharp
public class NotFoundException : BusinessException
{
    public NotFoundException(string resourceName, object key) 
        : base($"'{key}' id'li {resourceName} bulunamadı!", 404, "NOT_FOUND_ERROR")
    {
    }

    public NotFoundException(string message) 
        : base(message, 404, "NOT_FOUND_ERROR")
    {
    }
}
```

**Kullanım:**
```csharp
throw new NotFoundException("Property", propertyId);
// Mesaj: "'123' id'li Property bulunamadı!"
```

### ValidationException

```csharp
public class ValidationException : BusinessException
{
    public Dictionary<string, string[]> Errors { get; }

    public ValidationException(Dictionary<string, string[]> errors) 
        : base("Doğrulama hatası!", 400, "VALIDATION_ERROR")
    {
        Errors = errors;
    }

    public ValidationException(string message) 
        : base(message, 400, "VALIDATION_ERROR")
    {
        Errors = new Dictionary<string, string[]>();
    }
}
```

**Kullanım:**
```csharp
var errors = new Dictionary<string, string[]>
{
    { "Title", new[] { "Başlık zorunludur!" } },
    { "Price", new[] { "Fiyat 0'dan büyük olmalıdır!" } }
};
throw new ValidationException(errors);
```

### UnauthorizedException

```csharp
public class UnauthorizedException : BusinessException
{
    public UnauthorizedException(string message = "Yetkisiz erişim!") 
        : base(message, 401, "UNAUTHORIZED_ERROR")
    {
    }
}
```

---

## ⚙️ ExceptionHandlingMiddleware

**ECommerce.API/Middleware/ExceptionHandlingMiddleware.cs:**

```csharp
using System.Net;
using System.Text.Json;
using ECommerce.Business.DTOs.ResponseDtos;
using ECommerce.Business.Exceptions;

namespace ECommerce.API.Middleware;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IWebHostEnvironment _environment;

    public ExceptionHandlingMiddleware(
        RequestDelegate next, 
        ILogger<ExceptionHandlingMiddleware> logger,
        IWebHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        var response = context.Response;
        ResponseDto<object> errorResponse;

        switch (exception)
        {
            case NotFoundException notFoundException:
                response.StatusCode = (int)HttpStatusCode.NotFound;
                errorResponse = ResponseDto<object>.Fail(
                    notFoundException.Message, 
                    (int)HttpStatusCode.NotFound
                );
                _logger.LogWarning(exception, "Kaynak bulunamadı: {Message}", notFoundException.Message);
                break;

            case ValidationException validationException:
                response.StatusCode = (int)HttpStatusCode.BadRequest;
                var validationMessage = validationException.Errors.Any()
                    ? string.Join("; ", validationException.Errors.SelectMany(e => 
                        e.Value.Select(v => $"{e.Key}: {v}")))
                    : validationException.Message;
                errorResponse = ResponseDto<object>.Fail(validationMessage, (int)HttpStatusCode.BadRequest);
                _logger.LogWarning(exception, "Doğrulama hatası: {Message}", validationException.Message);
                break;

            case UnauthorizedException unauthorizedException:
                response.StatusCode = (int)HttpStatusCode.Unauthorized;
                errorResponse = ResponseDto<object>.Fail(
                    unauthorizedException.Message, 
                    (int)HttpStatusCode.Unauthorized
                );
                _logger.LogWarning(exception, "Yetki hatası: {Message}", unauthorizedException.Message);
                break;

            case BusinessException businessException:
                response.StatusCode = businessException.StatusCode;
                var businessErrorMessage = _environment.IsDevelopment()
                    ? $"[{businessException.ErrorCode}] {businessException.Message}"
                    : businessException.Message;
                errorResponse = ResponseDto<object>.Fail(businessErrorMessage, businessException.StatusCode);
                _logger.LogWarning(exception, "Servis hatası: {ErrorCode} - {Message}", 
                    businessException.ErrorCode, businessException.Message);
                break;

            default:
                response.StatusCode = (int)HttpStatusCode.InternalServerError;
                var errorMessage = _environment.IsDevelopment()
                    ? exception.Message
                    : "Bir hata oluştu.";
                errorResponse = ResponseDto<object>.Fail(
                    errorMessage, 
                    (int)HttpStatusCode.InternalServerError
                );
                _logger.LogError(exception, "Beklenmedik hata: {Message}", exception.Message);
                break;
        }

        var options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };

        var jsonResponse = JsonSerializer.Serialize(errorResponse, options);
        await response.WriteAsync(jsonResponse);
    }
}
```

**Açıklamalar:**

1. **InvokeAsync:** Middleware pipeline'ında çalışır, tüm exception'ları yakalar
2. **HandleExceptionAsync:** Exception tipine göre response oluşturur
3. **Switch Expression:** Exception tipine göre farklı işlemler
4. **Environment Check:** Development'ta detaylı, production'da generic mesaj
5. **Logging:** Her exception loglanır (structured logging)

---

## ⚙️ Program.cs Yapılandırması

```csharp
using ECommerce.API.Middleware;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Exception Handling Middleware (EN ÖNDE!)
app.UseMiddleware<ExceptionHandlingMiddleware>();

// Diğer middleware'ler...
app.UseHttpsRedirection();
app.UseCors("AllowedSpecificOrigins");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Önemli:** ExceptionHandlingMiddleware **mutlaka en önde** olmalı!

---

## 🎮 Service Katmanında Kullanım

### Eski Yöntem (ResponseDto.Fail)

```csharp
// ❌ ESKİ YÖNTEM
public async Task<ResponseDto<PropertyDto>> GetByIdAsync(int id)
{
    var property = await _repository.GetAsync(id);
    if (property == null)
    {
        return ResponseDto<PropertyDto>.Fail("Property bulunamadı!", 404);
    }
    return ResponseDto<PropertyDto>.Success(property, 200);
}
```

### Yeni Yöntem (Custom Exception)

```csharp
// ✅ YENİ YÖNTEM
public async Task<ResponseDto<PropertyDto>> GetByIdAsync(int id)
{
    var property = await _repository.GetAsync(id);
    if (property == null)
    {
        throw new NotFoundException("Property", id);
    }
    return ResponseDto<PropertyDto>.Success(property, 200);
}
```

**Avantajlar:**
- Kod daha temiz
- Exception handling merkezi
- Tutarlı response formatı

---

## 🧪 Test Senaryoları

### Senaryo 1: NotFoundException

```bash
curl http://localhost:5070/api/properties/99999
```

**Beklenen Response:**
```json
{
  "success": false,
  "message": "'99999' id'li Property bulunamadı!",
  "data": null
}
```
**Status Code:** 404

### Senaryo 2: ValidationException

```bash
curl -X POST http://localhost:5070/api/properties \
  -H "Content-Type: application/json" \
  -d '{"title":""}'
```

**Beklenen Response:**
```json
{
  "success": false,
  "message": "Title: Başlık zorunludur!; Price: Fiyat zorunludur!",
  "data": null
}
```
**Status Code:** 400

### Senaryo 3: Unhandled Exception

**Kod:**
```csharp
throw new Exception("Beklenmedik hata!");
```

**Development Response:**
```json
{
  "success": false,
  "message": "Beklenmedik hata!",
  "data": null
}
```

**Production Response:**
```json
{
  "success": false,
  "message": "Bir hata oluştu.",
  "data": null
}
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Middleware Sırası:** ExceptionHandlingMiddleware en önde
2. **Custom Exceptions:** İhtiyaca göre yeni exception sınıfları eklenebilir
3. **Environment Check:** Production'da detaylı mesaj gizlenir
4. **Logging:** Tüm exception'lar loglanır

### Sık Yapılan Hatalar

1. **Middleware Sırası:** ❌ ExceptionHandlingMiddleware en önde değil
2. **Try-Catch Unutmak:** ❌ Service'te try-catch kullanmaya gerek yok (middleware yakalar)
3. **Generic Exception:** ❌ Mümkünse spesifik exception kullan

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Global Exception Handling:** Merkezi exception yönetimi
2. ✅ **Custom Exceptions:** NotFoundException, ValidationException, vb.
3. ✅ **ExceptionHandlingMiddleware:** Merkezi exception handling
4. ✅ **Service Kullanımı:** throw new Exception() pattern
5. ✅ **Environment Check:** Development vs Production

**Sonraki Adım:** Performans bölümüne geçebiliriz!

---

**Başarılar! 🚀**

