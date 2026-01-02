# Query Optimization ve N+1 Problem
## ECommerce API - Performans

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [N+1 Problem Nedir?](#n1-problem-nedir)
3. [Eager Loading (Include)](#eager-loading-include)
4. [Batch Loading](#batch-loading)
5. [AsNoTracking](#asnotracking)
6. [Select Projection](#select-projection)
7. [Index Kullanımı](#index-kullanımı)
8. [Test Senaryoları](#test-senaryoları)
9. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### N+1 Problem Nedir?

**N+1 Problem**, bir ana kayıt için N tane ilişkili kayıt çekerken, her biri için ayrı sorgu çalışmasıdır.

**Örnek:**
```
1 sorgu: SELECT * FROM Orders (10 order)
10 sorgu: SELECT * FROM OrderItems WHERE OrderId = 1
10 sorgu: SELECT * FROM OrderItems WHERE OrderId = 2
...
TOPLAM: 1 + 10*10 = 101 sorgu! ❌
```

**Çözüm:**
```
1 sorgu: SELECT * FROM Orders
1 sorgu: SELECT * FROM OrderItems WHERE OrderId IN (1,2,3,...,10)
TOPLAM: 2 sorgu! ✅
```

---

## 📚 Eager Loading (Include)

### Problem Senaryosu

```csharp
// ❌ N+1 Problem
var orders = await _orderRepository.GetAllAsync();
foreach (var order in orders)
{
    // Her order için ayrı sorgu!
    var items = await _orderItemRepository.GetAllAsync(x => x.OrderId == order.Id);
}
```

**SQL:**
```sql
-- 1. Sorgu
SELECT * FROM Orders;

-- 2. Sorgu (Order 1 için)
SELECT * FROM OrderItems WHERE OrderId = 1;

-- 3. Sorgu (Order 2 için)
SELECT * FROM OrderItems WHERE OrderId = 2;
-- ... 10 sorgu daha
```

### Çözüm: Include Kullanımı

```csharp
// ✅ Eager Loading
var orders = await _orderRepository.GetAllAsync(
    includes: new Func<IQueryable<Order>, IQueryable<Order>>[]
    {
        q => q.Include(o => o.OrderItems).ThenInclude(oi => oi.Product)
    }
);
```

**SQL:**
```sql
-- Tek sorgu!
SELECT o.*, oi.*, p.*
FROM Orders o
LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
LEFT JOIN Products p ON oi.ProductId = p.Id;
```

**Açıklamalar:**
- **Include():** İlişkili entity'yi yükle
- **ThenInclude():** İlişkili entity'nin ilişkisini yükle

---

## 🔄 Batch Loading

### Problem Senaryosu (OrderService)

```csharp
// ❌ N+1 Problem
var order = new Order { ... };
foreach (var itemDto in orderDto.Items)
{
    // Her item için ayrı sorgu!
    var product = await _productRepository.GetAsync(itemDto.ProductId);
    var orderItem = new OrderItem { Product = product, ... };
    order.OrderItems.Add(orderItem);
}
```

**SQL:**
```sql
-- Her item için ayrı sorgu
SELECT * FROM Products WHERE Id = 1;
SELECT * FROM Products WHERE Id = 2;
SELECT * FROM Products WHERE Id = 3;
-- ... 10 sorgu
```

### Çözüm: Batch Loading

```csharp
// ✅ Batch Loading
var productIds = orderDto.Items.Select(x => x.ProductId).ToList();
var products = await _productRepository.GetAllAsync(x => productIds.Contains(x.Id));
var productDict = products.ToDictionary(p => p.Id);

foreach (var itemDto in orderDto.Items)
{
    var product = productDict[itemDto.ProductId];  // O(1) lookup
    var orderItem = new OrderItem { Product = product, ... };
    order.OrderItems.Add(orderItem);
}
```

**SQL:**
```sql
-- Tek sorgu!
SELECT * FROM Products WHERE Id IN (1, 2, 3, ..., 10);
```

**Açıklamalar:**
- **Contains():** SQL'de `IN` clause'a çevrilir
- **ToDictionary():** O(1) lookup için dictionary

---

## 🚀 AsNoTracking

### Tracking Overhead

**EF Core**, varsayılan olarak entity'leri track eder (değişiklikleri izler):

```csharp
// ✅ Tracking var (update için gerekli)
var product = await _dbContext.Products.FirstAsync(x => x.Id == 1);
product.Name = "Yeni İsim";
await _dbContext.SaveChangesAsync();  // Değişiklik kaydedilir
```

**Read-Only Sorgular İçin:**
```csharp
// ❌ Gereksiz tracking (read-only sorgu)
var products = await _dbContext.Products.ToListAsync();  // Track edilir (gereksiz!)
```

### Çözüm: AsNoTracking

```csharp
// ✅ AsNoTracking (tracking yok, daha hızlı)
var products = await _dbContext.Products.AsNoTracking().ToListAsync();
```

**Performans:**
- Tracking: ~100ms
- AsNoTracking: ~50ms
- **2x daha hızlı!**

---

## 📊 Select Projection

### Problem: Gereksiz Veri Çekme

```csharp
// ❌ Tüm kolonları çek (gereksiz veri)
var products = await _dbContext.Products.ToListAsync();
var productNames = products.Select(p => p.Name).ToList();
```

**SQL:**
```sql
SELECT Id, Name, Price, Description, ImageUrl, ... FROM Products;
-- Tüm kolonlar çekildi (gereksiz!)
```

### Çözüm: Select Projection

```csharp
// ✅ Sadece ihtiyaç olan kolonları çek
var productNames = await _dbContext.Products
    .Select(p => p.Name)
    .ToListAsync();
```

**SQL:**
```sql
SELECT Name FROM Products;
-- Sadece Name kolonu çekildi!
```

**Avantajlar:**
- ✅ Daha az veri transfer
- ✅ Daha hızlı sorgu
- ✅ Daha az bellek kullanımı

---

## 🔍 Index Kullanımı

### Index Nedir?

**Index**, veritabanında hızlı arama için kullanılan yapılardır.

**Örnek:**
```sql
-- Index olmadan
SELECT * FROM Products WHERE Name = 'Laptop';
-- Full table scan (yavaş!)

-- Index ile
CREATE INDEX IX_Products_Name ON Products(Name);
SELECT * FROM Products WHERE Name = 'Laptop';
-- Index scan (hızlı!)
```

### Entity Framework'te Index

**ECommerce.Data/ECommerceDbContext.cs:**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Product.Name için index
    modelBuilder.Entity<Product>()
        .HasIndex(p => p.Name)
        .HasDatabaseName("IX_Products_Name");

    // Composite index (Name + IsDeleted)
    modelBuilder.Entity<Product>()
        .HasIndex(p => new { p.Name, p.IsDeleted })
        .HasDatabaseName("IX_Products_Name_IsDeleted");

    // Order.AppUserId için index
    modelBuilder.Entity<Order>()
        .HasIndex(o => o.AppUserId)
        .HasDatabaseName("IX_Orders_AppUserId");
}
```

**Migration:**
```bash
dotnet ef migrations add AddIndexes
dotnet ef database update
```

---

## 🧪 Test Senaryoları

### Senaryo 1: N+1 Problem Tespiti

**EF Core Logging:**
```csharp
builder.Services.AddDbContext<ECommerceDbContext>(options =>
    options.UseNpgsql(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information));  // SQL loglarını göster
```

**Beklenen:**
- ❌ 101 sorgu (N+1 problem)
- ✅ 2 sorgu (çözüm sonrası)

### Senaryo 2: AsNoTracking Performans Testi

```csharp
// Tracking ile
var stopwatch = Stopwatch.StartNew();
var products = await _dbContext.Products.ToListAsync();
stopwatch.Stop();
Console.WriteLine($"Tracking: {stopwatch.ElapsedMilliseconds}ms");

// AsNoTracking ile
stopwatch.Restart();
var products2 = await _dbContext.Products.AsNoTracking().ToListAsync();
stopwatch.Stop();
Console.WriteLine($"AsNoTracking: {stopwatch.ElapsedMilliseconds}ms");
```

**Beklenen:**
- Tracking: ~100ms
- AsNoTracking: ~50ms (daha hızlı!)

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **N+1 Problem:** Include veya Batch Loading ile çözülür
2. **AsNoTracking:** Read-only sorgularda kullanılmalı
3. **Select Projection:** Sadece ihtiyaç olan kolonları çek
4. **Index:** Sık kullanılan kolonlarda index kullan

### Sık Yapılan Hatalar

1. **N+1 Problem Fark Etmemek:** ❌ Performans sorunu
2. **Gereksiz Tracking:** ❌ Yavaş sorgular
3. **Gereksiz Veri Çekme:** ❌ Select projection kullanmamak

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **N+1 Problem:** Çoklu sorgu problemi
2. ✅ **Eager Loading:** Include/ThenInclude
3. ✅ **Batch Loading:** Contains() ile toplu sorgu
4. ✅ **AsNoTracking:** Read-only sorgular
5. ✅ **Select Projection:** Sadece gerekli kolonlar
6. ✅ **Index:** Hızlı arama için index

**Sonraki Adım:** Response Compression dersine geçebiliriz.

---

**Başarılar! 🚀**
