# Dependency Injection (DI) và Vòng Đời Service trong ASP.NET Core

## 1. Vòng Đời của Service trong ASP.NET Core
ASP.NET Core cung cấp ba loại vòng đời service để sử dụng Dependency Injection:

### **1.1 Singleton**
- **Vòng đời:** Tồn tại trong suốt vòng đời của ứng dụng.
- **Chia sẻ instance:** Cùng một instance được chia sẻ giữa tất cả các request và service.
- **Ứng dụng:** Các service không có trạng thái hoặc các tài nguyên được chia sẻ như cache, cấu hình hoặc logging đơn luồng.

#### **Ví dụ:**
```csharp
services.AddSingleton<IMyService, MySingletonService>();
```

### **1.2 Scoped**
- **Vòng đời:** Tồn tại trong mỗi HTTP request.
- **Chia sẻ instance:** Cùng một instance được chia sẻ trong một request, nhưng khác nhau giữa các request.
- **Ứng dụng:** Các service phụ thuộc vào dữ liệu cụ thể của request, như context của cơ sở dữ liệu.

#### **Ví dụ:**
```csharp
services.AddScoped<IMyService, MyScopedService>();
```

### **1.3 Transient**
- **Vòng đời:** Tạo mới mỗi lần được yêu cầu.
- **Chia sẻ instance:** Không có chia sẻ; luôn tạo mới.
- **Ứng dụng:** Các thao tác nhẹ, không có trạng thái, hoặc ngắn hạn.

#### **Ví dụ:**
```csharp
services.AddTransient<IMyService, MyTransientService>();
```

---

## 2. Khả Năng Tương Thích Giữa Các Vòng Đời

### **2.1 Quy Tắc Chung**
1. **Singleton:** Có thể inject **Singleton**, **Scoped**, hoặc **Transient**.
2. **Scoped:** Có thể inject **Singleton** hoặc **Transient**.
3. **Transient:** Có thể inject **Singleton**, **Scoped**, hoặc **Transient**.

### **2.2 Bảng Tương Thích**
| **Inject**        | **Singleton** | **Scoped**  | **Transient** |
|--------------------|---------------|-------------|---------------|
| **Singleton**      | ✅ Có thể inject | ✅ Có thể inject | ✅ Có thể inject |
| **Scoped**         | ⚠️ Không khuyến khích | ✅ Có thể inject | ✅ Có thể inject |
| **Transient**      | ✅ Có thể inject | ✅ Có thể inject | ✅ Có thể inject |

### **2.3 Trường Hợp Đặc Biệt**
- **Scoped trong Singleton:** Inject Scoped vào Singleton không được khuyến khích vì Scoped phụ thuộc vào HTTP request hiện tại, trong khi Singleton tồn tại suốt vòng đời ứng dụng.
  - **Giải pháp:** Sử dụng `IServiceProvider` để tạo một scope mới.

#### **Ví dụ Scoped trong Singleton:**
```csharp
public class MySingletonService
{
    private readonly IServiceProvider _serviceProvider;

    public MySingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void UseScopedService()
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
            scopedService.DoWork();
        }
    }
}
```

---

## 3. IServiceProvider
`IServiceProvider` là một interface cung cấp quyền truy cập vào các service được quản lý bởi container Dependency Injection.

### **3.1 Khi Nào Sử Dụng `IServiceProvider`**

#### **1. Truy Cập Scoped/Transient Service trong Singleton**
```csharp
public class MySingletonService
{
    private readonly IServiceProvider _serviceProvider;

    public MySingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void PerformScopedWork()
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
            scopedService.DoWork();
        }
    }
}
```

#### **2. Truy Cập Service Một Cách Động**
Hữu ích khi service cần resolve phụ thuộc vào điều kiện runtime.

```csharp
public class DynamicServiceResolver
{
    private readonly IServiceProvider _serviceProvider;

    public DynamicServiceResolver(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void ResolveAndUseService(string serviceName)
    {
        if (serviceName == "ServiceA")
        {
            var serviceA = _serviceProvider.GetService<IServiceA>();
            serviceA.DoWork();
        }
        else if (serviceName == "ServiceB")
        {
            var serviceB = _serviceProvider.GetService<IServiceB>();
            serviceB.DoWork();
        }
    }
}
```

#### **3. Middleware hoặc Background Services**
Trong Middleware hoặc Background Services (cả hai đều là Singleton), `IServiceProvider` cho phép tạo và quản lý các Scoped service.

##### **Ví dụ Middleware:**
```csharp
public class MyMiddleware
{
    private readonly RequestDelegate _next;

    public MyMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context, IServiceProvider serviceProvider)
    {
        using (var scope = serviceProvider.CreateScope())
        {
            var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
            scopedService.DoWork();
        }

        await _next(context);
    }
}
```

##### **Ví dụ Background Service:**
```csharp
public class MyBackgroundService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;

    public MyBackgroundService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using (var scope = _serviceProvider.CreateScope())
            {
                var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
                scopedService.DoBackgroundWork();
            }

            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

---

## 4. Thực Hành Tốt Nhất và Cân Nhắc
1. **Ưu Tiên Constructor Injection:** Chỉ sử dụng `IServiceProvider` khi không thể sử dụng constructor injection.
2. **Quản Lý Vòng Đời Hợp Lý:** Tránh inject Scoped service trực tiếp vào Singleton.
3. **Hạn Chế Dynamic Service Resolution:** Lạm dụng `IServiceProvider.GetService` có thể làm code khó bảo trì và kiểm thử.
4. **Sử Dụng Middleware và Background Services Cẩn Thận:** Tạo scope cho các Scoped service trong các môi trường này để tránh vấn đề về vòng đời.

---

Tài liệu này cung cấp hướng dẫn toàn diện để hiểu Dependency Injection, vòng đời của service và cách sử dụng `IServiceProvider` trong ASP.NET Core.

