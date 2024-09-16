### **SAGA Transaction Pattern (Distributed Transactions)**

In microservices, transactions often span multiple services. A traditional database transaction (ACID) doesn't fit well because each microservice manages its own database. Here, **SAGA** is used as a solution to handle long-running, distributed transactions across multiple services in a way that maintains data consistency.

#### **What is a SAGA Pattern?**

A **SAGA** is a sequence of local transactions where each transaction updates the state within a single service and publishes an event. If a step fails, the SAGA must execute compensating transactions to undo the impact of the preceding transactions.

#### **Types of SAGA Patterns**

1. **Choreography (Event-based):**
   - Each service listens to events from other services and performs local transactions, then publishes an event.
   - No central coordinator.
   - Best for simple workflows where services can react to each other’s events.

2. **Orchestration (Command-based):**
   - A central orchestrator manages the saga’s steps.
   - Each service performs a local transaction when instructed by the orchestrator and reports back.
   - Best for complex workflows with many dependencies.

### **Example: E-commerce Order SAGA**

Let's say we want to implement an order creation process for an e-commerce platform using SAGA. When a customer places an order, the following steps are involved:

1. **Order Service**: Create the order.
2. **Payment Service**: Process the payment.
3. **Inventory Service**: Reserve the products.
4. **Shipping Service**: Prepare for delivery.

If any step fails, the previous steps must be rolled back (i.e., cancel payment, release inventory, and cancel the order).

---

### **Implementing a SAGA in .NET Core using MassTransit, RabbitMQ, and Orchestration**

We'll implement the **Order Creation** SAGA using **MassTransit** for orchestration and **RabbitMQ** for messaging. In this scenario:

- We have an **OrderService**, **PaymentService**, **InventoryService**, and **ShippingService**.
- The **OrderService** acts as the SAGA orchestrator, coordinating the entire process.

#### **1. Setup MassTransit and RabbitMQ in .NET Core**

First, we'll configure **MassTransit** with RabbitMQ in the project.

#### **Install Dependencies**

```bash
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.AspNetCore
```

#### **MassTransit Configuration**

In `Startup.cs` or the equivalent in your program configuration:

```csharp
services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .InMemoryRepository();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");

        cfg.ReceiveEndpoint("order-state", e =>
        {
            e.ConfigureSaga<OrderState>(context);
        });
    });
});

services.AddMassTransitHostedService();
```

#### **2. Define the SAGA State**

The saga state represents the current state of the order through the process.

```csharp
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; }
    public Guid OrderId { get; set; }
    public Guid PaymentId { get; set; }
    public Guid InventoryId { get; set; }
}
```

#### **3. Define Events (Messages)**

We define the messages exchanged between services:

```csharp
public interface OrderSubmitted
{
    Guid OrderId { get; }
    decimal Amount { get; }
}

public interface PaymentProcessed
{
    Guid OrderId { get; }
    bool Success { get; }
}

public interface InventoryReserved
{
    Guid OrderId { get; }
    bool Success { get; }
}

public interface ShippingStarted
{
    Guid OrderId { get; }
}
```

#### **4. Define the SAGA State Machine**

The state machine orchestrates the steps of the order process.

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Submitted { get; private set; }
    public State PaymentProcessed { get; private set; }
    public State InventoryReserved { get; private set; }
    public State ShippingStarted { get; private set; }
    public Event<OrderSubmitted> OrderSubmitted { get; private set; }
    public Event<PaymentProcessed> PaymentProcessed { get; private set; }
    public Event<InventoryReserved> InventoryReserved { get; private set; }
    public Event<ShippingStarted> ShippingStarted { get; private set; }

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderSubmitted, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => ShippingStarted, x => x.CorrelateById(context => context.Message.OrderId));

        Initially(
            When(OrderSubmitted)
                .Then(context => context.Instance.OrderId = context.Data.OrderId)
                .TransitionTo(Submitted)
                .Publish(context => new ProcessPayment
                {
                    OrderId = context.Data.OrderId,
                    Amount = context.Data.Amount
                })
        );

        During(Submitted,
            When(PaymentProcessed)
                .IfElse(context => context.Data.Success,
                    success => success
                        .TransitionTo(PaymentProcessed)
                        .Publish(context => new ReserveInventory { OrderId = context.Data.OrderId }),
                    failure => failure
                        .TransitionTo(Failed)) // Handle compensation here
        );

        During(PaymentProcessed,
            When(InventoryReserved)
                .IfElse(context => context.Data.Success,
                    success => success
                        .TransitionTo(InventoryReserved)
                        .Publish(context => new StartShipping { OrderId = context.Data.OrderId }),
                    failure => failure
                        .TransitionTo(Failed)) // Handle compensation here
        );

        During(InventoryReserved,
            When(ShippingStarted)
                .TransitionTo(ShippingStarted)
                .Finalize()
        );

        SetCompletedWhenFinalized();
    }
}
```

#### **5. Service Implementation**

For each service (Order, Payment, Inventory, Shipping), you handle the specific local business logic. For instance, the **PaymentService** listens to the `ProcessPayment` command and processes the payment.

#### **6. Running the Application**

- Run RabbitMQ (`docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management`).
- Run the microservices.
- The SAGA orchestrates the flow of the order process using messages over RabbitMQ.

---

### **Key Points**
- **MassTransit** handles the orchestration and state transitions.
- **RabbitMQ** acts as the message broker for services to communicate.
- Each service handles its part of the transaction and reports the result.
- If any step fails, the SAGA executes compensating transactions to roll back previous steps, ensuring eventual consistency.


### **Handling Compensation in a SAGA**

In a distributed system using the SAGA pattern, when a step in the workflow fails, compensating actions must be taken to undo any effects of the preceding steps. Compensation ensures eventual consistency by reverting successful steps when later steps fail.

For example, if an order is placed and the payment is processed successfully, but the inventory reservation fails, the system should:
1. Cancel the payment.
2. Cancel the order.

In the **MassTransit** implementation, this compensation logic is built into the state machine. Below, I’ll explain how to handle failures and trigger compensating actions when needed.

---

### **SAGA Compensation Scenario Example**

Let’s walk through an example in detail:

1. **OrderService**: Starts the process by placing an order.
2. **PaymentService**: Processes the payment.
3. **InventoryService**: Reserves inventory. If it fails, the payment must be refunded, and the order must be canceled.

If any of these steps fail, the SAGA orchestrator triggers compensating transactions.

---

### **1. Updated SAGA State Machine with Compensation**

The state machine will be modified to handle compensation logic. We’ll introduce a **Failed** state, and compensating commands like `CancelPayment` and `ReleaseInventory`.

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Submitted { get; private set; }
    public State PaymentProcessed { get; private set; }
    public State InventoryReserved { get; private set; }
    public State ShippingStarted { get; private set; }
    public State Failed { get; private set; } // New state for failure

    public Event<OrderSubmitted> OrderSubmitted { get; private set; }
    public Event<PaymentProcessed> PaymentProcessed { get; private set; }
    public Event<InventoryReserved> InventoryReserved { get; private set; }
    public Event<ShippingStarted> ShippingStarted { get; private set; }
    public Event<OrderFailed> OrderFailed { get; private set; } // New event for failure

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderSubmitted, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => ShippingStarted, x => x.CorrelateById(context => context.Message.OrderId));
        Event(() => OrderFailed, x => x.CorrelateById(context => context.Message.OrderId));

        Initially(
            When(OrderSubmitted)
                .Then(context => context.Instance.OrderId = context.Data.OrderId)
                .TransitionTo(Submitted)
                .Publish(context => new ProcessPayment
                {
                    OrderId = context.Data.OrderId,
                    Amount = context.Data.Amount
                })
        );

        During(Submitted,
            When(PaymentProcessed)
                .IfElse(context => context.Data.Success,
                    success => success
                        .TransitionTo(PaymentProcessed)
                        .Publish(context => new ReserveInventory { OrderId = context.Data.OrderId }),
                    failure => failure
                        .Publish(context => new CancelPayment { OrderId = context.Instance.OrderId }) // Compensation
                        .TransitionTo(Failed)) // Transition to failure state
        );

        During(PaymentProcessed,
            When(InventoryReserved)
                .IfElse(context => context.Data.Success,
                    success => success
                        .TransitionTo(InventoryReserved)
                        .Publish(context => new StartShipping { OrderId = context.Data.OrderId }),
                    failure => failure
                        .Publish(context => new CancelPayment { OrderId = context.Instance.OrderId }) // Compensation
                        .Publish(context => new ReleaseInventory { OrderId = context.Instance.OrderId }) // Compensation
                        .TransitionTo(Failed))
        );

        During(InventoryReserved,
            When(ShippingStarted)
                .TransitionTo(ShippingStarted)
                .Finalize()
        );

        SetCompletedWhenFinalized();
    }
}
```

### **2. Compensating Commands and Handlers**

We need to define the compensating commands and their respective service handlers to rollback successful actions.

#### **Compensating Commands**

Define commands for compensation actions, such as canceling payment or releasing inventory.

```csharp
public interface CancelPayment
{
    Guid OrderId { get; }
}

public interface ReleaseInventory
{
    Guid OrderId { get; }
}
```

#### **PaymentService - Handle Payment Cancellation**

In the `PaymentService`, we handle the cancellation of payment when the inventory reservation fails.

```csharp
public class CancelPaymentConsumer : IConsumer<CancelPayment>
{
    public async Task Consume(ConsumeContext<CancelPayment> context)
    {
        var orderId = context.Message.OrderId;
        
        // Logic to cancel the payment
        Console.WriteLine($"Payment for Order {orderId} has been canceled.");

        await Task.CompletedTask;
    }
}
```

#### **InventoryService - Handle Inventory Release**

In the `InventoryService`, we handle the release of reserved inventory if something goes wrong during the process.

```csharp
public class ReleaseInventoryConsumer : IConsumer<ReleaseInventory>
{
    public async Task Consume(ConsumeContext<ReleaseInventory> context)
    {
        var orderId = context.Message.OrderId;
        
        // Logic to release the reserved inventory
        Console.WriteLine($"Inventory for Order {orderId} has been released.");

        await Task.CompletedTask;
    }
}
```

### **3. Updating the Services to Handle Compensation**

- **PaymentService**: Now handles both `ProcessPayment` and `CancelPayment` messages.
- **InventoryService**: Handles both `ReserveInventory` and `ReleaseInventory` messages.

#### **PaymentService**

```csharp
public class PaymentService
{
    public class ProcessPaymentConsumer : IConsumer<ProcessPayment>
    {
        public async Task Consume(ConsumeContext<ProcessPayment> context)
        {
            var orderId = context.Message.OrderId;
            var success = true; // Assume payment is successful
            // Logic for processing payment

            // Return the result of payment processing
            await context.Publish<PaymentProcessed>(new
            {
                OrderId = orderId,
                Success = success
            });
        }
    }

    public class CancelPaymentConsumer : IConsumer<CancelPayment>
    {
        public async Task Consume(ConsumeContext<CancelPayment> context)
        {
            var orderId = context.Message.OrderId;
            
            // Logic to cancel the payment
            Console.WriteLine($"Payment for Order {orderId} has been canceled.");

            await Task.CompletedTask;
        }
    }
}
```

#### **InventoryService**

```csharp
public class InventoryService
{
    public class ReserveInventoryConsumer : IConsumer<ReserveInventory>
    {
        public async Task Consume(ConsumeContext<ReserveInventory> context)
        {
            var orderId = context.Message.OrderId;
            var success = false; // Simulate inventory failure
            
            // Logic for reserving inventory
            
            await context.Publish<InventoryReserved>(new
            {
                OrderId = orderId,
                Success = success
            });
        }
    }

    public class ReleaseInventoryConsumer : IConsumer<ReleaseInventory>
    {
        public async Task Consume(ConsumeContext<ReleaseInventory> context)
        {
            var orderId = context.Message.OrderId;
            
            // Logic to release the reserved inventory
            Console.WriteLine($"Inventory for Order {orderId} has been released.");

            await Task.CompletedTask;
        }
    }
}
```

### **4. Test Failure Scenarios**

Let’s simulate a failure in the inventory reservation:

1. An order is placed.
2. The payment is processed successfully.
3. The inventory reservation fails.
4. The system triggers the compensating actions:
   - Payment is canceled.
   - Reserved inventory is released.
5. The SAGA transitions to the **Failed** state, and the order is not processed further.

---

### **Summary**

In this extended implementation, we:
- Defined a failure state in the SAGA state machine (`Failed`).
- Added compensating commands (`CancelPayment`, `ReleaseInventory`).
- Implemented handlers in the respective services to undo successful steps if later steps fail.
- Updated the state machine to handle compensation when a service fails during the workflow.

This approach ensures that the system maintains eventual consistency by properly rolling back actions when part of the transaction fails.