# gRPC with Tonic - Rust Backend Patterns

Complete guide to building high-performance gRPC services in Rust using Tonic with Rust10x patterns.

## Table of Contents

- [Why gRPC in Rust](#why-grpc-in-rust)
- [Tonic Overview](#tonic-overview)
- [Protocol Buffers Setup](#protocol-buffers-setup)
- [Service Definition](#service-definition)
- [Server Implementation](#server-implementation)
- [Client Implementation](#client-implementation)
- [Streaming (Unary, Server, Client, Bidirectional)](#streaming-unary-server-client-bidirectional)
- [BMC Pattern Integration](#bmc-pattern-integration)
- [Interceptors and Middleware](#interceptors-and-middleware)
- [Authentication and Authorization](#authentication-and-authorization)
- [Error Handling](#error-handling)
- [Load Balancing and Service Discovery](#load-balancing-and-service-discovery)
- [Testing gRPC Services](#testing-grpc-services)
- [Complete Example](#complete-example)

---

## Why gRPC in Rust

**Advantages of Tonic:**
- ✅ **Performance** - Binary protocol (Protocol Buffers) + HTTP/2
- ✅ **Type safety** - Strong typing with code generation
- ✅ **Streaming** - Native support for bidirectional streaming
- ✅ **Async/await** - Built on Tokio, async by default
- ✅ **Interoperability** - Works with gRPC clients in any language

**vs REST:**
- 5-10x smaller payloads (Protobuf vs JSON)
- Lower latency (HTTP/2 multiplexing)
- Streaming support out of the box
- Strong schema enforcement

---

## Tonic Overview

### Dependencies

```toml
[dependencies]
# gRPC
tonic = "0.12"
prost = "0.13"

# Build dependencies
[build-dependencies]
tonic-build = "0.12"

# Optional: Reflection
tonic-reflection = "0.12"

# Optional: Health checking
tonic-health = "0.12"

# Async runtime
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"

# Error handling
derive_more = { version = "1.0", features = ["from", "display"] }
```

### Project Structure

```
crates/
├── libs/
│   ├── lib-core/           # Domain logic (BMC)
│   └── lib-grpc/           # gRPC layer
│       ├── proto/
│       │   └── user.proto  # Protocol buffer definitions
│       ├── src/
│       │   ├── generated/  # Auto-generated code
│       │   └── services/   # Service implementations
│       └── build.rs        # Protobuf compilation
└── services/
    └── grpc-server/        # Main gRPC service
        └── src/
            └── main.rs
```

---

## Protocol Buffers Setup

### build.rs

```rust
// lib-grpc/build.rs

fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .build_server(true)
        .build_client(true)
        .out_dir("src/generated")
        .compile(
            &[
                "proto/user.proto",
                "proto/post.proto",
            ],
            &["proto"],
        )?;
    Ok(())
}
```

### Proto Definitions

```protobuf
// lib-grpc/proto/user.proto

syntax = "proto3";

package user.v1;

// User service
service UserService {
  // Unary RPC
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming
  rpc ListUsers (ListUsersRequest) returns (stream UserMessage);

  // Client streaming
  rpc BatchCreateUsers (stream CreateUserRequest) returns (BatchCreateUsersResponse);

  // Bidirectional streaming
  rpc SyncUsers (stream UserSyncRequest) returns (stream UserSyncResponse);
}

// Messages
message User {
  int64 id = 1;
  string username = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
}

message CreateUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  int64 id = 1;
  optional string username = 2;
  optional string email = 3;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  int64 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  int32 page_token = 2;
}

message UserMessage {
  User user = 1;
}

message BatchCreateUsersResponse {
  repeated User users = 1;
  int32 created_count = 2;
}

message UserSyncRequest {
  oneof action {
    User create = 1;
    User update = 2;
    int64 delete = 3;
  }
}

message UserSyncResponse {
  bool success = 1;
  string message = 2;
}
```

---

## Service Definition

### Generated Code Integration

```rust
// lib-grpc/src/lib.rs

pub mod generated {
    tonic::include_proto!("user.v1");
}

pub use generated::{
    user_service_server::{UserService, UserServiceServer},
    user_service_client::UserServiceClient,
    *,
};
```

---

## Server Implementation

### Service Implementation

```rust
// lib-grpc/src/services/user_service.rs

use tonic::{Request, Response, Status};
use tokio_stream::wrappers::ReceiverStream;
use crate::generated::*;
use lib_core::{
    ctx::Ctx,
    model::{ModelManager, UserBmc, UserForCreate, UserForUpdate},
};

pub struct UserServiceImpl {
    mm: ModelManager,
}

impl UserServiceImpl {
    pub fn new(mm: ModelManager) -> Self {
        Self { mm }
    }
}

#[tonic::async_trait]
impl UserService for UserServiceImpl {
    // Unary RPC
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserResponse>, Status> {
        // Extract context from metadata
        let ctx = extract_ctx(&request)?;

        let req = request.into_inner();

        // Use BMC pattern
        let user = UserBmc::get(&ctx, &self.mm, req.id)
            .await
            .map_err(|e| Status::not_found(format!("User not found: {}", e)))?;

        // Convert domain model to proto
        let user_proto = User {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.created_at.timestamp(),
        };

        Ok(Response::new(GetUserResponse {
            user: Some(user_proto),
        }))
    }

    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<CreateUserResponse>, Status> {
        let ctx = extract_ctx(&request)?;
        let req = request.into_inner();

        let user_id = UserBmc::create(
            &ctx,
            &self.mm,
            UserForCreate {
                username: req.username,
                email: req.email,
                password_clear: req.password,
            },
        )
        .await
        .map_err(|e| Status::internal(format!("Failed to create user: {}", e)))?;

        let user = UserBmc::get(&ctx, &self.mm, user_id)
            .await
            .map_err(|e| Status::internal(format!("Failed to fetch created user: {}", e)))?;

        let user_proto = User {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.created_at.timestamp(),
        };

        Ok(Response::new(CreateUserResponse {
            user: Some(user_proto),
        }))
    }

    // Server streaming
    type ListUsersStream = ReceiverStream<Result<UserMessage, Status>>;

    async fn list_users(
        &self,
        request: Request<ListUsersRequest>,
    ) -> Result<Response<Self::ListUsersStream>, Status> {
        let ctx = extract_ctx(&request)?;
        let req = request.into_inner();

        let (tx, rx) = tokio::sync::mpsc::channel(100);

        let mm = self.mm.clone();

        tokio::spawn(async move {
            let users = UserBmc::list(&ctx, &mm)
                .await
                .map_err(|e| Status::internal(format!("Failed to list users: {}", e)))?;

            for user in users {
                let user_proto = User {
                    id: user.id,
                    username: user.username,
                    email: user.email,
                    created_at: user.created_at.timestamp(),
                };

                let message = UserMessage {
                    user: Some(user_proto),
                };

                if tx.send(Ok(message)).await.is_err() {
                    break;
                }
            }

            Ok::<_, Status>(())
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }

    // Client streaming
    async fn batch_create_users(
        &self,
        request: Request<tonic::Streaming<CreateUserRequest>>,
    ) -> Result<Response<BatchCreateUsersResponse>, Status> {
        let ctx = extract_ctx(&request)?;
        let mut stream = request.into_inner();

        let mut created_users = Vec::new();

        while let Some(req) = stream.message().await? {
            let user_id = UserBmc::create(
                &ctx,
                &self.mm,
                UserForCreate {
                    username: req.username,
                    email: req.email,
                    password_clear: req.password,
                },
            )
            .await
            .map_err(|e| Status::internal(format!("Failed to create user: {}", e)))?;

            let user = UserBmc::get(&ctx, &self.mm, user_id)
                .await
                .map_err(|e| Status::internal(format!("Failed to fetch user: {}", e)))?;

            created_users.push(User {
                id: user.id,
                username: user.username,
                email: user.email,
                created_at: user.created_at.timestamp(),
            });
        }

        Ok(Response::new(BatchCreateUsersResponse {
            users: created_users.clone(),
            created_count: created_users.len() as i32,
        }))
    }

    // Bidirectional streaming
    type SyncUsersStream = ReceiverStream<Result<UserSyncResponse, Status>>;

    async fn sync_users(
        &self,
        request: Request<tonic::Streaming<UserSyncRequest>>,
    ) -> Result<Response<Self::SyncUsersStream>, Status> {
        let ctx = extract_ctx(&request)?;
        let mut stream = request.into_inner();

        let (tx, rx) = tokio::sync::mpsc::channel(100);
        let mm = self.mm.clone();

        tokio::spawn(async move {
            while let Some(req) = stream.message().await? {
                let response = match req.action {
                    Some(user_sync_request::Action::Create(user)) => {
                        // Handle create
                        UserSyncResponse {
                            success: true,
                            message: format!("Created user {}", user.username),
                        }
                    }
                    Some(user_sync_request::Action::Update(user)) => {
                        // Handle update
                        UserSyncResponse {
                            success: true,
                            message: format!("Updated user {}", user.id),
                        }
                    }
                    Some(user_sync_request::Action::Delete(id)) => {
                        // Handle delete
                        UserSyncResponse {
                            success: true,
                            message: format!("Deleted user {}", id),
                        }
                    }
                    None => UserSyncResponse {
                        success: false,
                        message: "Invalid action".to_string(),
                    },
                };

                if tx.send(Ok(response)).await.is_err() {
                    break;
                }
            }

            Ok::<_, Status>(())
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }
}
```

### Server Setup

```rust
// grpc-server/src/main.rs

use tonic::transport::Server;
use lib_grpc::{UserServiceImpl, UserServiceServer};
use lib_core::model::ModelManager;

#[tokio::main]
async fn main() -> Result<()> {
    // Initialize database
    let mm = ModelManager::new().await?;

    // Create service
    let user_service = UserServiceImpl::new(mm);

    // Build server
    let addr = "[::1]:50051".parse()?;

    println!("🚀 gRPC server listening on {}", addr);

    Server::builder()
        .add_service(UserServiceServer::new(user_service))
        .serve(addr)
        .await?;

    Ok(())
}
```

---

## Client Implementation

### Basic Client

```rust
use lib_grpc::{UserServiceClient, GetUserRequest};

#[tokio::main]
async fn main() -> Result<()> {
    // Connect to server
    let mut client = UserServiceClient::connect("http://[::1]:50051").await?;

    // Make request
    let request = tonic::Request::new(GetUserRequest { id: 1 });

    let response = client.get_user(request).await?;

    println!("User: {:?}", response.into_inner().user);

    Ok(())
}
```

### Client with Authentication

```rust
use tonic::{Request, metadata::MetadataValue};

#[tokio::main]
async fn main() -> Result<()> {
    let mut client = UserServiceClient::connect("http://[::1]:50051").await?;

    // Create request with metadata
    let mut request = Request::new(GetUserRequest { id: 1 });

    // Add auth token
    let token = MetadataValue::try_from("Bearer my-token")?;
    request.metadata_mut().insert("authorization", token);

    let response = client.get_user(request).await?;

    Ok(())
}
```

### Streaming Client

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() -> Result<()> {
    let mut client = UserServiceClient::connect("http://[::1]:50051").await?;

    // Server streaming
    let request = Request::new(ListUsersRequest {
        page_size: 10,
        page_token: 0,
    });

    let mut stream = client.list_users(request).await?.into_inner();

    while let Some(user_msg) = stream.next().await {
        let user_msg = user_msg?;
        println!("User: {:?}", user_msg.user);
    }

    Ok(())
}
```

---

## BMC Pattern Integration

### Domain to Proto Conversion

```rust
// lib-grpc/src/conversions.rs

use lib_core::model::User as DomainUser;
use crate::generated::User as ProtoUser;

impl From<DomainUser> for ProtoUser {
    fn from(user: DomainUser) -> Self {
        Self {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.created_at.timestamp(),
        }
    }
}

impl TryFrom<ProtoUser> for DomainUser {
    type Error = String;

    fn try_from(proto: ProtoUser) -> Result<Self, Self::Error> {
        // Validation and conversion
        Ok(Self {
            id: proto.id,
            username: proto.username,
            email: proto.email,
            // ... additional fields
        })
    }
}
```

---

## Interceptors and Middleware

### Logging Interceptor

```rust
use tonic::{Request, Status};
use tonic::service::Interceptor;

#[derive(Clone)]
pub struct LoggingInterceptor;

impl Interceptor for LoggingInterceptor {
    fn call(&mut self, request: Request<()>) -> Result<Request<()>, Status> {
        println!("Received request: {:?}", request.metadata());
        Ok(request)
    }
}

// Use in server
Server::builder()
    .add_service(
        UserServiceServer::with_interceptor(user_service, LoggingInterceptor)
    )
    .serve(addr)
    .await?;
```

### Auth Interceptor

```rust
use tonic::metadata::MetadataMap;

#[derive(Clone)]
pub struct AuthInterceptor {
    required_token: String,
}

impl Interceptor for AuthInterceptor {
    fn call(&mut self, request: Request<()>) -> Result<Request<()>, Status> {
        match request.metadata().get("authorization") {
            Some(token) if token == self.required_token.as_str() => Ok(request),
            _ => Err(Status::unauthenticated("Invalid token")),
        }
    }
}
```

---

## Authentication and Authorization

### Context Extraction

```rust
use tonic::{Request, Status, metadata::MetadataMap};
use lib_core::ctx::Ctx;

pub fn extract_ctx<T>(request: &Request<T>) -> Result<Ctx, Status> {
    let metadata = request.metadata();

    // Extract user_id from token
    let token = metadata
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or_else(|| Status::unauthenticated("Missing token"))?;

    let user_id = validate_token(token)
        .map_err(|_| Status::unauthenticated("Invalid token"))?;

    Ok(Ctx { user_id })
}

fn validate_token(token: &str) -> Result<i64, String> {
    // Token validation logic
    // Return user_id
    Ok(1)
}
```

---

## Error Handling

### Custom Error Mapping

```rust
use tonic::Status;
use lib_core::model::Error as ModelError;

impl From<ModelError> for Status {
    fn from(err: ModelError) -> Self {
        match err {
            ModelError::EntityNotFound { entity, id } => {
                Status::not_found(format!("{} {} not found", entity, id))
            }
            ModelError::Unauthorized => {
                Status::unauthenticated("Unauthorized")
            }
            ModelError::Validation { message } => {
                Status::invalid_argument(message)
            }
            _ => Status::internal("Internal error"),
        }
    }
}
```

---

## Load Balancing and Service Discovery

### Client-Side Load Balancing

```rust
use tonic::transport::{Channel, Endpoint};

#[tokio::main]
async fn main() -> Result<()> {
    let channel = Channel::from_static("http://[::1]:50051")
        .connect()
        .await?;

    let mut client = UserServiceClient::new(channel);

    // Use client...
    Ok(())
}
```

---

## Testing gRPC Services

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_get_user() {
        let mm = create_test_model_manager().await;
        let service = UserServiceImpl::new(mm);

        let request = Request::new(GetUserRequest { id: 1 });

        let response = service.get_user(request).await.unwrap();
        let user = response.into_inner().user.unwrap();

        assert_eq!(user.id, 1);
        assert_eq!(user.username, "testuser");
    }
}
```

---

## Complete Example

See [complete-examples.md](complete-examples.md) for full gRPC service with authentication.

---

## Related Files

- [SKILL.md](../SKILL.md) - Main Rust backend guidelines
- [api-design.md](api-design.md) - REST API patterns
- [graphql-async-graphql.md](graphql-async-graphql.md) - GraphQL patterns
- [async-patterns.md](async-patterns.md) - Async/await best practices

---

**Pattern Status**: Production-ready gRPC with Rust10x integration ✅
