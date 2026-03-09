# Input Validation

Input validation, kullanıcı girdilerinin güvenli ve beklenen formatta olmasını sağlar; eksik doğrulama SQL Injection, XSS ve diğer saldırılara zemin hazırlar.

---

## 1. SQL Injection'a Açık Sorgular

❌ **Yanlış Kullanım:** Kullanıcı girdisini doğrudan SQL sorgusuna eklemek.

```csharp
[HttpGet("api/products")]
public async Task<IActionResult> Search(string name)
{
    var products = await _context.Products
        .FromSqlRaw($"SELECT * FROM Products WHERE Name = '{name}'") // SQL Injection!
        .ToListAsync();
    return Ok(products);
}
// name = "'; DROP TABLE Products; --" gönderilirse tablo silinir
```

✅ **İdeal Kullanım:** Parametreli sorgular veya LINQ kullanın.

```csharp
[HttpGet("api/products")]
public async Task<IActionResult> Search(string name)
{
    var products = await _context.Products
        .Where(p => p.Name.Contains(name))
        .ToListAsync();
    return Ok(products);
}

// Raw SQL gerekiyorsa parametreli kullanın
var products = await _context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {name}")
    .ToListAsync();
```

---

## 2. XSS'e Açık Çıktılar

❌ **Yanlış Kullanım:** Kullanıcı girdisini encode etmeden döndürmek.

```csharp
[HttpGet("api/greeting")]
public IActionResult Greet(string name)
{
    return Content($"<h1>Merhaba {name}</h1>", "text/html");
    // name = "<script>alert('XSS')</script>" gönderilirse script çalışır
}
```

✅ **İdeal Kullanım:** HTML encode yaparak çıktıyı güvenli hale getirin.

```csharp
[HttpGet("api/greeting")]
public IActionResult Greet(string name)
{
    var encoded = HtmlEncoder.Default.Encode(name);
    return Content($"<h1>Merhaba {encoded}</h1>", "text/html");
}

// Veya API'lerde JSON response kullanın (otomatik encode)
[HttpGet("api/greeting")]
public IActionResult Greet(string name)
{
    return Ok(new { message = $"Merhaba {name}" });
}
```

---

## 3. Model Validation Yapmamak

❌ **Yanlış Kullanım:** Request model'ini doğrulamadan işlemek.

```csharp
[HttpPost("api/users")]
public async Task<IActionResult> Create(CreateUserDto dto)
{
    var user = new User
    {
        Email = dto.Email,        // null olabilir
        Name = dto.Name,          // 10.000 karakter olabilir
        Age = dto.Age             // -1 olabilir
    };
    await _context.Users.AddAsync(user);
    await _context.SaveChangesAsync();
    return Ok(user);
}
```

✅ **İdeal Kullanım:** FluentValidation ile kapsamlı doğrulama yapın.

```csharp
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("E-posta boş olamaz.")
            .EmailAddress().WithMessage("Geçerli bir e-posta adresi giriniz.")
            .MaximumLength(256);

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ad boş olamaz.")
            .MaximumLength(100)
            .Matches(@"^[a-zA-ZğüşıöçĞÜŞİÖÇ\s]+$").WithMessage("Ad yalnızca harf içermelidir.");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Yaş 18-120 arasında olmalıdır.");
    }
}
```

---

## 4. File Upload Validasyonu Yapmamak

❌ **Yanlış Kullanım:** Yüklenen dosyayı kontrol etmeden kaydetmek.

```csharp
[HttpPost("api/upload")]
public async Task<IActionResult> Upload(IFormFile file)
{
    var path = Path.Combine("uploads", file.FileName); // Path traversal riski
    using var stream = new FileStream(path, FileMode.Create);
    await file.CopyToAsync(stream);
    return Ok();
}
// file.FileName = "../../etc/passwd" olabilir
```

✅ **İdeal Kullanım:** Dosya türünü, boyutunu ve adını doğrulayın.

```csharp
private static readonly HashSet<string> AllowedExtensions = new() { ".jpg", ".png", ".pdf" };
private const long MaxFileSize = 5 * 1024 * 1024; // 5MB

[HttpPost("api/upload")]
public async Task<IActionResult> Upload(IFormFile file)
{
    if (file.Length == 0 || file.Length > MaxFileSize)
        return BadRequest("Dosya boyutu geçersiz.");

    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
    if (!AllowedExtensions.Contains(extension))
        return BadRequest("Desteklenmeyen dosya türü.");

    var safeFileName = $"{Guid.NewGuid()}{extension}";
    var path = Path.Combine("uploads", safeFileName);

    using var stream = new FileStream(path, FileMode.Create);
    await file.CopyToAsync(stream);

    return Ok(new { FileName = safeFileName });
}
```

---

## 5. Mass Assignment (Over-Posting) Koruması

❌ **Yanlış Kullanım:** Entity'yi doğrudan bind etmek.

```csharp
[HttpPut("api/users/{id}")]
public async Task<IActionResult> Update(int id, User user)
{
    _context.Users.Update(user);
    await _context.SaveChangesAsync();
    return Ok();
    // Kullanıcı IsAdmin=true, Role="Admin" gibi alanları gönderebilir
}
```

✅ **İdeal Kullanım:** DTO ile sadece izin verilen alanları kabul edin.

```csharp
public record UpdateUserDto(string Name, string Email, string Phone);

[HttpPut("api/users/{id}")]
public async Task<IActionResult> Update(int id, UpdateUserDto dto)
{
    var user = await _context.Users.FindAsync(id);
    if (user == null) return NotFound();

    user.Name = dto.Name;
    user.Email = dto.Email;
    user.Phone = dto.Phone;
    // IsAdmin, Role gibi alanlar DTO'da yok, değiştirilemez

    await _context.SaveChangesAsync();
    return Ok();
}
```