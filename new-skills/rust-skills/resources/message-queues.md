# Message Queues - RabbitMQ (lapin) and Kafka (rdkafka)

Complete guide to integrating message queues in Rust backend services for async communication and event-driven architectures.

## Table of Contents

- [Message Queue Overview](#message-queue-overview)
- [RabbitMQ with lapin](#rabbitmq-with-lapin)
- [Apache Kafka with rdkafka](#apache-kafka-with-rdkafka)
- [Pattern Comparison](#pattern-comparison)
- [BMC Pattern Integration](#bmc-pattern-integration)
- [Error Handling and Retries](#error-handling-and-retries)
- [Message Serialization](#message-serialization)
- [Dead Letter Queues](#dead-letter-queues)
- [Monitoring and Observability](#monitoring-and-observability)
- [Testing Message Handlers](#testing-message-handlers)
- [Complete Examples](#complete-examples)

---

## Message Queue Overview

### When to Use Message Queues

**Use Cases:**
- ✅ **Async processing** - Offload time-consuming tasks
- ✅ **Event-driven architecture** - Decouple services
- ✅ **Load leveling** - Handle traffic spikes
- ✅ **Guaranteed delivery** - Persist messages until processed
- ✅ **Fan-out patterns** - Broadcast to multiple consumers

**RabbitMQ vs Kafka:**

| Feature | RabbitMQ (lapin) | Kafka (rdkafka) |
|---------|------------------|-----------------|
| **Use Case** | Task queues, RPC | Event streaming, logs |
| **Delivery** | At-most-once/at-least-once | At-least-once/exactly-once |
| **Ordering** | Per-queue | Per-partition |
| **Retention** | Until consumed | Time/size-based |
| **Complexity** | Lower | Higher |
| **Throughput** | 10K-50K msg/s | 100K-1M msg/s |

---

## RabbitMQ with lapin

### Dependencies

```toml
[dependencies]
# RabbitMQ client
lapin = "2.3"
futures-util = "0.3"

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Error handling
derive_more = { version = "1.0", features = ["from", "display"] }

# Tracing
tracing = "0.1"
```

### Connection Setup

```rust
// lib-infrastructure/src/rabbitmq/mod.rs

use lapin::{
    Connection, ConnectionProperties,
    Channel,
    options::*,
    types::FieldTable,
};
use std::sync::Arc;

#[derive(Clone)]
pub struct RabbitMqClient {
    connection: Arc<Connection>,
}

impl RabbitMqClient {
    pub async fn new(amqp_url: &str) -> Result<Self> {
        let connection = Connection::connect(
            amqp_url,
            ConnectionProperties::default(),
        )
        .await?;

        Ok(Self {
            connection: Arc::new(connection),
        })
    }

    pub async fn create_channel(&self) -> Result<Channel> {
        Ok(self.connection.create_channel().await?)
    }
}
```

### Producer Pattern

```rust
use lapin::{
    BasicProperties,
    options::BasicPublishOptions,
};
use serde::Serialize;

pub struct MessageProducer {
    channel: Channel,
}

impl MessageProducer {
    pub async fn new(client: &RabbitMqClient) -> Result<Self> {
        let channel = client.create_channel().await?;

        // Declare exchange
        channel
            .exchange_declare(
                "events",
                lapin::ExchangeKind::Topic,
                ExchangeDeclareOptions {
                    durable: true,
                    ..Default::default()
                },
                FieldTable::default(),
            )
            .await?;

        Ok(Self { channel })
    }

    pub async fn publish<T: Serialize>(
        &self,
        routing_key: &str,
        message: &T,
    ) -> Result<()> {
        let payload = serde_json::to_vec(message)?;

        self.channel
            .basic_publish(
                "events",
                routing_key,
                BasicPublishOptions::default(),
                &payload,
                BasicProperties::default()
                    .with_content_type("application/json".into())
                    .with_delivery_mode(2), // Persistent
            )
            .await?
            .await?; // Wait for confirmation

        Ok(())
    }
}

// Usage with BMC
impl UserBmc {
    pub async fn create_with_event(
        ctx: &Ctx,
        mm: &ModelManager,
        producer: &MessageProducer,
        data: UserForCreate,
    ) -> Result<i64> {
        // Create user
        let user_id = Self::create(ctx, mm, data).await?;

        // Publish event
        let event = UserCreatedEvent {
            user_id,
            username: data.username.clone(),
            timestamp: Utc::now(),
        };

        producer.publish("user.created", &event).await?;

        Ok(user_id)
    }
}
```

### Consumer Pattern

```rust
use lapin::{
    Consumer,
    options::{BasicConsumeOptions, BasicAckOptions, BasicNackOptions},
    message::Delivery,
};
use futures_util::StreamExt;
use serde::Deserialize;

pub struct MessageConsumer {
    channel: Channel,
}

impl MessageConsumer {
    pub async fn new(client: &RabbitMqClient, queue_name: &str) -> Result<Self> {
        let channel = client.create_channel().await?;

        // Declare queue
        channel
            .queue_declare(
                queue_name,
                QueueDeclareOptions {
                    durable: true,
                    ..Default::default()
                },
                FieldTable::default(),
            )
            .await?;

        // Bind to exchange
        channel
            .queue_bind(
                queue_name,
                "events",
                "user.*", // Routing pattern
                QueueBindOptions::default(),
                FieldTable::default(),
            )
            .await?;

        Ok(Self { channel })
    }

    pub async fn consume<T, F, Fut>(
        &self,
        queue_name: &str,
        handler: F,
    ) -> Result<()>
    where
        T: for<'de> Deserialize<'de>,
        F: Fn(T) -> Fut + Send + Sync + 'static,
        Fut: std::future::Future<Output = Result<()>> + Send,
    {
        let mut consumer = self
            .channel
            .basic_consume(
                queue_name,
                "consumer-tag",
                BasicConsumeOptions::default(),
                FieldTable::default(),
            )
            .await?;

        tracing::info!("🐰 Consuming from queue: {}", queue_name);

        while let Some(delivery) = consumer.next().await {
            match delivery {
                Ok(delivery) => {
                    if let Err(e) = self.process_message(&delivery, &handler).await {
                        tracing::error!("Failed to process message: {}", e);
                        // Nack with requeue
                        delivery
                            .nack(BasicNackOptions {
                                requeue: true,
                                multiple: false,
                            })
                            .await?;
                    } else {
                        // Ack
                        delivery.ack(BasicAckOptions::default()).await?;
                    }
                }
                Err(e) => {
                    tracing::error!("Consumer error: {}", e);
                }
            }
        }

        Ok(())
    }

    async fn process_message<T, F, Fut>(
        &self,
        delivery: &Delivery,
        handler: &F,
    ) -> Result<()>
    where
        T: for<'de> Deserialize<'de>,
        F: Fn(T) -> Fut,
        Fut: std::future::Future<Output = Result<()>>,
    {
        let message: T = serde_json::from_slice(&delivery.data)?;
        handler(message).await?;
        Ok(())
    }
}

// Usage
#[tokio::main]
async fn main() -> Result<()> {
    let client = RabbitMqClient::new("amqp://localhost").await?;
    let consumer = MessageConsumer::new(&client, "user-events").await?;

    consumer
        .consume("user-events", |event: UserCreatedEvent| async move {
            tracing::info!("User created: {:?}", event);
            // Process event with BMC
            send_welcome_email(&event).await?;
            Ok(())
        })
        .await?;

    Ok(())
}
```

### Work Queues Pattern

```rust
// Producer: Distribute tasks
pub async fn queue_task(producer: &MessageProducer, task: Task) -> Result<()> {
    producer.publish("tasks.process", &task).await?;
    Ok(())
}

// Consumer: Process tasks (multiple workers)
#[tokio::main]
async fn main() -> Result<()> {
    let client = RabbitMqClient::new("amqp://localhost").await?;
    let consumer = MessageConsumer::new(&client, "task-queue").await?;

    // Set QoS to process one at a time
    consumer.channel.basic_qos(1, BasicQosOptions::default()).await?;

    consumer
        .consume("task-queue", |task: Task| async move {
            process_task(task).await?;
            Ok(())
        })
        .await?;

    Ok(())
}
```

---

## Apache Kafka with rdkafka

### Dependencies

```toml
[dependencies]
# Kafka client
rdkafka = { version = "0.36", features = ["cmake-build", "ssl", "gssapi"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Tracing
tracing = "0.1"
```

### Producer Pattern

```rust
// lib-infrastructure/src/kafka/producer.rs

use rdkafka::{
    config::ClientConfig,
    producer::{FutureProducer, FutureRecord},
    util::Timeout,
};
use serde::Serialize;
use std::time::Duration;

pub struct KafkaProducer {
    producer: FutureProducer,
}

impl KafkaProducer {
    pub fn new(brokers: &str) -> Result<Self> {
        let producer: FutureProducer = ClientConfig::new()
            .set("bootstrap.servers", brokers)
            .set("message.timeout.ms", "5000")
            .set("compression.type", "snappy")
            .set("acks", "all") // Wait for all replicas
            .create()?;

        Ok(Self { producer })
    }

    pub async fn send<T: Serialize>(
        &self,
        topic: &str,
        key: Option<&str>,
        message: &T,
    ) -> Result<()> {
        let payload = serde_json::to_vec(message)?;

        let record = FutureRecord::to(topic)
            .payload(&payload)
            .key(key.unwrap_or(""));

        let delivery_status = self
            .producer
            .send(record, Timeout::After(Duration::from_secs(5)))
            .await;

        match delivery_status {
            Ok((partition, offset)) => {
                tracing::debug!("Sent to partition {} offset {}", partition, offset);
                Ok(())
            }
            Err((err, _)) => Err(err.into()),
        }
    }

    pub async fn send_batch<T: Serialize>(
        &self,
        topic: &str,
        messages: Vec<(Option<String>, T)>,
    ) -> Result<()> {
        for (key, message) in messages {
            self.send(topic, key.as_deref(), &message).await?;
        }
        Ok(())
    }
}

// Usage with BMC
impl UserBmc {
    pub async fn create_with_kafka(
        ctx: &Ctx,
        mm: &ModelManager,
        producer: &KafkaProducer,
        data: UserForCreate,
    ) -> Result<i64> {
        let user_id = Self::create(ctx, mm, data).await?;

        let event = UserCreatedEvent {
            user_id,
            username: data.username.clone(),
            timestamp: Utc::now(),
        };

        // Use user_id as partition key for ordering
        producer
            .send("user-events", Some(&user_id.to_string()), &event)
            .await?;

        Ok(user_id)
    }
}
```

### Consumer Pattern

```rust
// lib-infrastructure/src/kafka/consumer.rs

use rdkafka::{
    config::ClientConfig,
    consumer::{Consumer, StreamConsumer},
    Message,
    message::BorrowedMessage,
};
use futures_util::StreamExt;
use serde::Deserialize;

pub struct KafkaConsumer {
    consumer: StreamConsumer,
}

impl KafkaConsumer {
    pub fn new(brokers: &str, group_id: &str, topics: &[&str]) -> Result<Self> {
        let consumer: StreamConsumer = ClientConfig::new()
            .set("bootstrap.servers", brokers)
            .set("group.id", group_id)
            .set("enable.auto.commit", "true")
            .set("auto.offset.reset", "earliest")
            .set("session.timeout.ms", "6000")
            .create()?;

        consumer.subscribe(topics)?;

        Ok(Self { consumer })
    }

    pub async fn consume<T, F, Fut>(
        &self,
        handler: F,
    ) -> Result<()>
    where
        T: for<'de> Deserialize<'de>,
        F: Fn(T, Option<String>) -> Fut + Send + Sync,
        Fut: std::future::Future<Output = Result<()>> + Send,
    {
        tracing::info!("📨 Kafka consumer started");

        let mut stream = self.consumer.stream();

        while let Some(message) = stream.next().await {
            match message {
                Ok(msg) => {
                    if let Err(e) = self.process_message(&msg, &handler).await {
                        tracing::error!("Failed to process message: {}", e);
                    }
                }
                Err(e) => {
                    tracing::error!("Kafka error: {}", e);
                }
            }
        }

        Ok(())
    }

    async fn process_message<T, F, Fut>(
        &self,
        msg: &BorrowedMessage<'_>,
        handler: &F,
    ) -> Result<()>
    where
        T: for<'de> Deserialize<'de>,
        F: Fn(T, Option<String>) -> Fut,
        Fut: std::future::Future<Output = Result<()>>,
    {
        let payload = msg.payload().ok_or("Empty payload")?;
        let key = msg.key().map(|k| String::from_utf8_lossy(k).to_string());

        let message: T = serde_json::from_slice(payload)?;

        handler(message, key).await?;

        Ok(())
    }
}

// Usage
#[tokio::main]
async fn main() -> Result<()> {
    let consumer = KafkaConsumer::new(
        "localhost:9092",
        "user-service-group",
        &["user-events"],
    )?;

    consumer
        .consume(|event: UserCreatedEvent, key: Option<String>| async move {
            tracing::info!("User created: {:?} (key: {:?})", event, key);
            send_welcome_email(&event).await?;
            Ok(())
        })
        .await?;

    Ok(())
}
```

### Event Sourcing Pattern

```rust
use rdkafka::producer::FutureProducer;

#[derive(Serialize, Deserialize)]
pub enum UserEvent {
    Created { user_id: i64, username: String },
    Updated { user_id: i64, changes: HashMap<String, String> },
    Deleted { user_id: i64 },
}

pub struct EventStore {
    producer: FutureProducer,
}

impl EventStore {
    pub async fn append_event(&self, user_id: i64, event: UserEvent) -> Result<()> {
        let key = user_id.to_string();
        let payload = serde_json::to_vec(&event)?;

        let record = FutureRecord::to("user-event-store")
            .key(&key)
            .payload(&payload);

        self.producer.send(record, Timeout::Never).await?;

        Ok(())
    }

    pub async fn replay_events(&self, user_id: i64) -> Result<Vec<UserEvent>> {
        // Use Kafka consumer to read all events for this user
        // and rebuild state
        todo!()
    }
}
```

---

## Pattern Comparison

### Task Queue (RabbitMQ)

```rust
// Producer
producer.publish("tasks.email", &SendEmailTask {
    to: "user@example.com",
    subject: "Welcome",
}).await?;

// Consumer (worker)
consumer.consume("email-queue", |task: SendEmailTask| async move {
    send_email(&task).await?;
    Ok(())
}).await?;
```

### Event Stream (Kafka)

```rust
// Producer
producer.send("user-events", Some(&user_id.to_string()), &UserEvent::Created {
    user_id,
    username,
}).await?;

// Consumer (subscriber)
consumer.consume(|event: UserEvent, key| async move {
    match event {
        UserEvent::Created { user_id, username } => {
            // Handle creation
        }
        UserEvent::Updated { user_id, changes } => {
            // Handle update
        }
        _ => {}
    }
    Ok(())
}).await?;
```

---

## BMC Pattern Integration

### Domain Events

```rust
// lib-core/src/events.rs

use serde::{Serialize, Deserialize};
use chrono::{DateTime, Utc};

#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum DomainEvent {
    UserCreated(UserCreatedEvent),
    UserUpdated(UserUpdatedEvent),
    UserDeleted(UserDeletedEvent),
}

#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct UserCreatedEvent {
    pub user_id: i64,
    pub username: String,
    pub email: String,
    pub timestamp: DateTime<Utc>,
}

// BMC with event publishing
impl UserBmc {
    pub async fn create_and_publish(
        ctx: &Ctx,
        mm: &ModelManager,
        event_bus: &impl EventPublisher,
        data: UserForCreate,
    ) -> Result<i64> {
        // Create user
        let user_id = Self::create(ctx, mm, data.clone()).await?;

        // Publish event
        let event = DomainEvent::UserCreated(UserCreatedEvent {
            user_id,
            username: data.username,
            email: data.email,
            timestamp: Utc::now(),
        });

        event_bus.publish(event).await?;

        Ok(user_id)
    }
}

// Event publisher trait
#[async_trait::async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish(&self, event: DomainEvent) -> Result<()>;
}

// RabbitMQ implementation
#[async_trait::async_trait]
impl EventPublisher for MessageProducer {
    async fn publish(&self, event: DomainEvent) -> Result<()> {
        let routing_key = match &event {
            DomainEvent::UserCreated(_) => "user.created",
            DomainEvent::UserUpdated(_) => "user.updated",
            DomainEvent::UserDeleted(_) => "user.deleted",
        };

        self.publish(routing_key, &event).await
    }
}

// Kafka implementation
#[async_trait::async_trait]
impl EventPublisher for KafkaProducer {
    async fn publish(&self, event: DomainEvent) -> Result<()> {
        let key = match &event {
            DomainEvent::UserCreated(e) => e.user_id.to_string(),
            DomainEvent::UserUpdated(e) => e.user_id.to_string(),
            DomainEvent::UserDeleted(e) => e.user_id.to_string(),
        };

        self.send("domain-events", Some(&key), &event).await
    }
}
```

---

## Error Handling and Retries

### Retry Logic

```rust
use tokio::time::{sleep, Duration};

pub async fn with_retry<F, Fut, T>(
    max_retries: u32,
    delay: Duration,
    operation: F,
) -> Result<T>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T>>,
{
    let mut attempts = 0;

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempts < max_retries => {
                attempts += 1;
                tracing::warn!("Retry {}/{}: {}", attempts, max_retries, e);
                sleep(delay * attempts).await;
            }
            Err(e) => return Err(e),
        }
    }
}

// Usage
consumer.consume("user-events", |event: UserEvent| async move {
    with_retry(3, Duration::from_secs(1), || async {
        process_event(&event).await
    }).await?;
    Ok(())
}).await?;
```

---

## Message Serialization

### Multiple Formats

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct Message<T> {
    pub id: String,
    pub timestamp: DateTime<Utc>,
    pub payload: T,
}

// JSON (default)
let json = serde_json::to_vec(&message)?;

// MessagePack (binary, smaller)
let msgpack = rmp_serde::to_vec(&message)?;

// Protobuf (with prost)
let proto = message.encode_to_vec();
```

---

## Dead Letter Queues

### RabbitMQ DLQ

```rust
// Declare main queue with DLQ
let mut args = FieldTable::default();
args.insert("x-dead-letter-exchange".into(), "dlx".into());
args.insert("x-dead-letter-routing-key".into(), "failed".into());

channel
    .queue_declare("main-queue", QueueDeclareOptions::default(), args)
    .await?;

// Declare DLQ
channel
    .queue_declare("dlq", QueueDeclareOptions::default(), FieldTable::default())
    .await?;
```

### Kafka DLQ Pattern

```rust
async fn process_with_dlq(
    consumer: &KafkaConsumer,
    dlq_producer: &KafkaProducer,
) -> Result<()> {
    consumer
        .consume(|event: UserEvent, key| async move {
            if let Err(e) = process_event(&event).await {
                // Send to DLQ
                dlq_producer.send("user-events-dlq", key.as_deref(), &event).await?;
                tracing::error!("Sent to DLQ: {}", e);
            }
            Ok(())
        })
        .await
}
```

---

## Monitoring and Observability

### Metrics

```rust
use prometheus::{IntCounter, Histogram};

lazy_static! {
    static ref MESSAGES_PROCESSED: IntCounter =
        IntCounter::new("messages_processed_total", "Total messages processed").unwrap();

    static ref MESSAGE_PROCESSING_TIME: Histogram =
        Histogram::new("message_processing_seconds", "Message processing time").unwrap();
}

async fn process_with_metrics<T>(message: T) -> Result<()> {
    let timer = MESSAGE_PROCESSING_TIME.start_timer();

    process_message(message).await?;

    timer.observe_duration();
    MESSAGES_PROCESSED.inc();

    Ok(())
}
```

---

## Testing Message Handlers

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn test_event_handler() {
        let event = UserCreatedEvent {
            user_id: 1,
            username: "test".to_string(),
            email: "test@example.com".to_string(),
            timestamp: Utc::now(),
        };

        let result = handle_user_created(event).await;
        assert!(result.is_ok());
    }
}
```

---

## Complete Examples

See implementation in:
- [complete-examples.md](complete-examples.md) - Full message queue integration

---

## Related Files

- [SKILL.md](../SKILL.md) - Main Rust backend guidelines
- [async-patterns.md](async-patterns.md) - Async/await patterns
- [error-handling.md](error-handling.md) - Error handling strategies

---

**Pattern Status**: Production-ready message queue integration ✅
