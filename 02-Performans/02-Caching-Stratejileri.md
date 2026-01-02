# Caching Stratejileri
## ECommerce API - Performans Dersleri

**Seviye:** Orta  
**Hedef:** Sık kullanılan verileri cache'leme

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Caching Nedir?](#caching-nedir)
3. [Neden Caching?](#neden-caching)
4. [In-Memory Caching](#in-memory-caching)
5. [Cache-Aside Pattern](#cache-aside-pattern)
6. [Cache Key Stratejileri](#cache-key-stratejileri)
7. [Cache Invalidation](#cache-invalidation)
8. [Response Caching](#response-caching)
9. [Test Senaryoları](#test-senaryoları)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Caching Nedir?

**Caching**, sık kullanılan verileri hızlı erişim için geçici olarak saklama işlemidir.

**Örnek:**
- Kategori listesi → Cache'te sakla (5 dakika)
- Tekrar istek gelirse → Cache'ten dön (hızlı!)
- 5 dakika sonra → Cache expire, yeniden DB'den çek

### ECommerce API'de Neden Caching?

1. **Performans:**
   - DB sorgusu: 100ms
   - Cache'ten okuma: 1ms
   - **100x daha hızlı!**

2. **Database Yükünü Azaltma:**
   - Sık kullanılan veriler cache'te
   - DB'ye daha az istek

3. **Maliyet:**
   - DB işlemleri pahalı
   - Cache işlemleri ucuz

---

## 📚 Caching Nedir? (Detaylı)

### Cache-Aside Pattern

**Çalışma Mantığı:**
```
1. İstek Gelir
2. Cache'te Var mı? → Evet → Cache'ten Dön (Hızlı!)
3. Cache'te Yok → DB'den Çek → Cache'e Kaydet → Dön
4. Cache Expire → Tekrar DB'den Çek
```

**Avantajlar:**
- ✅ Basit implementasyon
- ✅ Cache miss durumunda DB'den çeker
- ✅ Cache invalidation kolay

---

## 🛠️ In-Memory Caching

### Program.cs Yapılandırması

```csharp
var builder = WebApplication.CreateBuilder(args);

// In-Memory Cache ekle
builder.Services.AddMemoryCache();

var app = builder.Build();
```

### CategoryService'de Cache Kullanımı

```csharp
using Microsoft.Extensions.Caching.Memory;

public class CategoryService : ICategoryService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;
    private readonly IRepository<Category> _categoryRepository;
    private readonly IMemoryCache _memoryCache;
    private const int CacheExpirationMinutes = 5;

    public CategoryService(
        IUnitOfWork unitOfWork, 
        IMapper mapper,
        IMemoryCache memoryCache)
    {
        _unitOfWork = unitOfWork;
        _mapper = mapper;
        _categoryRepository = _unitOfWork.GetRepository<Category>();
        _memoryCache = memoryCache;
    }

    public async Task<ResponseDto<IEnumerable<CategoryDto>>> GetAllAsync()
    {
        // Cache key
        var cacheKey = "all_categories";

        // Cache'te var mı kontrol et
        if (_memoryCache.TryGetValue(cacheKey, out IEnumerable<CategoryDto>? cachedCategories))
        {
            return ResponseDto<IEnumerable<CategoryDto>>.Success(cachedCategories!, StatusCodes.Status200OK);
        }

        // Cache'te yok, DB'den çek
        var categories = await _categoryRepository.GetAllAsync();
        var categoryDtos = _mapper.Map<IEnumerable<CategoryDto>>(categories);

        // Cache'e kaydet
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(CacheExpirationMinutes),
            Priority = CacheItemPriority.Normal
        };
        _memoryCache.Set(cacheKey, categoryDtos, cacheOptions);

        return ResponseDto<IEnumerable<CategoryDto>>.Success(categoryDtos, StatusCodes.Status200OK);
    }
}
```

**Açıklamalar:**
- **TryGetValue():** Cache'te var mı kontrol eder
- **Set():** Cache'e kaydeder
- **AbsoluteExpirationRelativeToNow:** Cache expire süresi (5 dakika)
- **Priority:** Cache önceliği (bellek dolduğunda hangi cache silinecek)

---

## 🔑 Cache Key Stratejileri

### Cache Key Extension

```csharp
public static class CacheKeyExtensions
{
    public static string AllCategories() => "all_categories";
    public static string CategoryById(int id) => $"category_{id}";
    public static string ProductsByCategory(int categoryId) => $"products_category_{categoryId}";
}
```

**Kullanım:**
```csharp
var cacheKey = CacheKeyExtensions.AllCategories();
```

**Avantajlar:**
- ✅ Tutarlı cache key'ler
- ✅ Typo hatalarını önler
- ✅ Kolay refactoring

---

## 🗑️ Cache Invalidation

### Update/Delete İşlemlerinde Cache Temizleme

```csharp
public async Task<ResponseDto<CategoryDto>> UpdateAsync(int id, CategoryUpdateDto categoryUpdateDto)
{
    var category = await _categoryRepository.GetAsync(id);
    if (category == null)
    {
        throw new NotFoundException("Category", id);
    }

    // Update işlemi
    _mapper.Map(categoryUpdateDto, category);
    _categoryRepository.Update(category);
    await _unitOfWork.SaveAsync();

    // Cache'i temizle
    _memoryCache.Remove(CacheKeyExtensions.AllCategories());
    _memoryCache.Remove(CacheKeyExtensions.CategoryById(id));

    var categoryDto = _mapper.Map<CategoryDto>(category);
    return ResponseDto<CategoryDto>.Success(categoryDto, StatusCodes.Status200OK);
}
```

**Önemli:** Update/Delete/Create işlemlerinde ilgili cache'leri temizleyin!

---

## 📡 Response Caching

### Program.cs Yapılandırması

```csharp
var builder = WebApplication.CreateBuilder(args);

// Response Caching
builder.Services.AddResponseCaching();

var app = builder.Build();

// Middleware
app.UseResponseCaching();
app.MapControllers();
```

### Controller'da Kullanım

```csharp
[HttpGet("paged")]
[ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "pageNumber", "pageSize", "categoryId" })]
public async Task<IActionResult> GetAllProductsPaged(
    [FromQuery] PaginationQueryDto paginationQueryDto,
    [FromQuery] int? categoryId = null)
{
    // ...
}
```

**Açıklamalar:**
- **Duration:** Cache süresi (saniye)
- **VaryByQueryKeys:** Query parametrelerine göre farklı cache (pageNumber, categoryId vb.)

---

## 🧪 Test Senaryoları

### Senaryo 1: Cache Hit (İlk İstekten Sonra)

```bash
# 1. İstek (DB'den çeker, cache'e kaydeder)
curl http://localhost:5070/api/categories

# 2. İstek (Cache'ten döner, çok hızlı!)
curl http://localhost:5070/api/categories
```

**Beklenen:**
- İlk istek: ~100ms (DB sorgusu)
- İkinci istek: ~1ms (Cache'ten)

### Senaryo 2: Cache Invalidation

```bash
# 1. Kategori listesini çek (cache'e kaydeder)
curl http://localhost:5070/api/categories

# 2. Kategori güncelle (cache temizlenir)
curl -X PUT http://localhost:5070/api/categories/1 -d '{"name":"Yeni Kategori"}'

# 3. Kategori listesini tekrar çek (yeniden DB'den çeker)
curl http://localhost:5070/api/categories
```

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **Cache Expiration:** Mutlaka expire süresi belirleyin
2. **Cache Invalidation:** Update/Delete işlemlerinde temizleyin
3. **Cache Key:** Tutarlı key stratejisi kullanın
4. **Memory Limit:** In-memory cache sınırlıdır

### Sık Yapılan Hatalar

1. **Cache Invalidation Unutmak:** ❌ Eski veri döner
2. **Cache Key Typo:** ❌ Cache çalışmaz
3. **Süresiz Cache:** ❌ Eski veri kalır

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Caching:** Veri cache'leme
2. ✅ **Cache-Aside Pattern:** Cache stratejisi
3. ✅ **In-Memory Cache:** IMemoryCache kullanımı
4. ✅ **Cache Keys:** Tutarlı key stratejisi
5. ✅ **Cache Invalidation:** Cache temizleme
6. ✅ **Response Caching:** HTTP response caching

**Sonraki Adım:** Query Optimization dersine geçebiliriz.

---

**Başarılar! 🚀**
