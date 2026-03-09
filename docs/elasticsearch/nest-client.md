# Elasticsearch NEST Client

NEST, .NET için resmi Elasticsearch istemcisidir; yanlış kullanımlar arama performansını düşürür ve index yönetimini zorlaştırır.

---

## 1. ElasticClient'ı Her Çağrıda Oluşturmak

❌ **Yanlış Kullanım:** Her istekte yeni ElasticClient oluşturmak.

```csharp
public async Task<List<Product>> SearchAsync(string query)
{
    var settings = new ConnectionSettings(new Uri("http://localhost:9200"))
        .DefaultIndex("products");
    var client = new ElasticClient(settings); // Her çağrıda yeni bağlantı

    var response = await client.SearchAsync<Product>(s => s
        .Query(q => q.Match(m => m.Field(f => f.Name).Query(query))));

    return response.Documents.ToList();
}
```

✅ **İdeal Kullanım:** DI ile singleton olarak kaydedin.

```csharp
builder.Services.AddSingleton<IElasticClient>(sp =>
{
    var settings = new ConnectionSettings(new Uri("http://localhost:9200"))
        .DefaultIndex("products")
        .DefaultMappingFor<Product>(m => m.IndexName("products"))
        .DefaultMappingFor<Order>(m => m.IndexName("orders"))
        .EnableDebugMode()
        .RequestTimeout(TimeSpan.FromSeconds(30));

    return new ElasticClient(settings);
});

public class ProductSearchService
{
    private readonly IElasticClient _client;

    public ProductSearchService(IElasticClient client) => _client = client;

    public async Task<List<Product>> SearchAsync(string query)
    {
        var response = await _client.SearchAsync<Product>(s => s
            .Query(q => q.Match(m => m.Field(f => f.Name).Query(query))));

        return response.Documents.ToList();
    }
}
```

---

## 2. Mapping Tanımlamamak

❌ **Yanlış Kullanım:** Elasticsearch'ün otomatik mapping yapmasına güvenmek.

```csharp
await _client.IndexDocumentAsync(new Product
{
    Name = "Laptop",
    Description = "Güçlü dizüstü bilgisayar",
    Price = 15000
});
// Elasticsearch tüm string'leri text+keyword olarak map'ler, gereksiz alan
```

✅ **İdeal Kullanım:** Explicit mapping ile index oluşturun.

```csharp
public static class ElasticsearchIndexSetup
{
    public static async Task CreateProductIndexAsync(IElasticClient client)
    {
        var existsResponse = await client.Indices.ExistsAsync("products");
        if (existsResponse.Exists) return;

        await client.Indices.CreateAsync("products", c => c
            .Settings(s => s
                .NumberOfShards(1)
                .NumberOfReplicas(1)
                .Analysis(a => a
                    .Analyzers(an => an
                        .Custom("turkish_analyzer", ca => ca
                            .Tokenizer("standard")
                            .Filters("lowercase", "turkish_stop")))))
            .Map<Product>(m => m
                .Properties(p => p
                    .Keyword(k => k.Name(n => n.Id))
                    .Text(t => t.Name(n => n.Name)
                        .Analyzer("turkish_analyzer")
                        .Fields(f => f.Keyword(kw => kw.Name("keyword"))))
                    .Text(t => t.Name(n => n.Description).Analyzer("turkish_analyzer"))
                    .Number(n => n.Name(nn => nn.Price).Type(NumberType.Double))
                    .Date(d => d.Name(n => n.CreatedAt))
                    .Keyword(k => k.Name(n => n.CategoryId)))));
    }
}
```

---

## 3. Arama Sorgusunu Tek Alana Sınırlamak

❌ **Yanlış Kullanım:** Sadece bir alanda arama yapmak.

```csharp
var response = await _client.SearchAsync<Product>(s => s
    .Query(q => q.Match(m => m.Field(f => f.Name).Query(searchTerm))));
// Sadece Name alanında arar, Description'da eşleşme kaçırılır
```

✅ **İdeal Kullanım:** Multi-match ve boosting ile kapsamlı arama yapın.

```csharp
public async Task<SearchResult<Product>> SearchAsync(string searchTerm, int page = 1, int pageSize = 20)
{
    var response = await _client.SearchAsync<Product>(s => s
        .From((page - 1) * pageSize)
        .Size(pageSize)
        .Query(q => q
            .Bool(b => b
                .Should(
                    sh => sh.MultiMatch(mm => mm
                        .Fields(f => f
                            .Field(p => p.Name, boost: 3)
                            .Field(p => p.Description, boost: 1))
                        .Query(searchTerm)
                        .Type(TextQueryType.BestFields)
                        .Fuzziness(Fuzziness.Auto)),
                    sh => sh.Prefix(p => p
                        .Field(f => f.Name)
                        .Value(searchTerm.ToLower())))))
        .Highlight(h => h
            .Fields(f => f
                .Field(p => p.Name)
                .Field(p => p.Description))
            .PreTags("<mark>")
            .PostTags("</mark>")));

    return new SearchResult<Product>
    {
        Items = response.Documents.ToList(),
        TotalCount = response.Total,
        Highlights = response.Hits.ToDictionary(
            h => h.Id, h => h.Highlight)
    };
}
```

---

## 4. Bulk İşlem Kullanmamak

❌ **Yanlış Kullanım:** Tek tek belge index'lemek.

```csharp
foreach (var product in products)
{
    await _client.IndexDocumentAsync(product); // Her belge için ayrı HTTP isteği
}
// 10.000 belge = 10.000 HTTP isteği
```

✅ **İdeal Kullanım:** Bulk API ile toplu işlem yapın.

```csharp
public async Task BulkIndexAsync(IEnumerable<Product> products)
{
    var bulkResponse = await _client.BulkAsync(b => b
        .Index("products")
        .IndexMany(products)
        .Refresh(Refresh.WaitFor));

    if (bulkResponse.Errors)
    {
        foreach (var item in bulkResponse.ItemsWithErrors)
        {
            _logger.LogError("Bulk index hatası: {Id} - {Error}",
                item.Id, item.Error.Reason);
        }
    }
}

// Çok büyük veri setleri için scroll ile okuma
public async IAsyncEnumerable<Product> ScrollAllAsync()
{
    var response = await _client.SearchAsync<Product>(s => s
        .Size(1000)
        .Scroll("2m")
        .Query(q => q.MatchAll()));

    while (response.Documents.Any())
    {
        foreach (var doc in response.Documents)
            yield return doc;

        response = await _client.ScrollAsync<Product>("2m", response.ScrollId);
    }

    await _client.ClearScrollAsync(c => c.ScrollId(response.ScrollId));
}
```

---

## 5. Aggregation Kullanmamak

❌ **Yanlış Kullanım:** İstatistikleri uygulama tarafında hesaplamak.

```csharp
var allProducts = await _client.SearchAsync<Product>(s => s.Size(10000));
var avgPrice = allProducts.Documents.Average(p => p.Price);
var categories = allProducts.Documents.GroupBy(p => p.CategoryId).Count();
// Tüm belgeleri çekip bellekte hesaplamak verimsiz
```

✅ **İdeal Kullanım:** Elasticsearch aggregation ile sunucu tarafında hesaplayın.

```csharp
public async Task<ProductStats> GetStatsAsync()
{
    var response = await _client.SearchAsync<Product>(s => s
        .Size(0) // Belge döndürme, sadece aggregation
        .Aggregations(a => a
            .Average("avg_price", avg => avg.Field(f => f.Price))
            .Terms("by_category", t => t
                .Field(f => f.CategoryId)
                .Size(20))
            .DateHistogram("by_month", dh => dh
                .Field(f => f.CreatedAt)
                .CalendarInterval(DateInterval.Month))));

    return new ProductStats
    {
        AveragePrice = response.Aggregations.Average("avg_price").Value ?? 0,
        CategoryCounts = response.Aggregations.Terms("by_category").Buckets
            .ToDictionary(b => b.Key, b => b.DocCount ?? 0),
        MonthlyTrend = response.Aggregations.DateHistogram("by_month").Buckets
            .Select(b => new MonthlyCount(b.Date, b.DocCount))
            .ToList()
    };
}
```
