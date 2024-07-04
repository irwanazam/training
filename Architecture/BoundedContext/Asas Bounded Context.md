***Asas Bounded Context***

Bounded Context adalah konsep dalam Domain-Driven Design (DDD) yang membantu kita memecah aplikasi besar 
menjadi bahagian-bahagian yang lebih kecil dan mudah diurus. 
Bounded Context ni macam 'ruang lingkup' dalam dunia Domain-Driven Design (DDD). Idea dia, dalam setiap ruang lingkup tu, 
istilah dan konsep domain tu ada makna tersendiri. Contohnya, dalam projek Conference Management, kita boleh ada beberapa Bounded Context macam Speaker Management, Attendee Management, Session Scheduling, dan Ticketing & Payment. Setiap satu focus pada aspek tertentu tanpa campur tangan dengan yang lain.
Setiap bahagian ini, atau "context", mempunyai batasan jelas dan bertanggungjawab terhadap sebahagian domain tertentu.
Cara ini untuk memastikan perubahan didalam sesuatu context/bahagian tak akan secara mengganggu domain model yang lain didalam context/bahagian 
yang lain.

Secara umum komunikasi antara bounded context adalah melalui konsep "Event". Sesuatu event yang berlaku akan di "Publish" dan pada mana-mana Bounded Context
yang memerlukan perlu "Subscribe" kepada event tersebut untuk membuat perubahan pada domain model mereka sendiri.

**Kenapa Penting?**

Kejelasan dan Fokus: Dengan Bounded Context, kita boleh fokus pada bahagian tertentu tanpa perlu risau tentang keseluruhan sistem. Ini memudahkan kita memahami dan bekerja dengan kod.
Independensi: Setiap Bounded Context adalah bebas. Maksudnya, perubahan dalam satu context tak akan memberi kesan kepada yang lain. Ini mengurangkan risiko bug dan masalah lain.
Scalability: Kita boleh menambah atau memperbaiki Bounded Context secara berasingan. Ini menjadikan aplikasi kita lebih scalable dan flexible.
Peningkatan Komunikasi: Menggunakan istilah dan logik yang konsisten dalam setiap Bounded Context membantu meningkatkan komunikasi 
antara ahli pasukan dan pemegang taruh kerana setiap orang memahami dan bersetuju dengan makna yang ditetapkan dalam konteks tersebut.

Untuk modelkan Buyer dalam bounded context Order dan uruskan kes penggunaan seperti menambah produk ke dalam bakul dan membuat pesanan, 
kita akan reka API minimal dalam .NET Core 8.0 dengan dua bounded contexts: Customer dan Order.

**Gambaran Umum Bounded Contexts**

Bounded Context Customer:

Entities: Customer
Tanggungjawab: Menguruskan data pelanggan.

Bounded Context Order:
Entities: Buyer, Basket, Order
Tanggungjawab: Menguruskan pesanan, bakul, dan hubungan antara pelanggan dengan pesanan mereka.

### Rekabentuk dan Pelaksanaan yang Dikemaskini

#### Langkah 1: Definisikan Model Domain

**Bounded Context Customer**:
```csharp
// Bounded Context Customer
namespace CustomerContext
{
    public class Customer
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
        // Properties tambahan
    }
}
```

**Bounded Context Order**:
```csharp
// Bounded Context Order
namespace OrderContext
{
    public class Buyer
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
        // Properties tambahan
    }

    public class Basket
    {
        public Guid Id { get; set; }
        public Guid BuyerId { get; set; }
        public List<BasketItem> Items { get; set; } = new();
    }

    public class BasketItem
    {
        public Guid ProductId { get; set; }
        public int Quantity { get; set; }
    }

    public class Order
    {
        public Guid Id { get; set; }
        public Guid BuyerId { get; set; }
        public List<OrderItem> Items { get; set; } = new();
        public DateTime OrderDate { get; set; }
    }

    public class OrderItem
    {
        public Guid ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal Price { get; set; }
    }
}
```

#### Langkah 2: Konfigurasi DbContext

**Bounded Context Customer**:
```csharp
// Bounded Context Customer
using Microsoft.EntityFrameworkCore;

namespace CustomerContext
{
    public class CustomerDbContext : DbContext
    {
        public DbSet<Customer> Customers { get; set; }

        public CustomerDbContext(DbContextOptions<CustomerDbContext> options) : base(options) { }
    }
}
```

**Bounded Context Order**:
```csharp
// Bounded Context Order
using Microsoft.EntityFrameworkCore;

namespace OrderContext
{
    public class OrderDbContext : DbContext
    {
        public DbSet<Buyer> Buyers { get; set; }
        public DbSet<Basket> Baskets { get; set; }
        public DbSet<Order> Orders { get; set; }

        public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options) { }
    }
}
```

#### Langkah 3: Laksanakan Event Bus

```csharp
// In-Memory Event Bus
public interface IEventBus
{
    void Publish<T>(T @event);
    void Subscribe<T>(Action<T> handler);
}

public class InMemoryEventBus : IEventBus
{
    private readonly Dictionary<Type, List<Action<object>>> _handlers = new();

    public void Publish<T>(T @event)
    {
        if (_handlers.TryGetValue(typeof(T), out var handlers))
        {
            foreach (var handler in handlers)
            {
                handler(@event);
            }
        }
    }

    public void Subscribe<T>(Action<T> handler)
    {
        if (!_handlers.ContainsKey(typeof(T)))
        {
            _handlers[typeof(T)] = new List<Action<object>>();
        }

        _handlers[typeof(T)].Add(e => handler((T)e));
    }
}
```

#### Langkah 4: Definisikan Event Domain

```csharp
// Domain Events
public class CustomerCreatedEvent
{
    public Guid CustomerId { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}
```

#### Langkah 5: Laksanakan API dengan Integrasi Event Bus

**Customer API**:
```csharp
// Customer API
using CustomerContext;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<CustomerDbContext>(options =>
    options.UseInMemoryDatabase("CustomerDb"));
builder.Services.AddSingleton<IEventBus, InMemoryEventBus>();
var app = builder.Build();

var eventBus = app.Services.GetRequiredService<IEventBus>();

app.MapPost("/customers", async (Customer customer, CustomerDbContext db, IEventBus eventBus) =>
{
    db.Customers.Add(customer);
    await db.SaveChangesAsync();
    
    var customerCreatedEvent = new CustomerCreatedEvent 
    {
        CustomerId = customer.Id,
        Name = customer.Name,
        Email = customer.Email
    };
    eventBus.Publish(customerCreatedEvent);

    return Results.Created($"/customers/{customer.Id}", customer);
});

app.Run();
```

**Order API**:
```csharp
// Order API
using OrderContext;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<OrderDbContext>(options =>
    options.UseInMemoryDatabase("OrderDb"));
builder.Services.AddSingleton<IEventBus, InMemoryEventBus>();
var app = builder.Build();

var eventBus = app.Services.GetRequiredService<IEventBus>();

eventBus.Subscribe<CustomerCreatedEvent>(async e =>
{
    using var scope = app.Services.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<OrderDbContext>();

    var buyer = new Buyer
    {
        Id = e.CustomerId,
        Name = e.Name,
        Email = e.Email
    };

    dbContext.Buyers.Add(buyer);
    await dbContext.SaveChangesAsync();
});

app.Run();
```

### Penjelasan Use Case

1. **Mencipta Customer**:
    - Bila `Customer` baru dicipta dalam `Customer` API, `CustomerCreatedEvent` akan diterbitkan kepada event bus.
    - `Order` API akan subscribe kepada `CustomerCreatedEvent` dan mencipta entiti `Buyer` yang bersamaan dalam bounded context `Order` menggunakan data dari event tersebut.

2. **Mengekalkan Hubungan**:
    - Entiti `Customer` dan `Buyer` adalah berasingan tetapi berkaitan melalui `CustomerId`.
    - Sebarang perubahan kepada entiti `Customer` boleh disebarkan kepada entiti `Buyer` menggunakan mekanisme event yang sama.

### Akhir Kata

Rekabentuk ini menunjukkan cara untuk mengekalkan entiti `Customer` dan `Buyer` yang berasingan dalam bounded context masing-masing sambil memastikan mereka tetap diselaraskan melalui penggunaan event domain dan event bus. 
Pendekatan ini mengekalkan integriti dan kebebasan setiap bounded context sambil membenarkan tingkah laku yang terkoordinasi di antara mereka.

![boundedcontext(2)_d7084c0f](https://github.com/irwanazam/training/assets/99638906/2f7bee9e-5dab-4150-ba86-7e2762ca7189)


