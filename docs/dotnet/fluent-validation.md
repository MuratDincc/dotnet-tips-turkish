# Validation ve FluentValidation Kullanımı

Validation (doğrulama), kullanıcı girişlerini kontrol ederek uygulamanın güvenliğini ve bütünlüğünü sağlamak için kritik bir süreçtir. FluentValidation, güçlü ve esnek doğrulama kuralları oluşturmayı kolaylaştırır. Yanlış kullanılan doğrulama kuralları veri tutarsızlığına ve güvenlik açıklarına yol açabilir.

---

## 1. Doğrudan Kodda Validation Kuralları Yazmak

❌ **Yanlış Kullanım:** Doğrulama kurallarını doğrudan iş mantığına dahil etmek.

```csharp
public IActionResult CreateUser(User user)
{
    if (string.IsNullOrWhiteSpace(user.Name) || user.Age < 18)
    {
        return BadRequest("Hatalı kullanıcı verisi.");
    }

    // İş mantığı
    return Ok();
}
```

✅ **İdeal Kullanım:** FluentValidation ile doğrulama kurallarını ayırın.

```csharp
public class UserValidator : AbstractValidator<User>
{
    public UserValidator()
    {
        RuleFor(user => user.Name).NotEmpty().WithMessage("Kullanıcı adı boş olamaz.");
        RuleFor(user => user.Age).GreaterThanOrEqualTo(18).WithMessage("Yaş 18'den büyük olmalıdır.");
    }
}

public IActionResult CreateUser(User user)
{
    var validator = new UserValidator();
    var result = validator.Validate(user);
    if (!result.IsValid)
    {
        return BadRequest(result.Errors);
    }

    // İş mantığı
    return Ok();
}
```

---

## 2. Çok Fazla Kurala Sahip Karmaşık Validatorlar

❌ **Yanlış Kullanım:** Tek bir sınıfta aşırı derecede karmaşık doğrulama kuralları tanımlamak.

```csharp
public class ComplexValidator : AbstractValidator<ComplexObject>
{
    public ComplexValidator()
    {
        RuleFor(x => x.Property1).NotEmpty();
        RuleFor(x => x.Property2).GreaterThan(0);
        RuleFor(x => x.NestedObject.Property).NotEmpty();
        // ... daha fazla karmaşık kural
    }
}
```

✅ **İdeal Kullanım:** Kuralları alt validator'lara ayırarak yönetilebilir hale getirin.

```csharp
public class NestedObjectValidator : AbstractValidator<NestedObject>
{
    public NestedObjectValidator()
    {
        RuleFor(x => x.Property).NotEmpty();
    }
}

public class ComplexValidator : AbstractValidator<ComplexObject>
{
    public ComplexValidator()
    {
        RuleFor(x => x.Property1).NotEmpty();
        RuleFor(x => x.Property2).GreaterThan(0);
        RuleFor(x => x.NestedObject).SetValidator(new NestedObjectValidator());
    }
}
```

---

## 3. `CascadeMode` Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Her kuralın ayrı ayrı tetiklenmesine neden olmak.

```csharp
public UserValidator()
{
    RuleFor(user => user.Email).NotEmpty().EmailAddress();
}
```

✅ **İdeal Kullanım:** CascadeMode kullanarak doğrulama işlemini optimize edin.

```csharp
public UserValidator()
{
    RuleFor(user => user.Email)
        .Cascade(CascadeMode.Stop)
        .NotEmpty()
        .EmailAddress();
}
```

---

## 4. Hataları Kullanıcı Dostu Şekilde Dönüştürmemek

❌ **Yanlış Kullanım:** Hataları kullanıcı dostu olmayan bir formatta döndürmek.

```csharp
return BadRequest(result.Errors);
```

✅ **İdeal Kullanım:** Hataları kullanıcı dostu bir formatta dönüştürün.

```csharp
return BadRequest(result.Errors.Select(e => e.ErrorMessage));
```

---

## 5. Global Doğrulama Yönetimi Kullanımını Atlamak

❌ **Yanlış Kullanım:** Her yerde manuel olarak validator çağırmak.

```csharp
var validator = new UserValidator();
var result = validator.Validate(user);
```

✅ **İdeal Kullanım:** Global doğrulama yönetimini etkinleştirin.

```csharp
services.AddFluentValidationAutoValidation()
        .AddValidatorsFromAssemblyContaining<UserValidator>();
```

---

## 6. Veri Tabanı Bağımlılıklarını Validator'da Kullanmak

❌ **Yanlış Kullanım:** Validator içinde doğrudan veri tabanı sorguları yapmak.

```csharp
RuleFor(user => user.Email).Must(email => !dbContext.Users.Any(u => u.Email == email))
    .WithMessage("Bu e-posta zaten kullanılıyor.");
```

✅ **İdeal Kullanım:** Bağımlılıkları constructor üzerinden inject edin.

```csharp
public class UserValidator : AbstractValidator<User>
{
    private readonly MyDbContext _dbContext;

    public UserValidator(MyDbContext dbContext)
    {
        _dbContext = dbContext;

        RuleFor(user => user.Email).Must(IsUniqueEmail)
            .WithMessage("Bu e-posta zaten kullanılıyor.");
    }

    private bool IsUniqueEmail(string email)
    {
        return !_dbContext.Users.Any(u => u.Email == email);
    }
}
```