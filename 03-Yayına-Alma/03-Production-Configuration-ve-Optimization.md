# Production Configuration ve Optimization
## ECommerce API - Yayına Alma

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Production vs Development](#production-vs-development)
3. [Environment-Based Configuration](#environment-based-configuration)
4. [Logging Configuration](#logging-configuration)
5. [Health Checks](#health-checks)
6. [Performance Optimizations](#performance-optimizations)
7. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Production vs Development

**Development:**
- Detaylı hata mesajları
- Swagger UI aktif
- User Secrets kullanımı
- Debug mode

**Production:**
- Generic hata mesajları (güvenlik)
- Swagger UI kapalı
- Environment Variables kullanımı
- Release mode
- Optimizasyonlar

---

## ⚙️ Environment-Based Configuration

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
    builder.Services.AddSwaggerGen();  // Sadece Development
}

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.Run();
```

---

## 📊 Logging Configuration

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Error"
    }
  }
}
```

**Açıklama:**
- Production'da daha az log (performans)
- Sadece önemli loglar (Warning, Error)

---

## 🏥 Health Checks

### Program.cs

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString);

app.MapHealthChecks("/health");
```

**Kullanım:**
```bash
curl https://your-api.com/health
```

---

## 🚀 Performance Optimizations

### 1. Response Caching
### 2. Compression
### 3. AsNoTracking (Read-only queries)
### 4. Connection Pooling

---

## ✅ Özet

1. ✅ **Environment Check:** Development vs Production
2. ✅ **Logging:** Production'da minimal logging
3. ✅ **Health Checks:** Uygulama durumu kontrolü
4. ✅ **Optimizations:** Performance iyileştirmeleri

**Sonraki Adım:** Database Migration Deployment dersine geçebiliriz.

---

**Başarılar! 🚀**

