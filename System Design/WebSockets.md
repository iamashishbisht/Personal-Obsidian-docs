**WebSocket** are a **communication protocol** used to build **real-time features** by establishing a **two-way connection** between a client and a server.

Imagine an online multiplayer game where the leaderboard **updates instantly** as players score points, showing **real-time rankings** of all players.

> [!IMPORTANT] WebSockets enable **full-duplex, bidirectional** communication between a client (typically a web browser) and a server over a **single** **TCP connection**.

## 1. How do WebSockets work?

The WebSocket connection starts with a standard HTTP request from the client to the server.

However, instead of completing the request and closing the connection, the server responds with an **[HTTP 101](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/101)** status code, indicating that the protocol is switching to WebSockets.

After this handshake, a WebSocket connection is established, and both the client and server can send messages to each other over the open connection.

The HTTP **`101 Switching Protocols`** [informational response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status#informational_responses) status code indicates the protocol that a server has switched to. The protocol is specified in the [`Upgrade`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Upgrade) request header received from a client.
The HTTP `Upgrade` [request](https://developer.mozilla.org/en-US/docs/Glossary/Request_header) and [response header](https://developer.mozilla.org/en-US/docs/Glossary/Response_header) can be used to upgrade an already-established client/server connection to a different protocol (over the same transport protocol). For example, it can be used by a client to upgrade a connection from HTTP/1.1 to HTTP/2, or an HTTP(S) connection to a WebSocket connection.

![[Pasted image 20260930224708.png]]


 ###  Step-by-Step Process:
### 1. Handshake

The client initiates a connection request using a standard HTTP GET request with an "Upgrade" header set to "websocket".

![](https://substackcdn.com/image/fetch/$s_!AdVo!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F44500cd5-32b6-459e-bb1e-f4fca01cbc6f_836x446.png)

If the server supports WebSockets and accepts the request, it responds with a special 101 status code, indicating that the protocol will be changed to WebSocket.

![](https://substackcdn.com/image/fetch/$s_!_rzC!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb9d4230e-b56a-4b6a-a85f-cb0f1e5a6f78_974x306.png)

#### 2. Connection

Once the handshake is complete, the WebSocket connection is established. This connection remains open until explicitly closed by either the client or the server.

#### 3. Data Transfer

Both the client and server can now send and receive messages in real-time.

These messages are sent in small packets called **frames**, and carry minimal overhead compared to traditional HTTP requests.

#### 4. Closure

The connection can be closed at any time by either the client or server, typically with a **"close" frame** indicating the reason for closure.

## 2. Why are WebSockets used?

WebSockets offer several advantages that make them ideal for certain types of applications:

- **Real-time Updates**: WebSockets enable instant data transmission, making them perfect for applications that require real-time updates, like live chat, gaming, or financial trading platforms.
    
- **Reduced Latency**: Since the connection is persistent, there's no need to establish a new connection for each message, significantly reducing latency.
    
- **Efficient Resource Usage**: WebSockets are more efficient than traditional polling techniques, as they don't require the client to continuously ask the server for updates.
    
- **Bidirectional Communication**: Both the client and server can initiate communication, allowing for more dynamic and interactive applications.
    
- **Lower Overhead**: After the initial handshake, WebSocket frames have a small header (as little as 2 bytes), reducing the amount of data transferred.

## 4. Challenges and Considerations

While WebSockets offer numerous benefits, there are some challenges to consider:

- **Proxy Servers**: Some proxy servers don't support WebSocket connections and certain firewalls may block them.
    
- **Scalability Concerns**: Managing a large number of WebSocket connections can be challenging. Consider using **load balancers** and distributed WebSocket servers to handle large-scale deployments.
    
- **Fallback Mechanism**: Not all clients or networks support WebSockets, which can lead to connectivity issues. Implement fallback mechanisms like long-polling for clients that cannot establish WebSocket connections.
    
- **Network Reliability:** WebSockets rely on a persistent connection, which can be disrupted by network issues. Implementing **reconnection strategies** and **heartbeat mechanisms** (regular ping/pong messages) can help maintain the connection's stability and detect when a connection has been lost.
    
- **Security:** WebSockets are vulnerable to attacks such as Cross-Site WebSocket Hijacking and Distributed Denial of Service (DDoS) attacks. Implement **secure WebSocket connections (wss://)**, authenticate users, and validate input to protect against common vulnerabilities.