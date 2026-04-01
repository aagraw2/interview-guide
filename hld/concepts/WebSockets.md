## 1. What Are WebSockets?

WebSockets provide full-duplex, bidirectional communication between client and server over a single, persistent TCP connection.

```
HTTP (request-response):
  Client → Request → Server
  Client ← Response ← Server
  Connection closes

WebSocket (persistent):
  Client ↔ Server (connection stays open)
  Both can send messages anytime
```

**Key difference:** HTTP is request-response. WebSocket is persistent, bidirectional.

---

## 2. Why WebSockets?

### The polling problem

**Short polling:**

```
Client → "Any updates?" → Server → "No"
Wait 1 second
Client → "Any updates?" → Server → "No"
Wait 1 second
Client → "Any updates?" → Server → "Yes, here's data"

Problems:
  - Wasteful (many empty responses)
  - Latency (up to polling interval)
  - Server load (constant requests)
```

**Long polling:**

```
Client → "Any updates?" → Server (holds connection open)
                                  (waits for data)
                                  (30 seconds pass)
                         ← "Here's data"
Client → "Any updates?" → Server (holds again)

Better, but:
  - Still request-response model
  - Reconnection overhead
  - Harder to scale
```

---

### WebSocket solution

```
Client → Upgrade to WebSocket → Server
Client ↔ Persistent connection ↔ Server

Server has data → pushes to client immediately
Client has data → sends to server immediately

Benefits:
  - Real-time (no polling delay)
  - Efficient (no repeated requests)
  - Bidirectional (both can initiate)
```

---

## 3. WebSocket Handshake

WebSocket starts as HTTP, then upgrades.

### Client request

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Key headers:**
- `Upgrade: websocket` — requesting upgrade
- `Connection: Upgrade` — connection will change
- `Sec-WebSocket-Key` — random key for security

---

### Server response

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**101 Switching Protocols** — connection upgraded to WebSocket.

After this, HTTP is done. Connection becomes WebSocket.

---

## 4. WebSocket Communication

### Sending messages

```javascript
// Client (JavaScript)
const ws = new WebSocket('ws://example.com/chat');

// Connection opened
ws.onopen = () => {
  console.log('Connected');
  ws.send('Hello Server!');
};

// Receive message
ws.onmessage = (event) => {
  console.log('Received:', event.data);
};

// Connection closed
ws.onclose = () => {
  console.log('Disconnected');
};

// Error
ws.onerror = (error) => {
  console.error('Error:', error);
};
```

---

### Server (Node.js with ws library)

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('Client connected');
  
  // Receive message
  ws.on('message', (message) => {
    console.log('Received:', message);
    
    // Send response
    ws.send(`Echo: ${message}`);
  });
  
  // Connection closed
  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

---

## 5. Use Cases

### 1. Chat applications

```
User A types message → WebSocket → Server
                                  → broadcasts to all users
User B receives message ← WebSocket ← Server

Real-time, bidirectional
```

---

### 2. Live notifications

```
Server: New order placed
Server → WebSocket → Client (dashboard)
Dashboard updates in real-time
```

---

### 3. Collaborative editing

```
User A types in Google Docs
User A → WebSocket → Server → User B
User B sees changes in real-time
```

---

### 4. Live sports scores

```
Score updates:
Server → WebSocket → All connected clients
Clients see score update immediately
```

---

### 5. Real-time dashboards

```
Metrics update:
Server → WebSocket → Dashboard
Dashboard charts update in real-time
```

---

### 6. Multiplayer games

```
Player A moves
Player A → WebSocket → Server → Player B
Player B sees movement in real-time
```

---

## 6. Scaling WebSockets

### Challenge: Sticky sessions

```
Problem:
  Client connects to Server 1 (WebSocket established)
  Load balancer routes next request to Server 2
  Server 2 doesn't have the WebSocket connection
  → Connection lost

Solution: Sticky sessions
  Load balancer routes all requests from a client to same server
  Client always talks to Server 1
```

**Implementation:**

```
Load balancer:
  - IP hash (route based on client IP)
  - Cookie-based (set cookie, route based on cookie)
```

---

### Challenge: Horizontal scaling

```
Problem:
  User A connected to Server 1
  User B connected to Server 2
  User A sends message → Server 1
  How does Server 2 know to send to User B?

Solution: Message broker (Redis Pub/Sub, Kafka)
  Server 1 → [Redis Pub/Sub] → Server 2
  
  User A → Server 1 → publish to Redis → Server 2 → User B
```

**Implementation:**

```javascript
// Server 1
ws.on('message', (message) => {
  // Publish to Redis
  redis.publish('chat', JSON.stringify({
    user: 'User A',
    message: message
  }));
});

// Server 2
redis.subscribe('chat');
redis.on('message', (channel, data) => {
  const msg = JSON.parse(data);
  // Send to all connected clients on this server
  wss.clients.forEach(client => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg.message);
    }
  });
});
```

---

## 7. WebSocket vs Alternatives

### Server-Sent Events (SSE)

```
SSE:
  - Server → Client only (one-way)
  - HTTP-based (simpler)
  - Auto-reconnect
  - Text only

WebSocket:
  - Client ↔ Server (bidirectional)
  - Custom protocol
  - Manual reconnect
  - Binary or text

Use SSE for: Notifications, live feeds (server pushes to client)
Use WebSocket for: Chat, gaming (bidirectional)
```

---

### Long polling

```
Long polling:
  - HTTP request-response
  - Server holds connection until data available
  - Reconnect after each response

WebSocket:
  - Persistent connection
  - No reconnection overhead
  - Lower latency

Use long polling for: Fallback when WebSocket not available
Use WebSocket for: Real-time, high-frequency updates
```

---

### HTTP/2 Server Push

```
HTTP/2 Server Push:
  - Server pushes resources (CSS, JS) proactively
  - Not for application data
  - One-way (server → client)

WebSocket:
  - Bidirectional application data
  - Real-time communication

Different use cases
```

---

## 8. WebSocket Security

### Authentication

```
Option 1: Token in URL (not recommended)
  ws://example.com/chat?token=abc123
  Problem: Token in URL (logged, cached)

Option 2: Token in first message (recommended)
  Client connects → sends auth message
  ws.send(JSON.stringify({type: 'auth', token: 'abc123'}))
  Server validates → accepts or closes connection

Option 3: Cookie-based
  Client has HTTP session cookie
  WebSocket handshake includes cookie
  Server validates session
```

---

### Authorization

```
Server tracks user permissions:
  - User A can access room "general"
  - User B can access room "admin"

User A tries to join "admin" room:
  Server checks permissions → denies
```

---

### Rate limiting

```
Prevent abuse:
  - Limit messages per second per connection
  - Limit connections per IP
  - Limit message size

Example:
  Max 10 messages/second per user
  If exceeded → disconnect or throttle
```

---

## 9. Connection Management

### Heartbeat / Ping-Pong

Keep connection alive and detect dead connections.

```javascript
// Server sends ping every 30 seconds
setInterval(() => {
  wss.clients.forEach(ws => {
    if (ws.isAlive === false) {
      return ws.terminate();  // connection dead
    }
    ws.isAlive = false;
    ws.ping();  // send ping
  });
}, 30000);

// Client responds with pong
ws.on('pong', () => {
  ws.isAlive = true;
});
```

**Why:** Detect dead connections (client crashed, network issue).

---

### Reconnection

```javascript
// Client auto-reconnect
function connect() {
  const ws = new WebSocket('ws://example.com/chat');
  
  ws.onclose = () => {
    console.log('Disconnected, reconnecting in 5s...');
    setTimeout(connect, 5000);  // reconnect after 5 seconds
  };
  
  ws.onerror = () => {
    ws.close();  // trigger onclose
  };
}

connect();
```

**Exponential backoff:**

```javascript
let reconnectDelay = 1000;  // start with 1 second

ws.onclose = () => {
  setTimeout(connect, reconnectDelay);
  reconnectDelay = Math.min(reconnectDelay * 2, 30000);  // max 30 seconds
};
```

---

## 10. WebSocket Protocols

### Socket.IO

```javascript
// Abstraction over WebSocket with fallbacks
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  console.log('User connected');
  
  socket.on('chat message', (msg) => {
    io.emit('chat message', msg);  // broadcast to all
  });
  
  socket.on('disconnect', () => {
    console.log('User disconnected');
  });
});
```

**Features:**
- Auto-reconnect
- Fallback to long polling
- Rooms and namespaces
- Binary support

**Tradeoff:** Heavier than raw WebSocket.

---

### STOMP (Simple Text Oriented Messaging Protocol)

```
WebSocket + messaging protocol
Used with message brokers (RabbitMQ, ActiveMQ)

SEND /queue/orders
content-type:application/json

{"order_id": 123, "amount": 99.99}
```

---

## 11. Common Interview Questions + Answers

### Q: What are WebSockets and when would you use them?

> "WebSockets provide full-duplex, bidirectional communication over a persistent TCP connection. Unlike HTTP's request-response model, WebSockets allow both client and server to send messages anytime. I'd use WebSockets for real-time applications like chat, live notifications, collaborative editing, or multiplayer games — anywhere you need server-to-client push or high-frequency bidirectional communication. For simple server-to-client updates, Server-Sent Events might be simpler. For occasional updates, long polling or regular HTTP polling might be sufficient."

### Q: How do you scale WebSockets horizontally?

> "WebSockets require sticky sessions — all requests from a client must go to the same server since the WebSocket connection is stateful. I'd configure the load balancer to use IP hash or cookie-based routing. For broadcasting messages across servers, I'd use a message broker like Redis Pub/Sub. When a user on Server 1 sends a message, Server 1 publishes it to Redis, and all servers (including Server 2, 3, etc.) subscribe and forward to their connected clients. This way, users on different servers can communicate."

### Q: What's the difference between WebSockets and Server-Sent Events?

> "WebSockets are bidirectional — both client and server can send messages anytime. SSE is unidirectional — only server can push to client. SSE uses regular HTTP, has built-in auto-reconnect, and is simpler to implement. WebSockets use a custom protocol, require manual reconnect logic, but support binary data and bidirectional communication. I'd use SSE for notifications or live feeds where the server pushes updates to the client. I'd use WebSockets for chat or gaming where both sides need to send messages."

### Q: How do you handle authentication with WebSockets?

> "I'd send an authentication token in the first message after the WebSocket connection is established. The client connects, then immediately sends a message like `{type: 'auth', token: 'jwt_token'}`. The server validates the token and either accepts the connection or closes it. I wouldn't put the token in the URL because URLs are logged and cached. Cookie-based auth works too if you already have HTTP sessions, but token-based is more flexible for mobile apps and third-party clients."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention sticky sessions

When discussing scaling:

> "WebSockets require sticky sessions because the connection is stateful. I'd configure the load balancer to route all requests from a client to the same server using IP hash or cookies. Without this, the client might connect to Server 1 but subsequent requests go to Server 2, breaking the connection."

### ✅ Trick 2: Discuss the message broker pattern

Show you understand distributed WebSockets:

> "For broadcasting across servers, I'd use Redis Pub/Sub. When a user on Server 1 sends a message, Server 1 publishes to Redis. All servers subscribe and forward to their connected clients. This enables users on different servers to communicate."

### ✅ Trick 3: Know when NOT to use WebSockets

Don't default to WebSockets:

> "For simple server-to-client notifications, Server-Sent Events are simpler — they use HTTP, have auto-reconnect, and don't require sticky sessions. For occasional updates, regular HTTP polling might be sufficient. WebSockets add complexity, so I'd only use them when I need real-time bidirectional communication."

### ❌ Pitfall 1: Forgetting about connection management

WebSockets need heartbeats:

> "I'd implement ping-pong heartbeats to detect dead connections. The server sends a ping every 30 seconds, and if the client doesn't respond with a pong, the server closes the connection. This prevents accumulating dead connections that waste resources."

### ❌ Pitfall 2: Not mentioning reconnection

Connections will drop:

> "I'd implement auto-reconnect on the client with exponential backoff. Start with a 1-second delay, double it on each failure, up to a maximum of 30 seconds. This handles temporary network issues without overwhelming the server."

### ❌ Pitfall 3: Ignoring security

WebSockets need auth and rate limiting:

> "I'd authenticate users by validating a token sent in the first message. I'd also implement rate limiting — max 10 messages per second per connection — to prevent abuse. And I'd validate message size to prevent memory exhaustion attacks."

---

## 13. Quick Reference

```
WebSocket = persistent, bidirectional communication over TCP

vs HTTP:
  HTTP:      Request-response, connection closes
  WebSocket: Persistent, bidirectional, both can send anytime

Handshake:
  1. Client sends HTTP Upgrade request
  2. Server responds 101 Switching Protocols
  3. Connection becomes WebSocket

Use cases:
  - Chat applications
  - Live notifications
  - Collaborative editing
  - Live sports scores
  - Real-time dashboards
  - Multiplayer games

Scaling:
  - Sticky sessions (route client to same server)
  - Message broker (Redis Pub/Sub, Kafka) for broadcasting

vs Alternatives:
  SSE:          Server → Client only, simpler, auto-reconnect
  Long polling: HTTP-based, fallback option
  HTTP/2 Push:  For resources, not application data

Security:
  - Auth: Token in first message (not URL)
  - Authorization: Check permissions per message
  - Rate limiting: Max messages/second per connection

Connection management:
  - Heartbeat: Ping-pong every 30 seconds
  - Reconnect: Auto-reconnect with exponential backoff
  - Dead connection detection

Libraries:
  - Socket.IO: Abstraction with fallbacks, rooms, namespaces
  - ws (Node.js): Lightweight WebSocket library
  - STOMP: Messaging protocol over WebSocket

Common patterns:
  - Broadcast: Send to all connected clients
  - Rooms: Group clients, send to room
  - Direct message: Send to specific client

Key insight: WebSockets enable real-time bidirectional communication
```
