# Global Using Directives

C# 10.0 ile gelen `global using` direktifleri, projenizdeki tüm dosyalarda ortak kullanılan namespace'leri tek bir yerde tanımlamanıza olanak tanır. Ancak, yanlış kullanımlar bağımlılık yönetimini ve kod okunabilirliğini olumsuz etkileyebilir.

---

## 1. Her Dosyada Tekrarlayan `using` İfadeleri

❌ **Yanlış Kullanım:** Aynı `using` ifadelerini her dosyanın başına tekrar tekrar yazmak.

```csharp
// Controllers/ProductController.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc;

// Controllers/OrderController.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc;
```

✅ **İdeal Kullanım:** Ortak namespace'leri `global using` ile merkezi bir dosyada tanımlayın.

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using Microsoft.AspNetCore.Mvc;
```

Artık her dosyada bu namespace'leri tekrar yazmanıza gerek kalmaz.

---

## 2. `global using` ile Tüm Namespace'leri Eklemek

❌ **Yanlış Kullanım:** Projede kullanılan her namespace'i `global using` olarak tanımlamak.

```csharp
// GlobalUsings.cs
global using System;
global using System.IO;
global using System.Text;
global using System.Text.Json;
global using System.Net.Http;
global using System.Threading.Tasks;
global using Microsoft.AspNetCore.Mvc;
global using Microsoft.EntityFrameworkCore;
global using AutoMapper;
global using FluentValidation;
global using MediatR;
```

✅ **İdeal Kullanım:** Yalnızca projede yaygın olarak kullanılan namespace'leri `global using` yapın. Sadece birkaç dosyada kullanılanları lokal bırakın.

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Microsoft.AspNetCore.Mvc;
```

```csharp
// Sadece ilgili dosyada
using AutoMapper;
using FluentValidation;
```

---

## 3. `global using static` Kullanımını Abartmak

❌ **Yanlış Kullanım:** Çok sayıda `global using static` tanımlayarak metodların hangi sınıfa ait olduğunu belirsizleştirmek.

```csharp
global using static System.Console;
global using static System.Math;
global using static System.IO.Path;

// Kullanım
var result = Max(10, 20); // System.Math.Max mı yoksa başka bir şey mi?
WriteLine(result); // Nereden geliyor?
```

✅ **İdeal Kullanım:** `global using static` kullanımını sınırlı tutun ve yalnızca çok sık kullanılan yardımcı sınıflar için tercih edin.

```csharp
global using static System.Math;

// Kullanım
var result = Max(10, 20);
Console.WriteLine(result); // Kaynak açıkça belirtilmiş
```

---

## 4. `global using` ile İsim Çakışmalarını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Farklı namespace'lerdeki aynı isimli türlerin çakışmasını dikkate almamak.

```csharp
// GlobalUsings.cs
global using System.Threading.Tasks;
global using MyProject.Tasks; // "Task" isim çakışması!
```

✅ **İdeal Kullanım:** Çakışma riski olan namespace'leri `global using` yapmayın veya alias kullanın.

```csharp
// GlobalUsings.cs
global using System.Threading.Tasks;
global using ProjectTask = MyProject.Tasks.ProjectTask;
```

---

## 5. Katmanlı Mimaride Yanlış Scope Kullanımı

❌ **Yanlış Kullanım:** Altyapı katmanına ait namespace'leri tüm projede `global using` yapmak.

```csharp
// Domain katmanında GlobalUsings.cs
global using Microsoft.EntityFrameworkCore;
global using Microsoft.AspNetCore.Http;
```

✅ **İdeal Kullanım:** Her katmanda yalnızca o katmana uygun namespace'leri `global using` olarak tanımlayın.

```csharp
// Domain katmanında GlobalUsings.cs
global using System;
global using System.Collections.Generic;

// Infrastructure katmanında GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using Microsoft.EntityFrameworkCore;
```

---

## 6. `.csproj` ile `global using` Kullanımı

`global using` ifadelerini `.csproj` dosyasında da tanımlayabilirsiniz. Bu, özellikle SDK projelerinde otomatik olarak eklenen `implicit usings` ile birlikte kullanışlıdır.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <Using Include="AutoMapper" />
    <Using Include="MediatR" />
    <Using Include="System.Text.Json" Static="true" />
    <Using Include="MyProject.Tasks.ProjectTask" Alias="ProjectTask" />
  </ItemGroup>
</Project>
```

**Not:** `ImplicitUsings` etkinleştirildiğinde, .NET SDK projede yaygın namespace'leri otomatik olarak ekler. Bu özelliği `enable` yerine `disable` yaparak kapatabilirsiniz.