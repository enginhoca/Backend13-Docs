# ---
layout: page
title: Render.com Setup ve Configuration
order: 2
bolum: 3
---

# Render.com Setup ve Configuration
## ECommerce API - Yayına Alma

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Render.com Nedir?](#rendercom-nedir)
3. [PostgreSQL Database Oluşturma](#postgresql-database-oluşturma)
4. [Web Service Oluşturma](#web-service-oluşturma)
5. [Environment Variables](#environment-variables)
6. [Test Senaryoları](#test-senaryoları)
7. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Render.com Nedir?

**Render.com**, modern web uygulamalarını deploy etmek için kullanılan bir PaaS (Platform as a Service) çözümüdür.

**Özellikler:**
- ✅ Ücretsiz tier (PostgreSQL + Web Service)
- ✅ Otomatik HTTPS
- ✅ GitHub entegrasyonu
- ✅ Docker desteği

---

## 🗄️ PostgreSQL Database Oluşturma

### Adım 1: Database Oluşturma

1. Render.com Dashboard → "New +" → "PostgreSQL"
2. Ayarlar:
   - Name: `ecommerce-db`
   - Database: `ecommerce`
   - User: `ecommerce_user`
   - Region: Frankfurt (EU) veya Oregon (US)
   - Plan: Free
3. "Create Database" → Bekle (birkaç dakika)

### Adım 2: Connection String

**Internal Database URL:**
```
postgres://user:password@host:5432/database
```

**ASP.NET Core Format:**
```
Host=host;Port=5432;Database=database;Username=user;Password=password;SSL Mode=Require;
```

**Not:** Npgsql genellikle her iki formatı da kabul eder.

---

## 🚀 Web Service Oluşturma

### Adım 1: Repository Bağlama

1. Render.com Dashboard → "New +" → "Web Service"
2. GitHub repository'yi seç
3. Ayarlar:
   - Name: `ecommerce-api`
   - Runtime: **Docker** (önemli: .NET/C# doğrudan yok!)
   - Region: Database ile aynı region
   - Plan: Free

### Adım 2: Build Ayarları

- Build Command: (boş, Dockerfile kullanılacak)
- Start Command: (boş, Dockerfile kullanılacak)

**Not:** Dockerfile varsa otomatik kullanılır.

---

## ⚙️ Environment Variables

### Gerekli Environment Variables

```
ASPNETCORE_ENVIRONMENT=Production
ASPNETCORE_URLS=http://+:8080
JwtConfig__Secret=your-secret-key-here
JwtConfig__Issuer=ECommerce_Backend
JwtConfig__Audience=ECommerce_Web
JwtConfig__AccessTokenExpiration=30
ConnectionStrings__PostgreSqlConnection=Host=postgres;Port=5432;Database=ecommerce;Username=user;Password=password
```

### Database Connection String (Internal)

Render.com'da database service'ini seç → "Connections" → **Internal Database URL**'i kullan.

**Not:** Internal URL, aynı network'teki servisler için optimize edilmiştir.

---

## 🧪 Test Senaryoları

### Senaryo 1: Deployment Kontrolü

1. Render.com Dashboard → Service → "Logs"
2. Build loglarını kontrol et
3. Runtime loglarını kontrol et

**Başarılı Build:**
```
✓ Building Docker image...
✓ Pushing image...
✓ Starting service...
```

### Senaryo 2: API Test

```bash
curl https://your-service.onrender.com/api/health
```

**Beklenen:**
```json
{
  "status": "Healthy"
}
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Runtime:** Docker seçilmeli (.NET/C# doğrudan yok)
2. **Connection String:** Internal URL kullan (hızlı)
3. **Environment Variables:** `__` (çift alt çizgi) kullan
4. **Free Tier:** 90 gün sonra ücretli (eğitim için yeterli)

### Sık Yapılan Hatalar

1. **Yanlış Runtime:** ❌ .NET seçmek (yok!)
2. **External URL:** ❌ Internal yerine external URL kullanmak
3. **Environment Variable Format:** ❌ `.` yerine `__` kullanmak

---

## ✅ Özet

1. ✅ **Render.com:** PaaS platformu
2. ✅ **PostgreSQL:** Database service
3. ✅ **Web Service:** Docker ile deployment
4. ✅ **Environment Variables:** Configuration

**Sonraki Adım:** Production Configuration dersine geçebiliriz.

---

**Başarılar! 🚀**

