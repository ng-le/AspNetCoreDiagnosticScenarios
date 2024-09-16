### **Outbox/Inbox Pattern in Microservices**

In a distributed system, ensuring **message delivery reliability** and **data consistency** between microservices is a major challenge. The **Outbox/Inbox pattern** is a solution to ensure that messages between services are reliably delivered, even in the face of system failures, without needing distributed transactions.

#### **What is the Outbox Pattern?**

The **Outbox Pattern** ensures that **a message is only sent if the associated transaction in the service’s database is successfully committed**. This guarantees **atomicity** between data changes in the database and the events/messages sent to other services.

- **Outbox**: Stores events/messages within the local transaction boundary (usually in a database) so that they are sent **only after the transaction is committed**.
- **Inbox**: Ensures that **duplicate messages** are handled idempotently by recording the **message ID** in a separate table to prevent processing the same message multiple times.

### **How the Outbox/Inbox Pattern Works**

1. **Outbox Process (Message Sending)**
   - A service performs some database operations (e.g., creating an order).
   - The event or message that needs to be sent to other services is stored in an **Outbox table** in the same database transaction.
   - After the transaction is committed, a background worker reads the Outbox table and sends the message to a message broker (like RabbitMQ).
   - Once the message is successfully sent, the Outbox entry is marked as processed.

2. **Inbox Process (Message Receiving)**
   - When a service receives a message from another service, it first checks if the message has already been processed by looking at the **Inbox table**.
   - If the message is new, it performs the required business logic and stores the message ID in the Inbox table.
   - If the message is a duplicate, it is ignored.

This pattern ensures **eventual consistency** and **resilience** against failures, and it avoids the need for distributed transactions.

---

### **Outbox/Inbox Pattern Example: Order and Payment System**

Let’s implement a real-world scenario where:
1. **OrderService** places an order and creates an entry in its database.
2. After creating the order, **OrderService** sends a message to the **PaymentService** to process the payment.
3. Both services will implement the **Outbox/Inbox** pattern to ensure reliable message delivery.

We will use **MassTransit** for handling messages and **RabbitMQ** as the message broker.

---

### **1. Database Schema for Outbox and Inbox**

#### **Outbox Table**

```sql
CREATE TABLE OutboxMessage (
    Id BIGINT PRIMARY KEY AUTO_INCREMENT,
    MessageId VARCHAR(255) NOT NULL,
    MessageType VARCHAR(255),
    Body TEXT,
    Processed BOOLEAN DEFAULT FALSE,
    CreatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### **Inbox Table**

```sql
CREATE TABLE InboxMessage (
    MessageId VARCHAR(255) PRIMARY KEY,
    ProcessedAt DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### **2. Setting up MassTransit and RabbitMQ**

Ensure the following dependencies are installed:

```bash
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

---

### **3. Outbox Implementation (Order Service)**

The `OrderService` creates an order and stores a message in the **Outbox** table as part of the same database transaction.

#### **OrderService – Creating Order and Storing Outbox Message**

```csharp
public class OrderService
{
    private readonly OrderDbContext _context;

    public OrderService(OrderDbContext context)
    {
        _context = context;
    }

    public async Task PlaceOrder(Order order)
    {
        using (var transaction = await _context.Database.BeginTransactionAsync())
        {
            try
            {
                // Create the order in the database
                _context.Orders.Add(order);
                await _context.SaveChangesAsync();

                // Create an Outbox message for the payment event
                var paymentMessage = new PaymentRequested
                {
                    OrderId = order.Id,
                    Amount = order.TotalAmount
                };

                var outboxMessage = new OutboxMessage
                {
                    MessageId = Guid.NewGuid().ToString(),
                    MessageType = nameof(PaymentRequested),
                    Body = JsonConvert.SerializeObject(paymentMessage),
                    Processed = false
                };

                _context.OutboxMessages.Add(outboxMessage);
                await _context.SaveChangesAsync();

                // Commit the transaction
                await transaction.CommitAsync();
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        }
    }
}
```

#### **Outbox Worker to Send Messages to RabbitMQ**

A background worker will continuously check the Outbox table for unprocessed messages, send them to RabbitMQ, and then mark them as processed.

```csharp
public class OutboxWorker : BackgroundService
{
    private readonly OrderDbContext _context;
    private readonly IBus _bus;

    public OutboxWorker(OrderDbContext context, IBus bus)
    {
        _context = context;
        _bus = bus;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var unprocessedMessages = await _context.OutboxMessages
                .Where(m => !m.Processed)
                .ToListAsync();

            foreach (var outboxMessage in unprocessedMessages)
            {
                // Deserialize the message body
                var paymentMessage = JsonConvert.DeserializeObject<PaymentRequested>(outboxMessage.Body);

                // Send the message to RabbitMQ via MassTransit
                await _bus.Publish(paymentMessage);

                // Mark the message as processed
                outboxMessage.Processed = true;
                await _context.SaveChangesAsync();
            }

            await Task.Delay(5000, stoppingToken); // Check every 5 seconds
        }
    }
}
```

---

### **4. Inbox Implementation (Payment Service)**

The **PaymentService** will implement an Inbox pattern to ensure idempotency. When it receives a `PaymentRequested` message, it checks if the message has already been processed by looking at the **Inbox** table.

#### **PaymentService – Inbox Consumer**

```csharp
public class PaymentRequestedConsumer : IConsumer<PaymentRequested>
{
    private readonly PaymentDbContext _context;

    public PaymentRequestedConsumer(PaymentDbContext context)
    {
        _context = context;
    }

    public async Task Consume(ConsumeContext<PaymentRequested> context)
    {
        var messageId = context.MessageId?.ToString();

        // Check if the message has already been processed
        var alreadyProcessed = await _context.InboxMessages
            .AnyAsync(m => m.MessageId == messageId);

        if (alreadyProcessed)
        {
            // If the message is a duplicate, do nothing
            return;
        }

        // Process the payment (business logic)
        var payment = new Payment
        {
            OrderId = context.Message.OrderId,
            Amount = context.Message.Amount,
            Status = "Processed"
        };

        _context.Payments.Add(payment);
        await _context.SaveChangesAsync();

        // Add the message to the Inbox table to mark it as processed
        var inboxMessage = new InboxMessage
        {
            MessageId = messageId
        };

        _context.InboxMessages.Add(inboxMessage);
        await _context.SaveChangesAsync();
    }
}
```

---

### **5. Configuring MassTransit with RabbitMQ**

Configure **MassTransit** and RabbitMQ in the `Startup.cs` (or equivalent configuration file):

```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<PaymentRequestedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");

        cfg.ReceiveEndpoint("payment-queue", e =>
        {
            e.ConfigureConsumer<PaymentRequestedConsumer>(context);
        });
    });
});

services.AddMassTransitHostedService();
```

---

### **6. Database Configuration for OrderDbContext and PaymentDbContext**

Define the necessary DbContext classes to manage the Outbox and Inbox tables, as well as the business data (e.g., orders, payments).

#### **OrderDbContext**

```csharp
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OutboxMessage> OutboxMessages { get; set; }

    public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options)
    {
    }
}
```

#### **PaymentDbContext**

```csharp
public class PaymentDbContext : DbContext
{
    public DbSet<Payment> Payments { get; set; }
    public DbSet<InboxMessage> InboxMessages { get; set; }

    public PaymentDbContext(DbContextOptions<PaymentDbContext> options) : base(options)
    {
    }
}
```

---

### **7. Running the Application**

- Run RabbitMQ (`docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management`).
- Start both the `OrderService` and `PaymentService`.
- Place an order by invoking the `PlaceOrder` method.
- The `OutboxWorker` will send the message to RabbitMQ, and the `PaymentRequestedConsumer` will handle it in the `PaymentService`, ensuring that duplicate messages are not processed.

---

### **Key Points**

1. **Outbox Pattern**: Ensures messages are sent only after the transaction in the local database is committed, preventing loss of events due to failures.
2. **Inbox Pattern**: Prevents duplicate message processing by checking an Inbox table for already processed message IDs.
3.

 **MassTransit**: Helps to manage message communication via RabbitMQ between microservices.
4. **Idempotency**: Guaranteed by checking for message duplicates using the Inbox table.

This implementation ensures reliable, resilient, and eventually consistent message delivery between services in a distributed system.