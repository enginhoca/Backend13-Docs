---
layout: page
title: Secrets Management ve Environment Variables
order: 4
bolum: 1
---

# Secrets Management ve Environment Variables
## ECommerce API - Güvenlik

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Secrets Management Nedir?](#secrets-management-nedir)
3. [Neden Secrets Management?](#neden-secrets-management)
4. [ASP.NET Core Configuration Hierarchy](#aspnet-core-configuration-hierarchy)
5. [Development: User Secrets](#development-user-secrets)
6. [Production: Environment Variables](#production-environment-variables)
7. [appsettings.json Yapılandırması](#appsettingsjson-yapılandırması)
8. [Program.cs Yapılandırması](#programcs-yapılandırması)
9. [Docker ve Docker Compose](#docker-ve-docker-compose)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Secrets Management Nedir?

**Secrets Management (Gizli Veri Yönetimi)**, hassas bilgileri (şifreler, API key'ler, connection string'ler) güvenli bir şekilde saklama ve yönetme işlemidir.

**Hassas Bilgiler:**
- JWT Secret Key
- Database Connection String (şifre içerir)
- API Key'ler
- OAuth Client Secret'ları

### ECommerce API'de Hangi Veriler Hassas?

1. **JWT Secret:**
   - Token imzalama için kullanılır
   - Çalınırsa → Sahte token üretilebilir

2. **Database Connection String:**
   - PostgreSQL şifresi içerir
   - Çalınırsa → Veritabanına yetkisiz erişim

3. **API Key'ler (Gelecekte):**
   - 3. parti servis entegrasyonları için
   - Çalınırsa → Servis kötüye kullanılabilir

---

## 📚 Secrets Management Nedir? (Detaylı)

### Neden appsettings.json'da Tutmamalıyız?

**appsettings.json:**
```json
{
  "JwtConfig": {
    "Secret": "super-secret-key-123456"  // ❌ GİT'E COMMIT EDİLİR!
  },
  "ConnectionStrings": {
    "PostgreSqlConnection": "Host=localhost;Password=admin123"  // ❌ ŞİFRE AÇIK!
  }
}
```

**Sorunlar:**
1. Git'e commit edilir → Herkes görebilir
2. Version control'de kalır → Geçmişte her zaman görülebilir
3. Production'da aynı dosya kullanılır → Güvenlik riski
4. Ekip üyeleri arasında paylaşılır → Gereksiz erişim

**Çözüm:** Hassas bilgileri appsettings.json'dan çıkar, environment variables veya User Secrets kullan!

---

## 🔒 Neden Secrets Management?

### Senaryo 1: Git Repository Güvenliği

**Problem:**
- Secret'lar appsettings.json'da
- Git'e commit edilir
- GitHub/GitLab'da görünür
- Eski commit'lerde hala var

**Çözüm:**
- Secret'ları appsettings.json'dan çıkar
- User Secrets (development) veya Environment Variables (production) kullan
- .gitignore ile User Secrets klasörünü ignore et

### Senaryo 2: Production Güvenliği

**Problem:**
- appsettings.json production sunucusunda
- Dosya sisteminde açık metin olarak duruyor
- Sunucu ele geçirilirse → Tüm secret'lar çalınır

**Çözüm:**
- Environment Variables kullan
- Container/Platform secret management kullan (Render.com, Azure Key Vault)
- Runtime'da environment'tan oku

### Senaryo 3: Ekip Çalışması

**Problem:**
- Herkes aynı secret'ları kullanıyor
- Birisi secret'ı değiştirirse → Diğerleri çalışmaz
- Secret rotation zor

**Çözüm:**
- Her geliştirici kendi User Secrets'ını kullanır
- Production'da central secret management (Key Vault)
- Secret rotation kolaylaşır

---

## 📊 ASP.NET Core Configuration Hierarchy

ASP.NET Core, configuration'ı **öncelik sırasına göre** yükler (yüksek öncelik üzerine yazar):

```
1. appsettings.json
2. appsettings.{Environment}.json  (ör: appsettings.Development.json)
3. User Secrets (sadece Development)
4. Environment Variables
5. Command Line Arguments
```

**Önemli:** Yüksek öncelikli kaynak, düşük öncelikli kaynağın değerlerini **override** eder!

**Örnek:**
```json
// appsettings.json
{
  "JwtConfig": {
    "Secret": "default-secret"  // Düşük öncelik
  }
}

// User Secrets
{
  "JwtConfig": {
    "Secret": "real-secret-from-user-secrets"  // Yüksek öncelik → Bu kullanılır!
  }
}
```

---

## 🛠️ Development: User Secrets

### User Secrets Nedir?

**User Secrets**, development ortamında hassas bilgileri lokal olarak saklamak için kullanılır.

**Özellikler:**
- ✅ Git'e commit edilmez (otomatik ignore edilir)
- ✅ Kullanıcı bazlı (her geliştirici kendi secret'larını kullanır)
- ✅ Sadece Development ortamında aktif
- ✅ appsettings.json'dan daha yüksek öncelik

### User Secrets Nasıl Çalışır?

**Konum (Windows):**
```
%APPDATA%\Microsoft\UserSecrets\{UserSecretsId}\secrets.json
```

**Konum (macOS/Linux):**
```
~/.microsoft/usersecrets/{UserSecretsId}/secrets.json
```

**UserSecretsId:** Her proje için unique bir ID (csproj dosyasında tanımlı)

### User Secrets Kurulumu

**1. User Secrets'ı Projeye Ekle:**

```bash
cd ECommerce.API
dotnet user-secrets init
```

**Çıktı:**
```
Successfully initialized User Secrets ID 'abc123...' for project 'ECommerce.API'.
```

**ECommerce.API/ECommerce.API.csproj:**
```xml
<PropertyGroup>
  <UserSecretsId>abc123-def456-ghi789</UserSecretsId>
</PropertyGroup>
```

**2. Secret Ekleme:**

```bash
# JWT Secret
dotnet user-secrets set "JwtConfig:Secret" "your-secret-key-here-min-32-characters"

# Connection String
dotnet user-secrets set "ConnectionStrings:PostgreSqlConnection" "Host=localhost;Port=5420;Database=ecommerce;Username=admin;Password=admin123"
```

**3. Secret Listeleme:**

```bash
dotnet user-secrets list
```

**Çıktı:**
```
JwtConfig:Secret = your-secret-key-here-min-32-characters
ConnectionStrings:PostgreSqlConnection = Host=localhost;Port=5420;Database=ecommerce;Username=admin;Password=admin123
```

**4. Secret Silme:**

```bash
dotnet user-secrets remove "JwtConfig:Secret"
```

**5. Tüm Secret'ları Temizleme:**

```bash
dotnet user-secrets clear
```

### Program.cs'de User Secrets Kullanımı

```csharp
var builder = WebApplication.CreateBuilder(args);

// Development ortamında User Secrets'ı ekle
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}

// Configuration'dan oku (User Secrets öncelikli)
var jwtConfig = builder.Configuration.GetSection("JwtConfig").Get<JwtConfig>();
var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection");
```

**Açıklama:**
- **AddUserSecrets<Program>():** User Secrets'ı configuration'a ekler
- **IsDevelopment():** Sadece Development ortamında aktif
- **GetSection():** Configuration'dan değeri okur (öncelik sırasına göre)

---

## 🚀 Production: Environment Variables

### Environment Variables Nedir?

**Environment Variables**, production ortamında hassas bilgileri saklamak için kullanılır.

**Özellikler:**
- ✅ Runtime'da okunur (kod değişikliği gerekmez)
- ✅ Platform bazlı (Render.com, Docker, Azure, AWS)
- ✅ Güvenli saklama (platform secret management)
- ✅ Kolay rotation (değiştir → restart)

### Environment Variables Formatı

**ASP.NET Core Naming Convention:**
- Nokta (`.`) yerine **çift alt çizgi (`__`)** kullanılır
- Array elementleri için `__0`, `__1` kullanılır

**Örnekler:**
```
appsettings.json:
  "JwtConfig": {
    "Secret": "value"
  }

Environment Variable:
  JwtConfig__Secret=value

appsettings.json:
  "ConnectionStrings": {
    "PostgreSqlConnection": "value"
  }

Environment Variable:
  ConnectionStrings__PostgreSqlConnection=value

appsettings.json:
  "CorsSettings": {
    "AllowedOrigins": ["http://localhost:3000", "http://localhost:5040"]
  }

Environment Variables:
  CorsSettings__AllowedOrigins__0=http://localhost:3000
  CorsSettings__AllowedOrigins__1=http://localhost:5040
```

### Environment Variables Kullanımı (Platform Bazlı)

**1. Render.com:**
- Dashboard → Service → Environment
- Key-Value çiftleri ekleyin
- Deploy sonrası aktif olur

**2. Docker (docker-compose.yml):**
```yaml
services:
  api:
    environment:
      - JwtConfig__Secret=your-secret-key
      - ConnectionStrings__PostgreSqlConnection=Host=postgres;Port=5432;Database=ecommerce;Username=admin;Password=admin123
```

**3. Linux/macOS (Terminal):**
```bash
export JwtConfig__Secret="your-secret-key"
export ConnectionStrings__PostgreSqlConnection="Host=localhost;Port=5432;Database=ecommerce;Username=admin;Password=admin123"
dotnet run
```

**4. Windows (PowerShell):**
```powershell
$env:JwtConfig__Secret="your-secret-key"
$env:ConnectionStrings__PostgreSqlConnection="Host=localhost;Port=5432;Database=ecommerce;Username=admin;Password=admin123"
dotnet run
```

### Program.cs'de Environment Variables Kullanımı

```csharp
var builder = WebApplication.CreateBuilder(args);

// Environment Variables otomatik olarak yüklenir (ek kod gerekmez!)
// Öncelik sırası: Environment Variables > appsettings.json

var jwtConfig = builder.Configuration.GetSection("JwtConfig").Get<JwtConfig>();
var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection");

// Secret kontrolü (production'da boş olmamalı)
if (string.IsNullOrEmpty(jwtConfig?.Secret))
{
    throw new InvalidOperationException("JwtConfig:Secret bulunamadı! Environment variable'ı kontrol edin.");
}
```

**Açıklama:**
- Environment Variables otomatik olarak yüklenir
- Ek kod gerekmez (WebApplication.CreateBuilder otomatik yükler)
- Öncelik: Environment Variables > appsettings.json

---

## 📝 appsettings.json Yapılandırması

### Development: appsettings.json (Secret'lar Boş)

**ECommerce.API/appsettings.json:**
```json
{
  "JwtConfig": {
    "Secret": "",  // ❌ Boş bırakın! User Secrets'dan gelecek
    "Issuer": "ECommerce_Backend",
    "Audience": "ECommerce_Web",
    "AccessTokenExpiration": 30
  },
  "ConnectionStrings": {
    "PostgreSqlConnection": ""  // ❌ Boş bırakın! User Secrets'dan gelecek
  }
}
```

**Not:** Secret değerlerini boş bırakın! User Secrets veya Environment Variables'dan gelecek.

### Production: appsettings.json (Secret'lar Boş)

Production'da da secret'ları boş bırakın, Environment Variables kullanın:

```json
{
  "JwtConfig": {
    "Secret": "",  // ❌ Boş! Environment Variable'dan gelecek
    "Issuer": "ECommerce_Backend",
    "Audience": "ECommerce_Web",
    "AccessTokenExpiration": 30
  },
  "ConnectionStrings": {
    "PostgreSqlConnection": ""  // ❌ Boş! Environment Variable'dan gelecek
  }
}
```

---

## ⚙️ Program.cs Yapılandırması

### Tam Yapılandırma Örneği

```csharp
using Microsoft.Extensions.Configuration;

var builder = WebApplication.CreateBuilder(args);

// Development: User Secrets ekle
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}

// Configuration'dan oku (öncelik sırasına göre)
var jwtConfig = builder.Configuration.GetSection("JwtConfig").Get<JwtConfig>();
var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection");

// Production'da secret kontrolü
if (!builder.Environment.IsDevelopment())
{
    if (string.IsNullOrEmpty(jwtConfig?.Secret))
    {
        throw new InvalidOperationException(
            "JwtConfig:Secret bulunamadı! Environment variable 'JwtConfig__Secret' ayarlanmalı."
        );
    }

    if (string.IsNullOrEmpty(connectionString))
    {
        throw new InvalidOperationException(
            "ConnectionStrings:PostgreSqlConnection bulunamadı! Environment variable 'ConnectionStrings__PostgreSqlConnection' ayarlanmalı."
        );
    }
}

// JWT Configuration'ı DI'ya kaydet
builder.Services.Configure<JwtConfig>(builder.Configuration.GetSection("JwtConfig"));

// Database Context
builder.Services.AddDbContext<ECommerceDbContext>(options =>
    options.UseNpgsql(connectionString));

var app = builder.Build();
app.Run();
```

**Açıklamalar:**

**1. AddUserSecrets<Program>():**
```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```
- Sadece Development ortamında User Secrets ekler
- Production'da çalışmaz (güvenlik)

**2. GetSection() ve Get<T>():**
```csharp
var jwtConfig = builder.Configuration.GetSection("JwtConfig").Get<JwtConfig>();
```
- Configuration hierarchy'den değeri okur
- Öncelik: Environment Variables > User Secrets > appsettings.json
- Strongly-typed mapping (JwtConfig sınıfına map eder)

**3. GetConnectionString():**
```csharp
var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection");
```
- Connection string'i okur
- "ConnectionStrings:PostgreSqlConnection" path'ini kullanır

**4. Production Secret Kontrolü:**
```csharp
if (string.IsNullOrEmpty(jwtConfig?.Secret))
{
    throw new InvalidOperationException("Secret bulunamadı!");
}
```
- Production'da secret'ların ayarlandığını doğrular
- Eksik secret varsa uygulama başlamaz (fail-fast)

---

## 🐳 Docker ve Docker Compose

### Docker Compose ile Environment Variables

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: ecommerce_db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123  # Production'da environment variable kullanın!
      POSTGRES_DB: ecommerce
    ports:
      - "5420:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d ecommerce"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build: .
    container_name: ecommerce_api
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - JwtConfig__Secret=${JWT_SECRET}  # .env dosyasından oku
      - JwtConfig__Issuer=ECommerce_Backend
      - JwtConfig__Audience=ECommerce_Web
      - JwtConfig__AccessTokenExpiration=30
      - ConnectionStrings__PostgreSqlConnection=Host=postgres;Port=5432;Database=ecommerce;Username=admin;Password=admin123
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
```

### .env Dosyası (Docker Compose için)

**.env:**
```
JWT_SECRET=your-secret-key-here-min-32-characters
```

**docker-compose.yml'de Kullanım:**
```yaml
environment:
  - JwtConfig__Secret=${JWT_SECRET}  # .env dosyasından oku
```

**.gitignore:**
```
.env
.env.local
.env.*.local
```

**Not:** .env dosyasını Git'e commit etmeyin!

### .env.example (Template)

**.env.example:**
```
JWT_SECRET=your-secret-key-here-min-32-characters
POSTGRES_PASSWORD=your-postgres-password
```

**Amaç:** Ekip üyeleri için template (gerçek secret'lar yok)

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Configuration Hierarchy:**
   - Environment Variables > User Secrets > appsettings.json
   - Yüksek öncelik düşük önceliği override eder

2. **Naming Convention:**
   - Environment Variables: `__` (çift alt çizgi)
   - appsettings.json: `.` (nokta)
   - Örnek: `JwtConfig__Secret` (env) = `JwtConfig.Secret` (json)

3. **User Secrets:**
   - Sadece Development
   - Git'e commit edilmez
   - Kullanıcı bazlı

4. **Environment Variables:**
   - Production'da kullanılır
   - Platform bazlı (Render.com, Docker, Azure)
   - Runtime'da okunur

5. **Secret Kontrolü:**
   - Production'da secret'ların ayarlandığını doğrulayın
   - Eksik secret varsa uygulama başlamamalı

### Sık Yapılan Hatalar

1. **appsettings.json'da Secret Tutmak:**
   - ❌ Yanlış: Secret'ları appsettings.json'a yazmak
   - ✅ Doğru: Secret'ları boş bırak, User Secrets/Environment Variables kullan

2. **User Secrets'ı Production'da Kullanmak:**
   - ❌ Yanlış: Production'da AddUserSecrets kullanmak
   - ✅ Doğru: Sadece Development'ta kullan

3. **Environment Variable Formatı:**
   - ❌ Yanlış: `JwtConfig.Secret` (nokta)
   - ✅ Doğru: `JwtConfig__Secret` (çift alt çizgi)

4. **.env Dosyasını Git'e Commit Etmek:**
   - ❌ Yanlış: .env dosyasını Git'e eklemek
   - ✅ Doğru: .env'i .gitignore'a ekle, .env.example kullan

5. **Secret Kontrolü Unutmak:**
   - ❌ Yanlış: Secret'ın null olup olmadığını kontrol etmemek
   - ✅ Doğru: Production'da secret kontrolü yap

### İpuçları

1. **Development Setup:**
   ```bash
   # 1. User Secrets init
   dotnet user-secrets init
   
   # 2. Secret'ları ekle
   dotnet user-secrets set "JwtConfig:Secret" "dev-secret-key"
   dotnet user-secrets set "ConnectionStrings:PostgreSqlConnection" "Host=localhost;Port=5420;..."
   
   # 3. Kontrol et
   dotnet user-secrets list
   ```

2. **Production Setup (Render.com):**
   - Dashboard → Environment Variables
   - `JwtConfig__Secret` = `your-production-secret`
   - `ConnectionStrings__PostgreSqlConnection` = `Host=...;Password=...`

3. **Secret Rotation:**
   - User Secrets: `dotnet user-secrets set` ile güncelle
   - Environment Variables: Platform'dan güncelle → Restart

4. **Debugging:**
   - Configuration değerlerini log'layın (secret'ları değil!)
   - Secret'ın yüklenip yüklenmediğini kontrol edin

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Secrets Management:** Hassas bilgileri güvenli saklama
2. ✅ **Configuration Hierarchy:** Öncelik sırası
3. ✅ **User Secrets:** Development ortamı için
4. ✅ **Environment Variables:** Production ortamı için
5. ✅ **Naming Convention:** `__` vs `.`
6. ✅ **Docker Compose:** .env dosyası kullanımı
7. ✅ **Secret Kontrolü:** Production'da doğrulama

**Sonraki Adım:** Security Headers dersine geçebiliriz.

---

**Başarılar! 🚀**

