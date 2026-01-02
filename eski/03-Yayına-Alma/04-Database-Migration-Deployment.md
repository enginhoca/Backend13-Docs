# ---
layout: page
title: Database Migration Deployment
order: 4
bolum: 3
---

# Database Migration Deployment
## ECommerce API - Yayına Alma

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Migration Stratejileri](#migration-stratejileri)
3. [Startup Hook Yöntemi](#startup-hook-yöntemi)
4. [Program.cs Implementasyonu](#programcs-implementasyonu)
5. [Test Senaryoları](#test-senaryoları)
6. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Migration Deployment Nedir?

**Migration Deployment**, production ortamında database migration'larını çalıştırma işlemidir.

**Stratejiler:**
1. **Startup Hook:** Uygulama başlarken otomatik çalıştır
2. **Build Command:** Render.com build sırasında çalıştır
3. **Manuel:** Manuel olarak çalıştır

**Öneri:** Startup Hook (otomatik, güvenli)

---

## 🛠️ Startup Hook Yöntemi

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Migration'ları uygula (startup hook)
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<ECommerceDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    
    try
    {
        logger.LogInformation("Migration işlemi başlatılıyor...");
        await dbContext.Database.MigrateAsync();
        logger.LogInformation("Migration işlemleri başarıyla tamamlandı!");
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Migration sırasında hata oluştu!");
        throw;  // Uygulama başlamasın
    }
}

app.Run();
```

**Açıklamalar:**
- **CreateScope():** Service scope oluştur
- **MigrateAsync():** Bekleyen migration'ları uygula
- **Try-Catch:** Hata durumunda uygulama başlamasın

---

## 🧪 Test Senaryoları

### Senaryo 1: İlk Deployment

1. Migration dosyaları oluştur
2. Render.com'a deploy et
3. Logları kontrol et

**Beklenen Log:**
```
Migration işlemi başlatılıyor...
Migration işlemleri başarıyla tamamlandı!
```

### Senaryo 2: Yeni Migration Ekleme

1. Yeni migration oluştur
2. Commit ve push
3. Render.com otomatik deploy
4. Migration otomatik uygulanır

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Startup Hook:** Otomatik migration
2. **Error Handling:** Hata durumunda uygulama başlamasın
3. **Logging:** Migration loglarını izle
4. **Idempotent:** Aynı migration tekrar çalıştırılabilir

### Sık Yapılan Hatalar

1. **Manuel Migration:** ❌ Unutulabilir
2. **Hata Handling Yok:** ❌ Uygulama hatalı başlar
3. **Logging Yok:** ❌ Sorun tespiti zor

---

## ✅ Özet

1. ✅ **Startup Hook:** Otomatik migration
2. ✅ **Error Handling:** Güvenli migration
3. ✅ **Logging:** Migration takibi
4. ✅ **Idempotent:** Güvenli re-run

---

**Başarılar! 🚀**

