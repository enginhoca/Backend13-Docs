# Pagination ve Sayfalama
## ECommerce API - Performans

---

## 📋 İçindekiler

1. [Giriş ve Amaç](#giriş-ve-amaç)
2. [Pagination Nedir?](#pagination-nedir)
3. [Neden Pagination?](#neden-pagination)
4. [Offset-Based Pagination](#offset-based-pagination)
5. [PagedResultDto ve PaginationQueryDto](#pagedresultdto-ve-paginationquerydto)
6. [Repository Katmanında Implementasyon](#repository-katmanında-implementasyon)
7. [Service Katmanında Implementasyon](#service-katmanında-implementasyon)
8. [Controller'da Kullanım](#controllerda-kullanım)
9. [Test Senaryoları](#test-senaryoları)
10. [Öğrenci Notları](#öğrenci-notları)

---

## 🎯 Giriş ve Amaç

### Pagination Nedir?

**Pagination (Sayfalama)**, büyük veri setlerini küçük parçalara bölerek getirme işlemidir.

**Örnek:**
- 10,000 ilan var
- Her sayfada 20 ilan göster
- Toplam 500 sayfa

### ECommerce API'de Neden Pagination?

1. **Performans:**
   - 10,000 ilanı tek seferde getirmek yavaş
   - 20 ilan getirmek hızlı

2. **Bellek Kullanımı:**
   - Tüm veriyi belleğe yüklemek gereksiz
   - Sadece gerekli veriyi getir

3. **Network Bandwidth:**
   - Küçük response = Hızlı transfer
   - Büyük response = Yavaş transfer

4. **Kullanıcı Deneyimi:**
   - Sayfa sayfa gezmek daha iyi
   - Tüm veriyi bir anda göstermek karmaşık

---

## 📚 Pagination Nedir? (Detaylı)

### Offset-Based Pagination

**Çalışma Mantığı:**
```
Sayfa 1: SKIP 0, TAKE 20  → İlan 1-20
Sayfa 2: SKIP 20, TAKE 20 → İlan 21-40
Sayfa 3: SKIP 40, TAKE 20 → İlan 41-60
```

**SQL Sorgusu:**
```sql
SELECT * FROM Properties
ORDER BY CreatedAt DESC
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
```

**Avantajlar:**
- ✅ Basit implementasyon
- ✅ Sayfa numarası ile kolay navigasyon
- ✅ Rastgele sayfa erişimi (örn: 5. sayfaya direkt git)

**Dezavantajlar:**
- ❌ Büyük offset'lerde yavaş (OFFSET 10000 → Yavaş)
- ❌ Yeni veri eklendiğinde sayfa tekrarı olabilir

---

## 🛠️ PagedResultDto ve PaginationQueryDto

### PaginationQueryDto

**ECommerce.Business/DTOs/PaginationQueryDto.cs:**

```csharp
namespace ECommerce.Business.DTOs;

public class PaginationQueryDto
{
    private int _pageNumber = 1;
    private int _pageSize = 10;

    public int PageNumber
    {
        get => _pageNumber;
        set => _pageNumber = value < 1 ? 1 : value;  // Minimum 1
    }

    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = value < 1 ? 10 : (value > 100 ? 100 : value);  // 1-100 arası
    }

    public int Skip => (PageNumber - 1) * PageSize;  // OFFSET değeri
    public int Take => PageSize;  // FETCH NEXT değeri
}
```

**Açıklamalar:**
- **PageNumber:** Sayfa numarası (1'den başlar)
- **PageSize:** Sayfa başına kayıt sayısı (1-100 arası, default 10)
- **Skip:** Atlanacak kayıt sayısı (OFFSET)
- **Take:** Getirilecek kayıt sayısı (FETCH NEXT)

### PagedResultDto

**ECommerce.Business/DTOs/PagedResultDto.cs:**

```csharp
namespace ECommerce.Business.DTOs;

public class PagedResultDto<T>
{
    public IEnumerable<T> Data { get; set; } = [];
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => PageNumber > 1;
    public bool HasNext => PageNumber < TotalPages;

    public PagedResultDto(IEnumerable<T> data, int pageNumber, int pageSize, int totalCount)
    {
        Data = data;
        PageNumber = pageNumber;
        PageSize = pageSize;
        TotalCount = totalCount;
    }

    public static PagedResultDto<T> Create(IEnumerable<T> data, int pageNumber, int pageSize, int totalCount)
    {
        return new PagedResultDto<T>(data, pageNumber, pageSize, totalCount);
    }
}
```

**Açıklamalar:**
- **Data:** Sayfadaki veriler
- **PageNumber:** Mevcut sayfa numarası
- **PageSize:** Sayfa başına kayıt sayısı
- **TotalCount:** Toplam kayıt sayısı
- **TotalPages:** Toplam sayfa sayısı (hesaplanmış)
- **HasPrevious:** Önceki sayfa var mı?
- **HasNext:** Sonraki sayfa var mı?

---

## 📝 Repository Katmanında Implementasyon

**ECommerce.Data/Abstract/IRepository.cs:**

```csharp
Task<(IEnumerable<T> Data, int TotalCount)> GetPagedAsync(
    Expression<Func<T, bool>>? predicate = null,
    Func<IQueryable<T>, IOrderedQueryable<T>>? orderBy = null,
    int skip = 0,
    int take = 10,
    bool showIsDeleted = false,
    bool asExpanded = false,
    params Func<IQueryable<T>, IQueryable<T>>[] includes
);
```

**ECommerce.Data/Concrete/Repository.cs:**

```csharp
public async Task<(IEnumerable<T> Data, int TotalCount)> GetPagedAsync(
    Expression<Func<T, bool>>? predicate = null,
    Func<IQueryable<T>, IOrderedQueryable<T>>? orderBy = null,
    int skip = 0,
    int take = 10,
    bool showIsDeleted = false,
    bool asExpanded = false,
    params Func<IQueryable<T>, IQueryable<T>>[] includes)
{
    var query = _dbSet.AsQueryable();

    if (!showIsDeleted && typeof(T).GetProperty("IsDeleted") != null)
    {
        query = query.Where(x => !(bool)x.GetType().GetProperty("IsDeleted")!.GetValue(x)!);
    }

    if (predicate != null)
    {
        query = query.Where(predicate);
    }

    // Total Count (filtreleme sonrası)
    var totalCount = await query.CountAsync();

    // Order By
    if (orderBy != null)
    {
        query = orderBy(query);
    }

    // Includes
    if (includes != null)
    {
        query = includes.Aggregate(query, (current, include) => include(current));
    }

    // Pagination (Skip & Take)
    var data = await query.Skip(skip).Take(take).ToListAsync();

    return (data, totalCount);
}
```

**Önemli:** CountAsync() mutlaka Skip/Take'den **ÖNCE** çağrılmalı!

---

## 🎮 Service Katmanında Implementasyon

**ECommerce.Business/Concrete/ProductService.cs:**

```csharp
public async Task<ResponseDto<PagedResultDto<ProductDto>>> GetAllPagedAsync(
    PaginationQueryDto paginationQueryDto,
    Expression<Func<Product, bool>>? predicate = null,
    Func<IQueryable<Product>, IOrderedQueryable<Product>>? orderBy = null,
    bool? includeCategories = false,
    int? categoryId = null,
    bool? isDeleted = null)
{
    // Predicate oluştur
    if (predicate == null)
    {
        predicate = PredicateBuilder.New<Product>(true);
    }

    if (isDeleted.HasValue)
    {
        predicate = predicate.And(x => x.IsDeleted == isDeleted);
    }

    if (categoryId.HasValue)
    {
        predicate = predicate.And(x => x.ProductCategories.Any(pc => pc.CategoryId == categoryId.Value));
    }

    // Order By (default: CreatedAt DESC)
    if (orderBy == null)
    {
        orderBy = x => x.OrderByDescending(y => y.CreatedAt);
    }

    // Includes
    var includeList = new List<Func<IQueryable<Product>, IQueryable<Product>>>();
    if (includeCategories.HasValue && includeCategories.Value)
    {
        includeList.Add(q => q.Include(x => x.ProductCategories).ThenInclude(y => y.Category));
    }

    // Repository'den paginated data getir
    var (products, totalCount) = await _productRepository.GetPagedAsync(
        predicate: predicate,
        orderBy: orderBy,
        skip: paginationQueryDto.Skip,
        take: paginationQueryDto.Take,
        showIsDeleted: isDeleted ?? false,
        asExpanded: true,
        includes: includeList.ToArray()
    );

    // DTO'ya map et
    var productDtos = _mapper.Map<IEnumerable<ProductDto>>(products);

    // PagedResultDto oluştur
    var pagedResultDto = PagedResultDto<ProductDto>.Create(
        productDtos,
        paginationQueryDto.PageNumber,
        paginationQueryDto.PageSize,
        totalCount
    );

    return ResponseDto<PagedResultDto<ProductDto>>.Success(pagedResultDto, StatusCodes.Status200OK);
}
```

---

## 🎯 Controller'da Kullanım

**ECommerce.API/Controllers/ProductsController.cs:**

```csharp
[HttpGet("paged")]
[ResponseCache(Duration = 60)]  // 60 saniye cache
public async Task<IActionResult> GetAllProductsPaged(
    [FromQuery] PaginationQueryDto paginationQueryDto,
    [FromQuery] int? categoryId = null)
{
    var response = await _productService.GetAllPagedAsync(
        paginationQueryDto: paginationQueryDto,
        orderBy: null,
        includeCategories: true,
        categoryId: categoryId
    );
    return CreateActionResult(response);
}
```

**API Kullanımı:**
```
GET /api/products/paged?pageNumber=1&pageSize=20&categoryId=5
```

**Response:**
```json
{
  "success": true,
  "data": {
    "data": [...],
    "pageNumber": 1,
    "pageSize": 20,
    "totalCount": 150,
    "totalPages": 8,
    "hasPrevious": false,
    "hasNext": true
  }
}
```

---

## 🧪 Test Senaryoları

### Senaryo 1: İlk Sayfa

```bash
curl "http://localhost:5070/api/products/paged?pageNumber=1&pageSize=20"
```

**Beklenen:**
- 20 ürün döner
- `hasPrevious: false`
- `hasNext: true` (toplam > 20 ise)

### Senaryo 2: Son Sayfa

```bash
curl "http://localhost:5070/api/products/paged?pageNumber=8&pageSize=20"
```

**Beklenen:**
- Son sayfadaki ürünler döner
- `hasPrevious: true`
- `hasNext: false`

### Senaryo 3: Geçersiz PageNumber

```bash
curl "http://localhost:5070/api/products/paged?pageNumber=0&pageSize=20"
```

**Beklenen:**
- `pageNumber` otomatik 1'e ayarlanır (validation)

---

## 📝 Öğrenci Notları

### Önemli Noktalar

1. **CountAsync() Sırası:** Skip/Take'den önce çağrılmalı
2. **PageNumber Validation:** Minimum 1
3. **PageSize Limiti:** 1-100 arası (performans için)
4. **TotalCount:** Filtreleme sonrası sayılmalı

### Sık Yapılan Hatalar

1. **CountAsync() Skip/Take'den Sonra:** ❌ Yanlış totalCount
2. **PageSize Limit Yok:** ❌ Çok büyük sayfa boyutları
3. **TotalCount Filtrelemeden Önce:** ❌ Yanlış sayfa sayısı

---

## ✅ Özet

Bu derste öğrendiklerimiz:

1. ✅ **Pagination:** Büyük veri setlerini sayfalama
2. ✅ **Offset-Based:** SKIP/TAKE kullanımı
3. ✅ **DTO'lar:** PaginationQueryDto, PagedResultDto
4. ✅ **Repository:** GetPagedAsync implementasyonu
5. ✅ **Service:** Pagination logic
6. ✅ **Controller:** API endpoint

**Sonraki Adım:** Caching Stratejileri dersine geçebiliriz.

---

**Başarılar! 🚀**
