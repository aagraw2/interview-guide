## 1. What are Server-Sent Events (SSE)?

Server-Sent Events (SSE) is a standard for servers to push real-time updates to clients over HTTP. It's a one-way communication channel from server to client.

```
Traditional polling:
  Client → Server: "Any updates?" (every 5 seconds)
  Server → Client: "No" or "Yes, here's data"
  → Wasteful, high latency

SSE:
  Client → Server: Open connection
  Server → Client: Push updates as they happen
  → Efficient, low latency
```

**Use cases:** Live notifications, real-time feeds, stock tickers, live scores, progress updates.

---

## 2. SSE vs WebSocket vs Long Polling

|Feature|SSE|WebSocket|Long Polling|
|---|---|---|---|
|Direction|Server → Client (one-way)|Bidirectional|Client → Server (request/response)|
|Protocol|HTTP|WebSocket (ws://)|HTTP|
|Complexity|Simple|Complex|Simple|
|Reconnection|Automatic|Manual|Manual|
|Browser support|All modern browsers|All modern browsers|All browsers|
|Firewall friendly|Yes (HTTP)|Sometimes blocked|Yes (HTTP)|
|Use case|Server pushes to client|Real-time chat, gaming|Fallback for SSE/WebSocket|

---

## 3. How SSE Works

### Connection establishment

```
Client:
  GET /events HTTP/1.1
  Accept: text/event-stream

Server:
  HTTP/1.1 200 OK
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive

Connection stays open
Server pushes events as they occur
```

### Event format

```
data: This is a message\n\n

data: {"user": "Alice", "message": "Hello"}\n\n

event: notification\n
data: {"type": "new_order", "id": 123}\n\n

id: 1\n
event: update\n
data: {"status": "processing"}\n\n
```

Format:
- `data:` Message content (required)
- `event:` Event type (optional, default: "message")
- `id:` Event ID for reconnection (optional)
- `\n\n` End of event (required)

---

## 4. Client Implementation

### JavaScript (EventSource API)

```javascript
// Open connection
const eventSource = new EventSource('/events');

// Listen to all events
eventSource.onmessage = (event) => {
  console.log('Message:', event.data);
  const data = JSON.parse(event.data);
  updateUI(data);
};

// Listen to specific event types
eventSource.addEventListener('notification', (event) => {
  console.log('Notification:', event.data);
  showNotification(JSON.parse(event.data));
});

// Handle errors
eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  // EventSource automatically reconnects
};

// Close connection
eventSource.close();
```

### Automatic reconnection

```
Connection lost:
  → EventSource automatically reconnects
  → Sends Last-Event-ID header with last received event ID
  → Server resumes from that point

Client:
  GET /events HTTP/1.1
  Last-Event-ID: 42

Server:
  → Send events starting from ID 43
```

---

## 5. Server Implementation

### Python (Flask)

```python
from flask import Flask, Response
import time
import json

app = Flask(__name__)

@app.route('/events')
def events():
    def generate():
        # Send events
        while True:
            # Get new data (from database, queue, etc.)
            data = get_new_notifications()
            
            if data:
                # Format as SSE
                yield f"data: {json.dumps(data)}\n\n"
            
            time.sleep(1)  # Check every second
    
    return Response(
        generate(),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'X-Accel-Buffering': 'no'  # Disable nginx buffering
        }
    )
```

### Node.js (Express)

```javascript
const express = require('express');
const app = express();

app.get('/events', (req, res) => {
  // Set headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  // Send event
  const sendEvent = (data) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
  
  // Send events periodically
  const interval = setInterval(() => {
    const data = getNewNotifications();
    if (data) {
      sendEvent(data);
    }
  }, 1000);
  
  // Cleanup on close
  req.on('close', () => {
    clearInterval(interval);
    res.end();
  });
});
```

---

## 6. Event Types and IDs

### Custom event types

```
Server:
  event: notification\n
  data: {"message": "New order"}\n\n
  
  event: update\n
  data: {"status": "shipped"}\n\n

Client:
  eventSource.addEventListener('notification', (e) => {
    showNotification(JSON.parse(e.data));
  });
  
  eventSource.addEventListener('update', (e) => {
    updateStatus(JSON.parse(e.data));
  });
```

### Event IDs for resumption

```
Server:
  id: 1\n
  data: {"message": "First"}\n\n
  
  id: 2\n
  data: {"message": "Second"}\n\n
  
  id: 3\n
  data: {"message": "Third"}\n\n

Client disconnects after event 2

Client reconnects:
  GET /events
  Last-Event-ID: 2

Server:
  → Send events starting from ID 3
```

---

## 7. Scaling SSE

### Problem: Long-lived connections

```
10,000 concurrent users with SSE:
  → 10,000 open connections
  → Each connection holds a thread/worker
  → High memory usage
```

### Solution 1: Async/event-driven servers

```
Use async servers that don't block threads:
  - Node.js (event loop)
  - Python asyncio (aiohttp)
  - Go (goroutines)

Can handle 10,000+ connections per server
```

### Solution 2: Message broker

```
Application servers → Redis Pub/Sub → SSE servers

Application:
  redis.publish('notifications', json.dumps(data))

SSE server:
  pubsub = redis.pubsub()
  pubsub.subscribe('notifications')
  
  for message in pubsub.listen():
      send_to_clients(message['data'])

Benefits:
  - Decouple application from SSE
  - Scale SSE servers independently
  - Multiple SSE servers share same events
```

### Solution 3: Sticky sessions

```
Load balancer with sticky sessions:
  User A → SSE Server 1 (always)
  User B → SSE Server 2 (always)

Ensures user stays connected to same server
```

---

## 8. Use Cases

### Live notifications

```
Server:
  event: notification\n
  data: {"type": "new_message", "from": "Alice", "text": "Hello"}\n\n

Client:
  eventSource.addEventListener('notification', (e) => {
    const notif = JSON.parse(e.data);
    showNotification(notif.text);
  });
```

### Real-time feed

```
Server:
  event: post\n
  data: {"user": "Bob", "content": "Just posted!", "timestamp": 1234567890}\n\n

Client:
  eventSource.addEventListener('post', (e) => {
    const post = JSON.parse(e.data);
    prependToFeed(post);
  });
```

### Progress updates

```
Server:
  event: progress\n
  data: {"percent": 25, "status": "Processing..."}\n\n
  
  event: progress\n
  data: {"percent": 50, "status": "Halfway there..."}\n\n
  
  event: progress\n
  data: {"percent": 100, "status": "Complete!"}\n\n

Client:
  eventSource.addEventListener('progress', (e) => {
    const progress = JSON.parse(e.data);
    updateProgressBar(progress.percent);
  });
```

### Live dashboard

```
Server:
  event: metrics\n
  data: {"cpu": 45, "memory": 60, "requests": 1234}\n\n

Client:
  eventSource.addEventListener('metrics', (e) => {
    const metrics = JSON.parse(e.data);
    updateDashboard(metrics);
  });
```

---

## 9. Limitations

### One-way communication

```
SSE: Server → Client only

If client needs to send data:
  → Use regular HTTP POST/PUT
  → Or use WebSocket instead

Example:
  SSE for notifications (server → client)
  HTTP POST for user actions (client → server)
```

### Browser connection limits

```
HTTP/1.1: 6 connections per domain
  → Opening 6 SSE connections blocks other requests

HTTP/2: Multiplexing (no limit)
  → Use HTTP/2 for SSE

Workaround:
  → Use subdomain for SSE (events.example.com)
  → Doesn't count toward main domain limit
```

### No binary data

```
SSE: Text only (UTF-8)

For binary data:
  → Base64 encode (increases size by 33%)
  → Or use WebSocket
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between SSE and WebSocket?

> "SSE is one-way communication from server to client over HTTP, while WebSocket is bidirectional over a separate protocol. SSE is simpler to implement, automatically reconnects, and is firewall-friendly since it uses HTTP. WebSocket is more powerful for bidirectional communication like chat or gaming. Use SSE when you only need server-to-client push (notifications, live feeds) and WebSocket when you need bidirectional communication. SSE is also easier to scale since it's just HTTP."

### Q: How does SSE handle reconnection?

> "SSE automatically reconnects when the connection is lost. The client sends a Last-Event-ID header with the ID of the last event it received. The server resumes sending events from that point, preventing missed events. This is built into the EventSource API — you don't need to implement reconnection logic. The server should assign sequential IDs to events and buffer recent events to support resumption."

### Q: How would you scale SSE to handle 100,000 concurrent connections?

> "Use async/event-driven servers like Node.js or Python asyncio that can handle many connections without blocking threads. Decouple the application from SSE servers using a message broker like Redis Pub/Sub — application servers publish events to Redis, and SSE servers subscribe and push to clients. Use sticky sessions at the load balancer so each client stays connected to the same SSE server. Scale horizontally by adding more SSE servers. Monitor connection counts and memory usage per server."

### Q: When would you use SSE instead of polling?

> "Use SSE when you need real-time updates with low latency and want to reduce server load. Polling wastes bandwidth and server resources by constantly asking for updates even when there are none. SSE keeps a connection open and pushes updates only when they occur. Use SSE for notifications, live feeds, dashboards, and progress updates. Use polling only as a fallback for browsers that don't support SSE or when SSE is blocked by firewalls."

---

## 11. Quick Reference

```
What is SSE?
  Server-to-client push over HTTP
  One-way communication
  Automatic reconnection

SSE vs WebSocket:
  SSE: One-way, HTTP, simple, auto-reconnect
  WebSocket: Bidirectional, ws://, complex, manual reconnect

Event format:
  data: Message content\n\n
  event: Event type (optional)\n
  id: Event ID (optional)\n
  \n\n (end of event)

Client (JavaScript):
  const es = new EventSource('/events');
  es.onmessage = (e) => console.log(e.data);
  es.addEventListener('notification', handler);
  es.close();

Server (Python Flask):
  def generate():
      yield f"data: {json.dumps(data)}\n\n"
  return Response(generate(), mimetype='text/event-stream')

Reconnection:
  Client sends Last-Event-ID header
  Server resumes from that event
  Automatic in EventSource API

Scaling:
  Async servers (Node.js, asyncio)
  Message broker (Redis Pub/Sub)
  Sticky sessions (load balancer)
  Horizontal scaling (multiple SSE servers)

Use cases:
  Live notifications
  Real-time feeds
  Progress updates
  Live dashboards
  Stock tickers

Limitations:
  One-way only (server → client)
  Browser connection limits (6 per domain in HTTP/1.1)
  Text only (no binary)

Best practices:
  - Use HTTP/2 (no connection limits)
  - Assign event IDs (support resumption)
  - Use message broker (decouple app from SSE)
  - Monitor connection counts
  - Set appropriate timeouts
```
