# Realtime Mouse Pointers

The main technologies are the websockets used to exchange data between the user and the Canva backend instance. We used the Redis database as a message broker routing data between the stateless microservice instances.

### **1. Backend-centric architecture**

![image.png](Realtime%20Mouse%20Pointers/image.png)

Data flow between the client and the server.

- When a new backend instance spins up, all new WebSocket connections are routed to the new instance until the number of connections on it is roughly the same as on all other instances. This load-balancing strategy ensures that the number of connections stays the same across all backend instances and the backend instances are not overloaded.
- users collaborating on the same design would connect to the same server instance, and the server would route messages in-memory. Redis has 2 ways of propagating messages, similar to the message queues: Redis Streams and PubSub. Streams provide better delivery guarantees but require the Redis clients to actively poll for new messages. PubSub messages are pushed to the subscribed Redis clients and not saved in Redis' memory. PubSub messages are lost if the connection between the server and Redis breaks.
- We used Streams for the data related to the session control information, such as notifying that a new user connected or disconnected. Streams also lets us ensure messages aren’t lost if the backend service loses the Redis connection. After the backend service reconnects, it can still read the messages it missed while offline.
- Each microservice instance has a Redis Stream entry assigned to it. Only that instance can read from it, and all other services can only write to it. This model ensures that the instance receives only messages related to the users who’re connected to it. When a client connects to the microservice instance, an entry is added to the design session information with the instance ID. The senders check the content of the session information and send the data only to the instances with the clients connected to them.
- We pass the mouse pointer position updates that make up most of the traffic through the PubSub channels. Each server instance has a single Redis connection dedicated to the PubSub messages. When a new client connects to the server, the server subscribes the PubSub connection to the messages related to that document.

![image.png](Realtime%20Mouse%20Pointers/image%201.png)

Routing messages between the users.

![image.png](Realtime%20Mouse%20Pointers/image%202.png)

Using the Redis PubSub channel to broadcase mouse pointer position updates.

- Binary Serialization: using Binary Encoding for the mouse pointer messages.
- Redis client flushing: By disabling auto flushing and batching the Redis PubSub commands by flushing them manually within a given time interval

---

### 2. WebRTC implementation

Creating a connection using WebRTC requires: Clients, STUN/TURN servers (to receive client IP address and NAT reversal), Signaling server (for exchanging metadata)

<aside>
🦩

Establishing a connection between 2 clients consists of:

1. Client A (the initiator) creates an offer Session Description Protocol (SDP) containing information about the session and codex and sends it through the signaling server to Client B.
2. When Client B (the receiver) gets an offer, it creates an answer SDP and sends it through the signaling server to Client A.
3. At the same time, both clients gather Interactive Connectivity Establishment (ICE) candidates from STUN servers, prioritizing each and sharing them through the signaling server.
4. The WebRTC browser implementation builds the ICE candidate pairs, choosing the best one and uses it to establish a peer-to-peer connection.
</aside>

![image.png](Realtime%20Mouse%20Pointers/image%203.png)

A simplified flow of establishing a WebRTC connection.

WebRTC connections are established from each user to all other users working on a design (mesh topology). WebRTC also supports other data exchange architectures, such as Selective Forwarding Unit (SFU) and Multipoint Control Unit (MCU). Their benefits include fewer connections and traffic for each client, an important consideration for real-time video conferencing. We chose the mesh architecture because it puts the lowest load on the backend.

Our frontend architecture already has information about the available clients in a session. When a new client joins a session, it acts as an initiator for all other clients in the session. WebRTC supports the SCTP data channel interface, which can be opened on the connection and allows sending and receiving data in String or binary formats.  We can now send the mouse pointer positions we share with the WebSocket solution through the WebRTC data channel.

The protocol uses a 2-way handshake to open a data channel. To keep the channel open, it periodically sends ping messages, adding extra overhead, and we must carefully control the number of active connections.

#### TURN V/S P2P CONNECTION:

The ICE candidate represents the client's transport address. When establishing the WebRTC connection, a client sends its ICE candidates to all other clients, letting them know how they can connect to it. The ICE candidate could have the following types: host (address directly to client), reflex (address to public side of NAT), and relay (address is allocated from the TURN server).

If both clients:

1. host-to-host ICE candidates: P2P connection with best latency and no intermediate server.
2. not a direct connection: WebRTC uses relayed candidates, which connects clients through a TURN server. 

Migration from WebSockets: We’re prioritizing peer-to-peer connections, and because we’re tracking all active clients in the session, we know which clients successfully opened the WebRTC channel. If the client fails to open the WebRTC channel with at least one other active client in the session, we’re activating a fallback WebSocket channel to share updates with those clients. All the messages include the sender’s client ID. For those clients that open the WebRTC channel, all the received messages are taken from there and ignored from the WebSocket channel. The update from the client is sent to both channels, so all clients, regardless of which solution they are using, receive an update. To simplify the migration, we created an abstraction over the WebRTC API that provided the same API as the WebSocket solution.

Existing open-source solutions like adaptor.js help with cross-browser implementation.

If the clients would have joined approximately at the same time. To resolve this issue, we follow the first-come, first-serve rule. Each client assigns a random ID to the offer they send, and the offer with the lower ID wins. If this happens, the other client terminates the connection they initiated and uses the one from the first client.