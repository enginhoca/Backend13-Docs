# ---
layout: page
title: Containerization ve Docker
order: 1
bolum: 3
---

# Containerization ve Docker
## ECommerce API - Yayına Alma

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Docker Nedir?](#docker-nedir)
3. [Dockerfile Oluşturma](#dockerfile-oluşturma)
4. [Multi-Stage Build](#multi-stage-build)
5. [Docker Compose](#docker-compose)
6. [Test Senaryoları](#test-senaryoları)
7. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Docker Nedir?

**Docker**, uygulamaları container'larda çalıştırmak için kullanılan bir platformdur.

**Avantajlar:**
- ✅ "Benim makinede çalışıyordu" sorunu yok
- ✅ Tutarlı ortam (development, test, production)
- ✅ Kolay deployment
- ✅ İzolasyon (uygulamalar birbirini etkilemez)

---

## 📚 Dockerfile Oluşturma

### Multi-Stage Build

**Dockerfile:**

```dockerfile
# 1. STAGE - Build Stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY ECommerce.sln .
COPY ECommerce.API/*.csproj ./ECommerce.API/
COPY ECommerce.Business/*.csproj ./ECommerce.Business/
COPY ECommerce.Data/*.csproj ./ECommerce.Data/
COPY ECommerce.Entity/*.csproj ./ECommerce.Entity/

COPY . .
WORKDIR /src/ECommerce.API
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

# 2. STAGE - Runtime Stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
ENTRYPOINT ["dotnet", "ECommerce.API.dll"]
```

**Açıklamalar:**
- **Stage 1 (build):** SDK ile build (büyük image)
- **Stage 2 (runtime):** Runtime ile çalıştır (küçük image)
- **Multi-stage:** Sadece runtime gereken dosyalar final image'da

---

## 🐳 Docker Compose

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: ecommerce_db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: ecommerce
    ports:
      - "5420:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
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
      - JwtConfig__Secret=${JWT_SECRET}
      - ConnectionStrings__PostgreSqlConnection=Host=postgres;Port=5432;Database=ecommerce;Username=admin;Password=admin123
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
```

**Komutlar:**
```bash
docker-compose up -d      # Başlat
docker-compose down       # Durdur
docker-compose logs -f    # Logları izle
```

---

## 🧪 Test Senaryoları

```bash
# Build
docker build -t ecommerce-api:latest .

# Run
docker run -p 5000:8080 ecommerce-api:latest

# Test
curl http://localhost:5000/api/health
```

---

## ✅ Özet

1. ✅ **Docker:** Containerization platformu
2. ✅ **Dockerfile:** Image tanımı
3. ✅ **Multi-Stage Build:** Optimize image
4. ✅ **Docker Compose:** Multi-container yönetimi

**Sonraki Adım:** Render.com Setup dersine geçebiliriz.

---

**Başarılar! 🚀**

