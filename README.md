# 🏗️ Clean Architecture Microservices — Java 21 / Spring Boot 4

Construire une application en microservices, c'est facile. Construire une
application en microservices qui reste maintenable, testable et évolutive
six mois plus tard — c'est une autre histoire. Ce projet applique la Clean
Architecture à une architecture microservices complète, avec Spring Boot 4,
Kafka, Eureka et une observabilité complète (Grafana/Loki/Tempo).

---

## Architecture globale

```
  Client (API Gateway)
          │
          ▼
  ┌───────────────────────────────────────────────────────────┐
  │                    API Gateway (Spring Cloud)             │
  │         Routing + Auth JWT + Rate Limiting                │
  └───┬──────────────┬───────────────┬───────────────────────┘
      │              │               │
      ▼              ▼               ▼
  ┌────────┐   ┌──────────┐   ┌───────────┐
  │ Order  │   │Inventory │   │Notification│
  │Service │   │Service   │   │Service    │
  │        │   │          │   │           │
  │Port:8081│  │Port:8082 │   │Port:8083  │
  └────┬───┘   └────┬─────┘   └─────┬─────┘
       │             │               │
       └─────────────┼───────────────┘
                     │  Kafka Topics
                     ▼
             ┌───────────────┐
             │ Apache Kafka  │
             │ order-placed  │
             │ order-updated │
             └───────────────┘

  Service Discovery : Eureka Server (tous les services s'y enregistrent)
  Config centralisé : Spring Cloud Config Server
  Observabilité     : Grafana + Loki (logs) + Tempo (traces)
```

---

## Structure Clean Architecture par service

```
  order-service/
  └── src/main/java/
      └── com/viseo/order/
          │
          ├── domain/                  ← CORE — zéro dépendance externe
          │   ├── model/
          │   │   └── Order.java       ← entité domain pure
          │   ├── port/
          │   │   ├── in/              ← interfaces use cases (driven)
          │   │   │   └── PlaceOrderUseCase.java
          │   │   └── out/             ← interfaces infrastructure (driver)
          │   │       ├── OrderRepository.java
          │   │       └── OrderEventPublisher.java
          │   └── exception/
          │       └── OrderNotFoundException.java
          │
          ├── application/             ← USE CASES — orchestration métier
          │   └── service/
          │       └── OrderService.java  ← implémente PlaceOrderUseCase
          │
          └── infrastructure/          ← ADAPTERS — détails techniques
              ├── web/
              │   └── OrderController.java   ← entrée HTTP
              ├── persistence/
              │   └── JpaOrderRepository.java ← implémente OrderRepository
              └── messaging/
                  └── KafkaOrderPublisher.java ← implémente OrderEventPublisher
```

---

## Entité Domain — pure, sans annotation Spring

```java
// domain/model/Order.java
package com.viseo.order.domain.model;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

public final class Order {

    private final OrderId    id;
    private final CustomerId customerId;
    private OrderStatus      status;
    private final BigDecimal amount;
    private final Instant    createdAt;

    private Order(OrderId id, CustomerId customerId, BigDecimal amount) {
        this.id         = id;
        this.customerId = customerId;
        this.amount     = amount;
        this.status     = OrderStatus.PENDING;
        this.createdAt  = Instant.now();
    }

    // Factory method — validation centralisée
    public static Order create(CustomerId customerId, BigDecimal amount) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Order amount must be positive");
        }
        return new Order(OrderId.generate(), customerId, amount);
    }

    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Only PENDING orders can be confirmed");
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("Cannot cancel a shipped order");
        }
        this.status = OrderStatus.CANCELLED;
    }

    // Getters — pas de setters publics, état contrôlé par les méthodes métier
    public OrderId    getId()         { return id; }
    public OrderStatus getStatus()    { return status; }
    public BigDecimal getAmount()     { return amount; }
    public Instant    getCreatedAt()  { return createdAt; }
}
```

---

## Port (interface) — découplage total de l'infrastructure

```java
// domain/port/in/PlaceOrderUseCase.java
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// domain/port/out/OrderRepository.java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
}

// domain/port/out/OrderEventPublisher.java
public interface OrderEventPublisher {
    void publishOrderPlaced(OrderPlacedEvent event);
}
```

---

## Use Case — Application Service

```java
// application/service/OrderService.java
@Service
@Transactional
public class OrderService implements PlaceOrderUseCase {

    private final OrderRepository    repository;
    private final InventoryPort      inventory;
    private final OrderEventPublisher publisher;

    public OrderService(OrderRepository repository,
                        InventoryPort inventory,
                        OrderEventPublisher publisher) {
        this.repository = repository;
        this.inventory  = inventory;
        this.publisher  = publisher;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 1. Vérifier le stock
        boolean inStock = inventory.isInStock(
            command.getProductId(), command.getQuantity()
        );
        if (!inStock) {
            throw new OutOfStockException(command.getProductId());
        }

        // 2. Créer l'entité domain
        Order order = Order.create(
            command.getCustomerId(),
            command.getAmount()
        );

        // 3. Persister
        repository.save(order);

        // 4. Publier l'événement Kafka
        publisher.publishOrderPlaced(
            new OrderPlacedEvent(order.getId(), command.getProductId())
        );

        return order.getId();
    }
}
```

---

## Communication async via Kafka

```java
// infrastructure/messaging/KafkaOrderPublisher.java
@Component
public class KafkaOrderPublisher implements OrderEventPublisher {

    private static final String TOPIC = "order-placed";
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    @Override
    public void publishOrderPlaced(OrderPlacedEvent event) {
        kafkaTemplate.send(TOPIC, event.getOrderId().value(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event: {}", ex.getMessage());
                } else {
                    log.info("Event published: {} offset={}",
                        event.getOrderId(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}

// Notification Service consomme les événements
@KafkaListener(topics = "order-placed", groupId = "notification-service")
public void handleOrderPlaced(OrderPlacedEvent event) {
    log.info("Received order placed event: {}", event.getOrderId());
    emailService.sendOrderConfirmation(event);
}
```

---

## Tests unitaires — Use Case isolé

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository    repository;
    @Mock InventoryPort      inventory;
    @Mock OrderEventPublisher publisher;

    @InjectMocks OrderService orderService;

    @Test
    void placeOrder_shouldCreateAndPublishEvent_whenInStock() {
        // Arrange
        var command = new PlaceOrderCommand(
            CustomerId.of("cust-1"),
            ProductId.of("prod-1"),
            2,
            new BigDecimal("99.99")
        );
        when(inventory.isInStock(any(), anyInt())).thenReturn(true);

        // Act
        OrderId orderId = orderService.placeOrder(command);

        // Assert
        assertNotNull(orderId);
        verify(repository, times(1)).save(any(Order.class));
        verify(publisher, times(1)).publishOrderPlaced(any());
    }

    @Test
    void placeOrder_shouldThrowException_whenOutOfStock() {
        when(inventory.isInStock(any(), anyInt())).thenReturn(false);

        assertThrows(OutOfStockException.class,
            () -> orderService.placeOrder(buildCommand()));

        verify(repository, never()).save(any());
        verify(publisher, never()).publishOrderPlaced(any());
    }
}
```

---

## Ce que j'ai appris

Les ports (interfaces) dans la couche Domain ne sont pas du code en plus —
ce sont ce qui rend les tests rapides et fiables. Sans eux, tester
`OrderService` nécessite une vraie base de données et un vrai Kafka.
Avec eux, on mock tout en deux lignes et le test tourne en millisecondes.

C'est aussi ce qui permet de changer l'implémentation sans toucher
au métier : passer de PostgreSQL à MongoDB, de Kafka à RabbitMQ —
le Domain ne sait pas, ne s'en soucie pas.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
