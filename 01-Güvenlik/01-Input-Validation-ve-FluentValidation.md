# Input Validation ve FluentValidation
## ECommerce API - Güvenlik Dersleri

**Seviye:** Orta  
**Hedef:** FluentValidation kullanarak güvenli input validation uygulama

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Teorik Açıklama](#teorik-açıklama)
3. [FluentValidation Kurulumu](#fluentvalidation-kurulumu)
4. [ECommerce API için Validator'lar](#ecommerce-api-için-validators)
5. [Program.cs Yapılandırması](#programcs-yapılandırması)
6. [ValidationFilter ile Entegrasyon](#validationfilter-ile-entegrasyon)
7. [Test Senaryoları](#test-senaryoları)
8. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Input Validation Nedir?

**Input Validation (Girdi Doğrulama)**, kullanıcıların API'ye gönderdiği verilerin doğru, güvenli ve geçerli olup olmadığını kontrol etme işlemidir.

### Neden Önemlidir?

1. **Güvenlik:** Zararlı verilerin sisteme girmesini engeller (SQL Injection, XSS saldırıları)
2. **Veri Bütünlüğü:** Yanlış veya eksik verilerin veritabanına kaydedilmesini önler
3. **Kullanıcı Deneyimi:** Hatalı veri girişlerinde anlamlı hata mesajları gösterir
4. **Sistem Stabilitesi:** Uygunsuz veriler nedeniyle oluşabilecek hataları önler

### ECommerce API'de Neden FluentValidation?

ECommerce API'de kullanıcılar şu verileri gönderebilir:
- **İlan Bilgileri:** Fiyat, oda sayısı, alan gibi sayısal değerler
- **İletişim Bilgileri:** E-posta, telefon gibi format kontrolü gereken veriler
- **Adres Bilgileri:** Şehir, ilçe, adres gibi metin alanları
- **Kullanıcı Bilgileri:** Şifre, e-posta, isim gibi hassas veriler

Bu verilerin hepsinin doğru formatta, güvenli ve geçerli olması kritiktir.

---

## 📚 Teorik Açıklama

### DataAnnotations vs FluentValidation

#### DataAnnotations (ASP.NET Core Varsayılan)

```csharp
public class PropertyCreateDto
{
    [Required(ErrorMessage = "Başlık zorunludur!")]
    [MinLength(3, ErrorMessage = "Başlık en az 3 karakter olmalıdır!")]
    public string Title { get; set; }
}
```

**Avantajları:**
- ✅ ASP.NET Core'da built-in (ek paket gerekmez)
- ✅ Basit kullanım
- ✅ DTO üzerinde direkt attribute'lar

**Dezavantajları:**
- ❌ Karmaşık validation kuralları zor
- ❌ Custom validation logic için ekstra kod gerekir
- ❌ Test yazmak zor
- ❌ Business logic ile karışır (DTO'lar kirli olur)

#### FluentValidation

```csharp
public class PropertyCreateDtoValidator : AbstractValidator<PropertyCreateDto>
{
    public PropertyCreateDtoValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("Başlık zorunludur!")
            .MinimumLength(3).WithMessage("Başlık en az 3 karakter olmalıdır!");
    }
}
```

**Avantajları:**
- ✅ Ayrı validator sınıfları (separation of concerns)
- ✅ Karmaşık validation kuralları kolay
- ✅ Test yazmak kolay
- ✅ DTO'lar temiz kalır
- ✅ Reusability (tekrar kullanılabilirlik)
- ✅ Fluent API syntax (okunabilirlik)

**Dezavantajları:**
- ❌ Ek paket gerekir
- ❌ Biraz daha fazla kod

### FluentValidation Nasıl Çalışır?

1. **Validator Sınıfı:** Her DTO için bir validator sınıfı oluşturulur
2. **RuleFor:** Her property için validation kuralları tanımlanır
3. **Otomatik Çalışma:** FluentValidation, request geldiğinde otomatik olarak validator'ları çalıştırır
4. **Hata Toplama:** Tüm hatalar toplanır ve ValidationException olarak fırlatılır
5. **Exception Handling:** Global exception handler bu hatayı yakalar ve ResponseDto formatında döner

### Validation Pipeline (Doğrulama İşlem Akışı)

```
1. Client → HTTP Request (PropertyCreateDto)
   ↓
2. Controller → Action Method'a girer
   ↓
3. FluentValidation → Validator çalışır
   ↓
4a. Geçerli ise → Service Layer'e geçer
4b. Geçersiz ise → ValidationException fırlatılır
   ↓
5. ExceptionHandlingMiddleware → Hatayı yakalar
   ↓
6. ResponseDto formatında hata response döner
```

---

## 🛠️ FluentValidation Kurulumu

### Adım 1: NuGet Paketi Yükleme

Business katmanına (ECommerce.Business) FluentValidation paketini ekleyin:

```bash
dotnet add ECommerce.Business package FluentValidation
```

API katmanına (ECommerce.API) FluentValidation.AspNetCore paketini ekleyin:

```bash
dotnet add ECommerce.API package FluentValidation.AspNetCore
```

**Paket Açıklamaları:**
- **FluentValidation:** Core validation kütüphanesi (Business katmanında)
- **FluentValidation.AspNetCore:** ASP.NET Core entegrasyonu (API katmanında)

---

## 📝 ECommerce API için Validator'lar

### 1. PropertyCreateDtoValidator

**PropertyCreateDtoValidator**, yeni bir emlak ilanı oluştururken gönderilen verileri doğrular.

#### Validator Sınıfı:

```csharp
using ECommerce.Business.DTOs;
using FluentValidation;

namespace ECommerce.Business.Validators;

public class PropertyCreateDtoValidator : AbstractValidator<PropertyCreateDto>
{
    public PropertyCreateDtoValidator()
    {
        // Başlık Validasyonu
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("İlan başlığı zorunludur!")
            .MinimumLength(3).WithMessage("İlan başlığı en az 3 karakter olmalıdır!")
            .MaximumLength(200).WithMessage("İlan başlığı en fazla 200 karakter olabilir!")
            .Matches(@"^[a-zA-Z0-9ğüşıöçĞÜŞİÖÇ\s\-,.]+$")
            .WithMessage("İlan başlığı sadece harf, rakam, boşluk ve özel karakterler (-,.,) içerebilir!");

        // Açıklama Validasyonu
        RuleFor(x => x.Description)
            .NotEmpty().WithMessage("İlan açıklaması zorunludur!")
            .MinimumLength(10).WithMessage("İlan açıklaması en az 10 karakter olmalıdır!")
            .MaximumLength(5000).WithMessage("İlan açıklaması en fazla 5000 karakter olabilir!");

        // Fiyat Validasyonu
        RuleFor(x => x.Price)
            .NotEmpty().WithMessage("Fiyat zorunludur!")
            .GreaterThan(0).WithMessage("Fiyat 0'dan büyük olmalıdır!")
            .LessThanOrEqualTo(999999999).WithMessage("Fiyat en fazla 999.999.999 TL olabilir!");

        // Adres Validasyonu
        RuleFor(x => x.Address)
            .NotEmpty().WithMessage("Adres zorunludur!")
            .MinimumLength(5).WithMessage("Adres en az 5 karakter olmalıdır!")
            .MaximumLength(500).WithMessage("Adres en fazla 500 karakter olabilir!");

        // Şehir Validasyonu
        RuleFor(x => x.City)
            .NotEmpty().WithMessage("Şehir bilgisi zorunludur!")
            .MinimumLength(2).WithMessage("Şehir adı en az 2 karakter olmalıdır!")
            .MaximumLength(100).WithMessage("Şehir adı en fazla 100 karakter olabilir!")
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]+$")
            .WithMessage("Şehir adı sadece harf ve boşluk içerebilir!");

        // İlçe Validasyonu (Opsiyonel)
        RuleFor(x => x.District)
            .MaximumLength(100).WithMessage("İlçe adı en fazla 100 karakter olabilir!")
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]*$")
            .When(x => !string.IsNullOrEmpty(x.District))
            .WithMessage("İlçe adı sadece harf ve boşluk içerebilir!");

        // Oda Sayısı Validasyonu
        RuleFor(x => x.Rooms)
            .NotEmpty().WithMessage("Oda sayısı zorunludur!")
            .GreaterThan(0).WithMessage("Oda sayısı 0'dan büyük olmalıdır!")
            .LessThanOrEqualTo(20).WithMessage("Oda sayısı en fazla 20 olabilir!");

        // Banyo Sayısı Validasyonu (Opsiyonel)
        RuleFor(x => x.Bathrooms)
            .GreaterThan(0).WithMessage("Banyo sayısı 0'dan büyük olmalıdır!")
            .LessThanOrEqualTo(10).WithMessage("Banyo sayısı en fazla 10 olabilir!")
            .When(x => x.Bathrooms.HasValue);

        // Alan (m²) Validasyonu
        RuleFor(x => x.Area)
            .NotEmpty().WithMessage("Alan bilgisi zorunludur!")
            .GreaterThan(0).WithMessage("Alan 0'dan büyük olmalıdır!")
            .LessThanOrEqualTo(100000).WithMessage("Alan en fazla 100.000 m² olabilir!");

        // Kat Numarası Validasyonu
        RuleFor(x => x.Floor)
            .NotEmpty().WithMessage("Kat numarası zorunludur!")
            .GreaterThanOrEqualTo(-10).WithMessage("Kat numarası en az -10 (bodrum) olabilir!")
            .LessThanOrEqualTo(100).WithMessage("Kat numarası en fazla 100 olabilir!");

        // Toplam Kat Sayısı Validasyonu (Opsiyonel)
        RuleFor(x => x.TotalFloors)
            .GreaterThan(0).WithMessage("Toplam kat sayısı 0'dan büyük olmalıdır!")
            .LessThanOrEqualTo(200).WithMessage("Toplam kat sayısı en fazla 200 olabilir!")
            .When(x => x.TotalFloors.HasValue);

        // Yapım Yılı Validasyonu
        RuleFor(x => x.YearBuilt)
            .NotEmpty().WithMessage("Yapım yılı zorunludur!")
            .GreaterThanOrEqualTo(1900).WithMessage("Yapım yılı en az 1900 olabilir!")
            .LessThanOrEqualTo(2100).WithMessage("Yapım yılı en fazla 2100 olabilir!")
            .Must(year => year <= DateTime.Now.Year)
            .WithMessage("Yapım yılı gelecek bir tarih olamaz!");

        // Emlak Tipi Validasyonu
        RuleFor(x => x.PropertyTypeId)
            .NotEmpty().WithMessage("Emlak tipi zorunludur!")
            .GreaterThan(0).WithMessage("Geçerli bir emlak tipi seçilmelidir!");

        // Durum Validasyonu
        RuleFor(x => x.Status)
            .IsInEnum().WithMessage("Geçerli bir durum seçilmelidir!");
    }
}
```

#### Detaylı Açıklamalar:

**1. Title (Başlık) Validasyonu:**
```csharp
RuleFor(x => x.Title)
    .NotEmpty()  // Boş olamaz
    .MinimumLength(3)  // En az 3 karakter
    .MaximumLength(200)  // En fazla 200 karakter
    .Matches(@"^[a-zA-Z0-9ğüşıöçĞÜŞİÖÇ\s\-,.]+$")  // Regex pattern
```

- **NotEmpty():** String'in null veya boş olmamasını sağlar
- **MinimumLength(3):** En az 3 karakter olmalıdır (çok kısa başlıklar anlamsız)
- **MaximumLength(200):** En fazla 200 karakter (veritabanı ve UI limitleri için)
- **Matches():** Regex pattern ile sadece izin verilen karakterler (harf, rakam, Türkçe karakterler, boşluk, tire, nokta, virgül)

**2. Price (Fiyat) Validasyonu:**
```csharp
RuleFor(x => x.Price)
    .GreaterThan(0)  // 0'dan büyük
    .LessThanOrEqualTo(999999999)  // Maksimum limit
```

- **GreaterThan(0):** Fiyat 0 veya negatif olamaz (iş kuralı)
- **LessThanOrEqualTo(999999999):** Çok yüksek değerlerin girişini engeller (veri bütünlüğü)

**3. YearBuilt (Yapım Yılı) Validasyonu:**
```csharp
RuleFor(x => x.YearBuilt)
    .GreaterThanOrEqualTo(1900)  // Mantıklı minimum
    .LessThanOrEqualTo(2100)  // Mantıklı maksimum
    .Must(year => year <= DateTime.Now.Year)  // Custom rule
```

- **GreaterThanOrEqualTo(1900):** Çok eski binalar için mantıklı limit
- **LessThanOrEqualTo(2100):** Gelecek tarihleri engeller
- **Must():** Custom validation rule - yapım yılı bugünden ileri olamaz

**4. When() Kullanımı (Koşullu Validasyon):**
```csharp
RuleFor(x => x.District)
    .MaximumLength(100)
    .When(x => !string.IsNullOrEmpty(x.District))  // Sadece dolu ise kontrol et
```

- **When():** Opsiyonel alanlar için - sadece değer varsa validation yap

### 2. PropertyUpdateDtoValidator

**PropertyUpdateDtoValidator**, mevcut bir ilanı güncellerken kullanılır. PropertyCreateDtoValidator ile aynı kurallar + Id kontrolü:

```csharp
public class PropertyUpdateDtoValidator : AbstractValidator<PropertyUpdateDto>
{
    public PropertyUpdateDtoValidator()
    {
        // Id zorunlu ve 0'dan büyük olmalı
        RuleFor(x => x.Id)
            .NotEmpty().WithMessage("İlan kimliği zorunludur!")
            .GreaterThan(0).WithMessage("Geçerli bir ilan kimliği olmalıdır!");

        // PropertyCreateDto ile aynı kurallar (kod tekrarını önlemek için base class kullanılabilir)
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("İlan başlığı zorunludur!")
            .MinimumLength(3).WithMessage("İlan başlığı en az 3 karakter olmalıdır!")
            // ... diğer kurallar
    }
}
```

### 3. InquiryCreateDtoValidator

**InquiryCreateDtoValidator**, müşterilerin ilanlar hakkında sorgu gönderirken kullanılır:

```csharp
public class InquiryCreateDtoValidator : AbstractValidator<InquiryCreateDto>
{
    public InquiryCreateDtoValidator()
    {
        // İsim Validasyonu
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ad Soyad zorunludur!")
            .MinimumLength(2).WithMessage("Ad Soyad en az 2 karakter olmalıdır!")
            .MaximumLength(100).WithMessage("Ad Soyad en fazla 100 karakter olabilir!")
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]+$")
            .WithMessage("Ad Soyad sadece harf ve boşluk içerebilir!");

        // E-posta Validasyonu
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("E-posta adresi zorunludur!")
            .EmailAddress().WithMessage("Geçerli bir e-posta adresi giriniz!")
            .MaximumLength(100).WithMessage("E-posta adresi en fazla 100 karakter olabilir!");

        // Telefon Validasyonu (Opsiyonel)
        RuleFor(x => x.Phone)
            .Matches(@"^[0-9]{10,15}$")
            .WithMessage("Telefon numarası 10-15 haneli rakam olmalıdır!")
            .When(x => !string.IsNullOrEmpty(x.Phone));

        // Mesaj Validasyonu
        RuleFor(x => x.Message)
            .NotEmpty().WithMessage("Mesaj zorunludur!")
            .MinimumLength(10).WithMessage("Mesaj en az 10 karakter olmalıdır!")
            .MaximumLength(1000).WithMessage("Mesaj en fazla 1000 karakter olabilir!");

        // İlan ID Validasyonu
        RuleFor(x => x.PropertyId)
            .NotEmpty().WithMessage("İlan kimliği zorunludur!")
            .GreaterThan(0).WithMessage("Geçerli bir ilan kimliği olmalıdır!");
    }
}
```

**Önemli Noktalar:**
- **EmailAddress():** FluentValidation'un built-in e-posta validation'ı
- **Regex Pattern (@"[0-9]{10,15}$"):** Telefon numarası için sadece rakam, 10-15 hane
- **When():** Telefon opsiyonel olduğu için sadece dolu ise kontrol et

### 4. RegisterDtoValidator

**RegisterDtoValidator**, kullanıcı kaydı için kullanılır:

```csharp
public class RegisterDtoValidator : AbstractValidator<RegisterDto>
{
    public RegisterDtoValidator()
    {
        // Ad Validasyonu
        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("Ad zorunludur!")
            .MinimumLength(2).WithMessage("Ad en az 2 karakter olmalıdır!")
            .MaximumLength(50).WithMessage("Ad en fazla 50 karakter olabilir!")
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]+$")
            .WithMessage("Ad sadece harf ve boşluk içerebilir!");

        // Soyad Validasyonu
        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Soyad zorunludur!")
            .MinimumLength(2).WithMessage("Soyad en az 2 karakter olmalıdır!")
            .MaximumLength(50).WithMessage("Soyad en fazla 50 karakter olabilir!")
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]+$")
            .WithMessage("Soyad sadece harf ve boşluk içerebilir!");

        // E-posta Validasyonu
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("E-posta zorunludur!")
            .EmailAddress().WithMessage("Geçerli bir e-posta adresi giriniz!")
            .MaximumLength(100).WithMessage("E-posta en fazla 100 karakter olabilir!");

        // Şifre Validasyonu (Güvenlik için karmaşık)
        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Şifre zorunludur!")
            .MinimumLength(8).WithMessage("Şifre en az 8 karakter olmalıdır!")
            .MaximumLength(50).WithMessage("Şifre en fazla 50 karakter olabilir!")
            .Matches(@"[A-Z]").WithMessage("Şifre en az bir büyük harf içermelidir!")
            .Matches(@"[a-z]").WithMessage("Şifre en az bir küçük harf içermelidir!")
            .Matches(@"[0-9]").WithMessage("Şifre en az bir rakam içermelidir!")
            .Matches(@"[!@#$%^&*(),.?"":{}|<>]")
            .WithMessage("Şifre en az bir özel karakter içermelidir!");

        // Şifre Tekrar Validasyonu
        RuleFor(x => x.ConfirmPassword)
            .NotEmpty().WithMessage("Şifre tekrarı zorunludur!")
            .Equal(x => x.Password).WithMessage("Şifreler eşleşmiyor!");
    }
}
```

**Şifre Güvenlik Kuralları:**
- **MinimumLength(8):** En az 8 karakter (güvenlik standardı)
- **Matches(@"[A-Z]"):** En az bir büyük harf
- **Matches(@"[a-z]"):** En az bir küçük harf
- **Matches(@"[0-9]"):** En az bir rakam
- **Matches(@"[!@#$%^&*(),.?"":{}|<>]"):** En az bir özel karakter
- **Equal(x => x.Password):** Şifre tekrarı ile şifrenin eşleşmesi

### 5. LoginDtoValidator

**LoginDtoValidator**, kullanıcı girişi için kullanılır:

```csharp
public class LoginDtoValidator : AbstractValidator<LoginDto>
{
    public LoginDtoValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("E-posta zorunludur!")
            .EmailAddress().WithMessage("Geçerli bir e-posta adresi giriniz!");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Şifre zorunludur!");
    }
}
```

**Not:** Login'de şifre format kontrolü yapılmaz (güvenlik nedeniyle - saldırgana ipucu vermemek için).

---

## ⚙️ Program.cs Yapılandırması

FluentValidation'ı ASP.NET Core'a entegre etmek için `Program.cs`'de yapılandırma yapılır:

```csharp
using FluentValidation;
using FluentValidation.AspNetCore;
using ECommerce.Business.Validators;

var builder = WebApplication.CreateBuilder(args);

// Controllers ekleme
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>();  // ValidationFilter eklenir
});

// FluentValidation yapılandırması
builder.Services.AddFluentValidationAutoValidation();  // Otomatik validation
builder.Services.AddFluentValidationClientsideAdapters();  // Client-side adapters
builder.Services.AddValidatorsFromAssemblyContaining<PropertyCreateDtoValidator>();  // Validator'ları bul

var app = builder.Build();

// Middleware pipeline...
```

### Detaylı Açıklamalar:

**1. AddFluentValidationAutoValidation():**
```csharp
builder.Services.AddFluentValidationAutoValidation();
```
- **Amaç:** FluentValidation'ın otomatik olarak çalışmasını sağlar
- **Ne Yapar:** Her HTTP request'te ilgili DTO için validator'ı otomatik bulur ve çalıştırır
- **Sonuç:** ModelState'e hataları ekler

**2. AddFluentValidationClientsideAdapters():**
```csharp
builder.Services.AddFluentValidationClientsideAdapters();
```
- **Amaç:** Client-side validation için metadata sağlar
- **Ne Yapar:** Swagger UI'da validation kurallarını gösterir
- **Sonuç:** Frontend developer'lar validation kurallarını görebilir

**3. AddValidatorsFromAssemblyContaining<T>():**
```csharp
builder.Services.AddValidatorsFromAssemblyContaining<PropertyCreateDtoValidator>();
```
- **Amaç:** Tüm validator sınıflarını otomatik bulur ve kaydeder
- **Ne Yapar:** `PropertyCreateDtoValidator`'ın bulunduğu assembly'deki (Business katmanı) tüm `AbstractValidator<T>` türevlerini bulur
- **Sonuç:** Her validator DI container'a eklenir, manuel registration gerekmez

**Neden AssemblyContaining Kullanıyoruz?**
- Tüm validator'ları tek tek eklemek yerine, assembly'deki hepsini otomatik bulur
- Yeni validator eklendiğinde otomatik çalışır
- Kod tekrarını önler

---

## 🔄 ValidationFilter ile Entegrasyon

FluentValidation hatalarını `ResponseDto<T>` formatına çevirmek için `ValidationFilter` kullanılır:

```csharp
using ECommerce.Business.Exceptions;
using Microsoft.AspNetCore.Mvc.Filters;

namespace ECommerce.API.Filters;

public class ValidationFilter : IAsyncResultFilter
{
    public async Task OnResultExecutionAsync(
        ResultExecutingContext context, 
        ResultExecutionDelegate next)
    {
        // ModelState geçerli mi kontrol et
        if (!context.ModelState.IsValid)
        {
            // Hataları dictionary'ye çevir
            var errors = context.ModelState
                .Where(x => x.Value != null && x.Value.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,  // Property adı (örn: "Title")
                    kvp => kvp.Value!.Errors.Select(e => e.ErrorMessage).ToArray()  // Hata mesajları array
                );
            
            // ValidationException fırlat
            throw new ValidationException(errors);
        }
        
        await next();  // Eğer geçerliyse bir sonraki middleware'e geç
    }
}
```

### Detaylı Açıklamalar:

**1. IAsyncResultFilter:**
- **Amaç:** Action çalıştıktan sonra, result dönmeden önce çalışır
- **Neden:** ModelState bu aşamada tamamen doldurulmuş olur (FluentValidation çalışmıştır)

**2. ModelState.IsValid Kontrolü:**
```csharp
if (!context.ModelState.IsValid)
```
- **Amaç:** FluentValidation hataları ModelState'e eklenir, bu kontrol ile hata olup olmadığını anlarız
- **Ne Yapar:** Eğer validation hataları varsa, bunları işler

**3. Errors Dictionary Oluşturma:**
```csharp
var errors = context.ModelState
    .Where(x => x.Value != null && x.Value.Errors.Count > 0)
    .ToDictionary(...)
```
- **Where():** Sadece hata içeren property'leri filtreler
- **ToDictionary():** Key-Value çiftlerine çevirir
  - **Key:** Property adı (örn: "Title", "Price")
  - **Value:** Hata mesajları array (bir property'de birden fazla hata olabilir)

**4. ValidationException Fırlatma:**
```csharp
throw new ValidationException(errors);
```
- **Amaç:** Global exception handler bu hatayı yakalar
- **Sonuç:** `ResponseDto<T>.Fail()` formatında hata response döner

**5. Controller'a Filter Ekleme:**

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>();  // Global olarak eklenir
});
```

**Not:** ASP.NET Core'un varsayılan validation response'unu kapatmak için:

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>();
    options.SuppressModelStateInvalidFilter = true;  // Varsayılan validation response'u kapat
});
```

---

## 🧪 Test Senaryoları

### Senaryo 1: Geçerli Property Oluşturma

**Request:**
```json
POST /api/properties
{
  "title": "Merkezi Konumda 3+1 Daire",
  "description": "Şehir merkezinde, okullara yakın, geniş balkonlu daire",
  "price": 2500000,
  "address": "Atatürk Mahallesi, İstiklal Caddesi No:123",
  "city": "İstanbul",
  "district": "Kadıköy",
  "rooms": 3,
  "bathrooms": 2,
  "area": 120,
  "floor": 5,
  "totalFloors": 10,
  "yearBuilt": 2015,
  "propertyTypeId": 1,
  "status": 0
}
```

**Beklenen Sonuç:**
- ✅ Status Code: 200 OK
- ✅ Response: Başarılı ilan oluşturma mesajı

### Senaryo 2: Geçersiz Property (Başlık Çok Kısa)

**Request:**
```json
POST /api/properties
{
  "title": "AB",  // Çok kısa!
  "description": "Açıklama",
  "price": 2500000,
  // ... diğer alanlar
}
```

**Beklenen Sonuç:**
- ❌ Status Code: 400 Bad Request
- ❌ Response:
```json
{
  "success": false,
  "message": "Validation hatası",
  "data": null,
  "errors": {
    "Title": ["İlan başlığı en az 3 karakter olmalıdır!"]
  }
}
```

### Senaryo 3: Geçersiz Property (Fiyat Negatif)

**Request:**
```json
POST /api/properties
{
  "title": "Merkezi Konumda 3+1 Daire",
  "price": -1000,  // Negatif!
  // ... diğer alanlar
}
```

**Beklenen Sonuç:**
- ❌ Status Code: 400 Bad Request
- ❌ Response:
```json
{
  "success": false,
  "message": "Validation hatası",
  "data": null,
  "errors": {
    "Price": ["Fiyat 0'dan büyük olmalıdır!"]
  }
}
```

### Senaryo 4: Çoklu Hata

**Request:**
```json
POST /api/properties
{
  "title": "AB",  // Çok kısa
  "price": -1000,  // Negatif
  "rooms": 0,  // Geçersiz
  "yearBuilt": 3000  // Gelecek tarih
}
```

**Beklenen Sonuç:**
- ❌ Status Code: 400 Bad Request
- ❌ Response: Tüm hatalar birlikte döner:
```json
{
  "success": false,
  "message": "Validation hatası",
  "data": null,
  "errors": {
    "Title": ["İlan başlığı en az 3 karakter olmalıdır!"],
    "Price": ["Fiyat 0'dan büyük olmalıdır!"],
    "Rooms": ["Oda sayısı 0'dan büyük olmalıdır!"],
    "YearBuilt": ["Yapım yılı gelecek bir tarih olamaz!"]
  }
}
```

### Senaryo 5: Geçerli Inquiry Gönderme

**Request:**
```json
POST /api/inquiries
{
  "name": "Ahmet Yılmaz",
  "email": "ahmet@example.com",
  "phone": "05321234567",
  "message": "Bu ilan hakkında bilgi almak istiyorum",
  "propertyId": 1
}
```

**Beklenen Sonuç:**
- ✅ Status Code: 200 OK

### Senaryo 6: Geçersiz E-posta

**Request:**
```json
POST /api/inquiries
{
  "name": "Ahmet Yılmaz",
  "email": "gecersiz-email",  // Geçersiz format
  "message": "Mesaj",
  "propertyId": 1
}
```

**Beklenen Sonuç:**
- ❌ Status Code: 400 Bad Request
- ❌ Response:
```json
{
  "success": false,
  "errors": {
    "Email": ["Geçerli bir e-posta adresi giriniz!"]
  }
}
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Validator Sınıfları Business Katmanında:**
   - Validator'lar `ECommerce.Business/Validators/` klasöründe olmalı
   - DTO'lar ile aynı katmanda (separation of concerns)

2. **Naming Convention:**
   - Validator sınıf adı: `{DtoName}Validator`
   - Örnek: `PropertyCreateDto` → `PropertyCreateDtoValidator`

3. **Hata Mesajları Türkçe:**
   - `WithMessage()` ile Türkçe hata mesajları kullanın
   - Kullanıcı dostu mesajlar yazın

4. **When() Kullanımı:**
   - Opsiyonel alanlar için `When()` kullanın
   - Sadece değer varsa validation yapın

5. **Custom Validation (Must()):**
   - Karmaşık kurallar için `Must()` kullanın
   - Örnek: Yapım yılı bugünden ileri olamaz

6. **Regex Patterns:**
   - Türkçe karakterler için: `[a-zA-Z0-9ğüşıöçĞÜŞİÖÇ]`
   - Telefon için: `^[0-9]{10,15}$`
   - E-posta için: FluentValidation'un `EmailAddress()` kullanın

### Sık Yapılan Hatalar

1. **Validator'ı DI'ye Kaydetmemek:**
   - `AddValidatorsFromAssemblyContaining<T>()` kullanmayı unutmayın

2. **ValidationFilter'ı Eklememek:**
   - `AddControllers()` içinde `options.Filters.Add<ValidationFilter>()` ekleyin

3. **ModelState Suppress Etmemek:**
   - `SuppressModelStateInvalidFilter = true` ekleyin (varsayılan response'u kapatmak için)

4. **Hata Mesajlarını Unutmak:**
   - Her kural için `WithMessage()` ekleyin
   - Varsayılan İngilizce mesajlar kullanıcı dostu değildir

5. **Regex Pattern Hataları:**
   - Türkçe karakterleri unutmayın
   - Escape karakterlerini doğru kullanın (`\.`, `\-`)

### İpuçları

1. **Validator'ları Test Edin:**
   - Her validator için unit test yazın
   - Edge case'leri test edin (null, empty, boundary values)

2. **Hata Mesajlarını Düşünün:**
   - Kullanıcının neyi yanlış yaptığını anlamasını sağlayın
   - Teknik terimlerden kaçının

3. **Validation Sırası:**
   - FluentValidation kuralları sırayla çalışır
   - İlk hata bulunduğunda durmaz, tüm hataları toplar

4. **Performance:**
   - Validator'lar DI ile singleton olarak çalışır (performanslı)
   - Her request'te yeni instance oluşturulmaz

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **FluentValidation Nedir:** Ayrı validator sınıfları ile validation yapma
2. ✅ **Kurulum:** NuGet paketleri ve Program.cs yapılandırması
3. ✅ **Validator Yazma:** Property, Inquiry, Register, Login için validator'lar
4. ✅ **Entegrasyon:** ValidationFilter ile ResponseDto formatına çevirme
5. ✅ **Test:** Geçerli ve geçersiz senaryolar

**Sonraki Adım:** CORS ve API Güvenliği dersine geçebiliriz.

---

**Başarılar! 🚀**

