## 1. What is gRPC?

gRPC is a high-performance RPC (Remote Procedure Call) framework that uses HTTP/2 and Protocol Buffers for efficient service-to-service communication.

```
REST API:
  Protocol: HTTP/1.1
  Format: JSON (text)
  Request: POST /api/users {"name": "Alice"}
  Response: {"id": 123, "name": "Alice"}
  
  Human-readable, slower

gRPC:
  Protocol: HTTP/2
  Format: Protocol Buffers (binary)
  Request: CreateUser(name="Alice")
  Response: User(id=123, name="Alice")
  
  Binary, faster, strongly typed
```

**Key principle:** Fast, type-safe service-to-service communication.

---

## 2. Core Concepts

### Protocol Buffers (Protobuf)

Binary serialization format with schema definition.

```protobuf
// user.proto
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}
```

**Benefits:**
- Strongly typed (compile-time validation)
- Compact (binary, smaller than JSON)
- Fast (efficient serialization)
- Language-agnostic (generate code for any language)

### Service Definition

Define RPC methods in .proto file.

```protobuf
service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}
```

### HTTP/2

Modern protocol with multiplexing and streaming.

```
HTTP/1.1:
  - One request per connection
  - Text-based headers
  - No server push

HTTP/2:
  - Multiple requests per connection (multiplexing)
  - Binary headers (compressed)
  - Server push
  - Bidirectional streaming
```

---

## 3. gRPC Communication Patterns

### Unary RPC (Request-Response)

Single request, single response.

```protobuf
rpc GetUser(GetUserRequest) returns (GetUserResponse);
```

```
Client → Request → Server
Client ← Response ← Server

Like REST API
```

### Server Streaming

Single request, stream of responses.

```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```
Client → Request → Server
Client ← Response 1 ← Server
Client ← Response 2 ← Server
Client ← Response 3 ← Server
...

Use case: Large result sets, real-time updates
```

### Client Streaming

Stream of requests, single response.

```protobuf
rpc UploadFile(stream FileChunk) returns (UploadResponse);
```

```
Client → Request 1 → Server
Client → Request 2 → Server
Client → Request 3 → Server
...
Client ← Response ← Server

Use case: File upload, batch processing
```

### Bidirectional Streaming

Stream of requests, stream of responses.

```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

```
Client → Request 1 → Server
Client ← Response 1 ← Server
Client → Request 2 → Server
Client ← Response 2 ← Server
...

Use case: Chat, real-time collaboration
```

---

## 4. Implementation Example

### Define Service (.proto)

```protobuf
syntax = "proto3";

package user;

service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  int32 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}
```

### Generate Code

```bash
# Python
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user.proto

# Go
protoc --go_out=. --go-grpc_out=. user.proto

# Java
protoc --java_out=. --grpc-java_out=. user.proto
```

### Server Implementation (Python)

```python
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserService(user_pb2_grpc.UserServiceServicer):
    def __init__(self):
        self.users = {}
        self.next_id = 1
    
    def CreateUser(self, request, context):
        user = user_pb2.User(
            id=self.next_id,
            name=request.name,
            email=request.email
        )
        self.users[self.next_id] = user
        self.next_id += 1
        
        return user_pb2.CreateUserResponse(user=user)
    
    def GetUser(self, request, context):
        user = self.users.get(request.id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details('User not found')
            return user_pb2.GetUserResponse()
        
        return user_pb2.GetUserResponse(user=user)
    
    def ListUsers(self, request, context):
        # Server streaming
        for user in self.users.values():
            yield user

# Start server
server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
user_pb2_grpc.add_UserServiceServicer_to_server(UserService(), server)
server.add_insecure_port('[::]:50051')
server.start()
server.wait_for_termination()
```

### Client Implementation (Python)

```python
import grpc
import user_pb2
import user_pb2_grpc

# Create channel
channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)

# Unary RPC
response = stub.CreateUser(user_pb2.CreateUserRequest(
    name='Alice',
    email='alice@example.com'
))
print(f"Created user: {response.user.id}, {response.user.name}")

# Server streaming
for user in stub.ListUsers(user_pb2.ListUsersRequest()):
    print(f"User: {user.id}, {user.name}")
```

---

## 5. gRPC vs REST

### Performance

```
gRPC:
  - Binary (Protocol Buffers)
  - HTTP/2 (multiplexing)
  - ~7x faster than REST
  - ~10x smaller payload

REST:
  - Text (JSON)
  - HTTP/1.1
  - Human-readable
  - Larger payload
```

### Type Safety

```
gRPC:
  - Strongly typed (protobuf schema)
  - Compile-time validation
  - Auto-generated client code

REST:
  - Loosely typed (JSON)
  - Runtime validation
  - Manual client code
```

### Streaming

```
gRPC:
  - Native streaming support
  - Bidirectional streaming
  - HTTP/2

REST:
  - No native streaming
  - Use WebSockets or SSE
  - HTTP/1.1
```

### Browser Support

```
gRPC:
  - Limited browser support
  - Requires gRPC-Web proxy
  - Not ideal for web clients

REST:
  - Full browser support
  - Standard HTTP
  - Ideal for web clients
```

---

## 6. Error Handling

### Status Codes

```
gRPC status codes:
  OK (0): Success
  CANCELLED (1): Operation cancelled
  UNKNOWN (2): Unknown error
  INVALID_ARGUMENT (3): Invalid argument
  DEADLINE_EXCEEDED (4): Timeout
  NOT_FOUND (5): Not found
  ALREADY_EXISTS (6): Already exists
  PERMISSION_DENIED (7): Permission denied
  UNAUTHENTICATED (16): Not authenticated
  INTERNAL (13): Internal error
  UNAVAILABLE (14): Service unavailable
```

### Server-Side Error

```python
def GetUser(self, request, context):
    user = self.users.get(request.id)
    if not user:
        context.set_code(grpc.StatusCode.NOT_FOUND)
        context.set_details(f'User {request.id} not found')
        return user_pb2.GetUserResponse()
    
    return user_pb2.GetUserResponse(user=user)
```

### Client-Side Error Handling

```python
try:
    response = stub.GetUser(user_pb2.GetUserRequest(id=999))
    print(response.user)
except grpc.RpcError as e:
    print(f"Error: {e.code()}, {e.details()}")
    
    if e.code() == grpc.StatusCode.NOT_FOUND:
        print("User not found")
    elif e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Request timeout")
```

---

## 7. Advanced Features

### Metadata (Headers)

```python
# Server: Read metadata
def GetUser(self, request, context):
    metadata = dict(context.invocation_metadata())
    auth_token = metadata.get('authorization')
    
    if not auth_token:
        context.set_code(grpc.StatusCode.UNAUTHENTICATED)
        return user_pb2.GetUserResponse()
    
    # Process request...

# Client: Send metadata
metadata = [('authorization', 'Bearer token123')]
response = stub.GetUser(
    user_pb2.GetUserRequest(id=1),
    metadata=metadata
)
```

### Interceptors (Middleware)

```python
class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        auth_token = metadata.get('authorization')
        
        if not auth_token:
            return grpc.unary_unary_rpc_method_handler(
                lambda request, context: self._unauthenticated(context)
            )
        
        return continuation(handler_call_details)
    
    def _unauthenticated(self, context):
        context.set_code(grpc.StatusCode.UNAUTHENTICATED)
        context.set_details('Missing authorization token')

# Add interceptor to server
server = grpc.server(
    futures.ThreadPoolExecutor(max_workers=10),
    interceptors=[AuthInterceptor()]
)
```

### Deadlines (Timeouts)

```python
# Client: Set deadline
response = stub.GetUser(
    user_pb2.GetUserRequest(id=1),
    timeout=5.0  # 5 seconds
)

# Server: Check deadline
def GetUser(self, request, context):
    if context.time_remaining() < 1.0:
        context.set_code(grpc.StatusCode.DEADLINE_EXCEEDED)
        return user_pb2.GetUserResponse()
    
    # Process request...
```

### Load Balancing

```python
# Client-side load balancing
channel = grpc.insecure_channel(
    'dns:///my-service:50051',
    options=[
        ('grpc.lb_policy_name', 'round_robin'),
        ('grpc.enable_retries', 1),
    ]
)
```

---

## 8. When to Use gRPC

### Good fit

```
✅ Service-to-service communication (microservices)
✅ High performance requirements
✅ Streaming (real-time data)
✅ Polyglot environments (multiple languages)
✅ Internal APIs

Examples:
  - Microservices communication
  - Real-time data streaming
  - Mobile apps (gRPC-Mobile)
  - IoT devices
```

### Poor fit

```
❌ Browser clients (limited support)
❌ Public APIs (REST is more standard)
❌ Simple CRUD operations
❌ Human-readable APIs

Examples:
  - Public REST APIs
  - Web applications (use REST)
  - Simple internal tools
```

---

## 9. gRPC in Production

### Service Mesh Integration

```
Istio/Linkerd:
  - Automatic mTLS
  - Load balancing
  - Circuit breaking
  - Observability

gRPC works well with service mesh
```

### Observability

```
Metrics:
  - Request count
  - Latency (p50, p95, p99)
  - Error rate

Tracing:
  - OpenTelemetry integration
  - Distributed tracing

Logging:
  - Request/response logging
  - Error logging
```

### Security

```
TLS:
  - Encrypt traffic
  - Mutual TLS (mTLS)

Authentication:
  - JWT tokens in metadata
  - OAuth 2.0

Authorization:
  - Interceptors for auth checks
```

---

## 10. Common Interview Questions + Answers

### Q: What is gRPC and when would you use it?

> "gRPC is a high-performance RPC framework that uses HTTP/2 and Protocol Buffers for efficient service-to-service communication. It's strongly typed with auto-generated client code, supports streaming, and is much faster than REST. You'd use it for internal microservices communication where performance matters, real-time streaming applications, or polyglot environments where you need type-safe APIs across multiple languages. It's not ideal for browser clients or public APIs where REST is more standard."

### Q: How does gRPC differ from REST?

> "gRPC uses binary Protocol Buffers instead of JSON, making it about 7x faster with 10x smaller payloads. It uses HTTP/2 for multiplexing and native streaming support, while REST uses HTTP/1.1. gRPC is strongly typed with compile-time validation, while REST is loosely typed with runtime validation. However, REST has better browser support and is more human-readable, making it better for public APIs. gRPC is better for internal service-to-service communication where performance matters."

### Q: What are the different communication patterns in gRPC?

> "gRPC supports four patterns: Unary RPC is like REST — single request, single response. Server streaming sends one request and receives a stream of responses, useful for large result sets. Client streaming sends a stream of requests and receives one response, useful for file uploads. Bidirectional streaming allows both client and server to send streams independently, useful for chat or real-time collaboration. All patterns use HTTP/2 for efficient multiplexing."

### Q: What are the challenges with gRPC?

> "First, limited browser support — you need gRPC-Web with a proxy for browser clients. Second, debugging is harder since payloads are binary, not human-readable like JSON. Third, it requires code generation from .proto files, adding build complexity. Fourth, it's less familiar than REST, requiring team training. Finally, it's overkill for simple CRUD operations where REST would suffice. Despite these challenges, the performance benefits make it worthwhile for internal microservices."

---

## 11. Quick Reference

```
What is gRPC?
  High-performance RPC framework
  HTTP/2 + Protocol Buffers
  Strongly typed, fast, streaming

Core concepts:
  Protocol Buffers: Binary serialization, schema definition
  Service Definition: RPC methods in .proto file
  HTTP/2: Multiplexing, streaming, binary headers

Communication patterns:
  - Unary: Single request, single response
  - Server streaming: Single request, stream of responses
  - Client streaming: Stream of requests, single response
  - Bidirectional: Stream of requests and responses

gRPC vs REST:
  gRPC: Binary, HTTP/2, fast, strongly typed, streaming
  REST: JSON, HTTP/1.1, human-readable, loosely typed

Benefits:
  - 7x faster than REST
  - 10x smaller payload
  - Strongly typed (compile-time validation)
  - Native streaming support
  - Auto-generated client code

Challenges:
  - Limited browser support
  - Binary (not human-readable)
  - Requires code generation
  - Less familiar than REST

Error handling:
  Status codes: OK, NOT_FOUND, INVALID_ARGUMENT, etc.
  context.set_code() and context.set_details()

Advanced features:
  - Metadata (headers)
  - Interceptors (middleware)
  - Deadlines (timeouts)
  - Load balancing

When to use:
  ✅ Service-to-service communication
  ✅ High performance requirements
  ✅ Streaming
  ✅ Polyglot environments
  
  ❌ Browser clients
  ❌ Public APIs
  ❌ Simple CRUD

Best practices:
  - Use for internal APIs
  - Enable TLS/mTLS
  - Add interceptors for auth
  - Set deadlines on all calls
  - Monitor latency and errors
  - Use service mesh (Istio)
```
