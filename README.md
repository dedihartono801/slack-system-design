# Chat Realtime System Design - Flow Phases

## Diagram
![alt text](https://github.com/dedihartono801/slack-system-design/blob/master/system-design.jpg)

## Functional Requirements

### Users can send and receive messages in 1:1 or group chats

This covers both private and group conversations, with messages delivered in order and stored persistently. Group chats can support dynamic membership and maintain message history.

### Users can send rich media (e.g., images, videos, files) as part of a message

Messages can include text, attachments, or both. The system must support media upload, storage, and preview rendering across devices.

### Users can receive real-time notifications for new messages

When a user receives a new message—either in a direct chat or group—they should be notified through in-app indicators, push notifications, or email (based on settings and availability).

### Users can delete their own messages from a chat

A user should be able to remove a message they previously sent. The deletion may be visible (e.g., "This message was deleted") or fully removed from view, depending on the UX and policy.

---

## Non-Functional Requirements

### High Scalability

The system should support 1 B users and 100k chats concurrently, especially in high-traffic group chats.

### Low Latency in Chat Delivery

Aim for P95 end-to-end message latency under 200ms for online users.

### CAP - Aware Messaging Guarantees

The system should prefer high availability, but must enforce strong ordering consistency within each chat room.

### Message Durability

Messages must be safely persisted before confirming to the sender. No acknowledged message should be lost, even if the server crashes right after.

---

## 1. Initialization Phase: WebSocket Handshake

When Client A launches the app, the system establishes a real-time communication channel:

**WebSocket Connection:** Client A initiates a handshake with the WebSocket Server (Go).

**Session Caching:** Upon connection, the server stores the session in Redis Cache using a `user_id:websocket_id` pair with a 10-second TTL (Time To Live).

**Keep-Alive:** To maintain the connection, the client sends a heartbeat every 5 seconds via `PEXPIRE` to refresh the session in Redis.

**Subscription:** The server queries the database for all channels (`chat_id`) the user belongs to and performs a `SUBSCRIBE` operation in Redis Pub/Sub for those specific IDs.

---

## 2. Message Submission Flow

When Client A sends a message:

**Gateway:** The request hits the API Gateway for authentication, rate limiting, and routing.

**Media Service:** If the message includes a file, this service validates the metadata and uploads the file to Amazon S3, returning a signed URL.

**Chat Service (Go):** The core service receives the payload (text + file URL) and acts as a Kafka Producer.

**Kafka Partitioning:** Messages are published to Kafka topics. To ensure message ordering within a specific chat, the partition is determined by:

```
Partition = hash(chat_id) mod n_partitions
```

---

## 3. Storage and Event Processing

Once the message is in the Kafka cluster:

**Persistence:** A Kafka Consumer reads the message and writes it into the Postgres DB (specifically the `message` tables).

**Change Data Capture (CDC):** The Kafka Worker monitors the Postgres transaction log. When a new message is detected, the worker fetches the list of channel members to begin the distribution process.

**Message Processing:** A Kafka Consumer consumes message events from Kafka, writes them atomically to PostgreSQL, fetches the list of channel members, and publishes a realtime event to Redis Pub/Sub to trigger fan-out to WebSocket servers.

---

## 4. Real-Time Delivery Logic

The system then determines how to reach the recipients (Client B and C):

**Pub/Sub Trigger:** The Kafka Consumer publishes the message to the Redis Pub/Sub channel.

**Websocket Filtering:** The WebSocket Server receives the event and checks the recipient's status in the cache:

- **Online (Client B):** The server pushes the message directly via the open WebSocket. Once delivered, the `is_delivered` flag is updated to `true` in the database.

- **Offline (Client C):** The server triggers the Push Notification Service and the message remains in the `inbox` table as "pending."

---

## 5. Maintenance: The Inbox Cleanup

The Clean Up Cronjob plays a vital role in database optimization regarding the `inbox` table:

**Purpose of the Inbox Table:** This table acts as a temporary buffer for messages that the WebSocket could not deliver immediately because a user was offline.

**Sync on Reconnect:** When an offline user comes back online, the WebSocket server fetches their pending messages from the `inbox` table, delivers them, and updates the status to `is_delivered = true`.

**The Cleanup Process:** The Cronjob periodically scans the `inbox` table and deletes records where `is_delivered` is `true`. This prevents the table from growing indefinitely and ensures high performance for active message syncing.

---

## Technology Choices

### Why PostgreSQL & Redis?

**PostgreSQL:** Chosen as the Primary Source of Truth because of its ACID compliance. Chat applications require strong consistency for message history and user relationships. It handles complex relational data (like `chat_members` and user profiles) much better than NoSQL.

**Redis:** Serves two critical roles:
- **Cache:** Stores ephemeral session data (`user_id:websocket_id`) for sub-millisecond lookups.
- **Pub/Sub:** Acts as a lightweight message bus to broadcast messages across multiple distributed WebSocket nodes.

### Why WebSocket & Kafka?

**WebSocket:** Essential for Full-Duplex communication. Unlike HTTP polling, WebSockets allow the server to "push" messages to the client the instant they arrive, reducing latency and overhead.

**Kafka:** Used as the Message Backbone. It provides high throughput and durability. If the database or worker services go down, Kafka retains the messages (replayability), ensuring no data is lost during spikes or outages.

### Why Golang?

**Concurrency (Goroutines):** Go can handle thousands of concurrent WebSocket connections per node with very low memory overhead compared to thread-based languages like Java.

**Performance:** It offers near-C performance with a much simpler syntax, making it ideal for high-throughput services like the Chat Service and Media Service.

**Standard Library:** Go has excellent built-in support for network primitives and binary serialization.

---

## Handling Scale and Reliability

### Horizontal Scaling

**Stateless Services:** The Gateway and Chat Services are stateless and can be scaled behind a Load Balancer.

**WebSocket Sharding:** Since WebSockets are stateful, we use Redis Pub/Sub as a backplane. If User A is on Server 1 and User B is on Server 2, Server 1 publishes to Redis, and Server 2 (subscribed to the same channel) picks it up and pushes it to User B.

**Database Partitioning:** As the message table grows, we can implement Sharding based on `chat_id`.

### Fault Tolerance

**Node Crashes:** If a WebSocket server crashes, the client's "Keep-Alive" logic (`PEXPIRE`) will fail. The client will then attempt to reconnect to a healthy node via the Load Balancer.

**Message Durability:** Kafka ensures that even if a worker crashes mid-process, the "offset" (progress marker) is only committed once the message is safely written to Postgres.

**Retry Logic:** The Push Notification Service uses exponential backoff to ensure offline users eventually receive alerts even if the third-party provider (FCM/APNs) is temporarily down.

### Load Balancing

**Layer 7 (L7) Load Balancer:** We use an L7 LB (like Nginx or HAProxy) with Sticky Sessions or Least Connections algorithm.

**Connection Draining:** When a node needs maintenance, the LB stops sending new connections to it, allowing existing WebSocket sessions to finish or reconnect naturally.

### Message Consistency

**Causal Ordering:** By using `hash(chat_id)` to select a Kafka partition, we guarantee that all messages within a specific conversation are processed by the same consumer in the exact order they were sent.

**Idempotency:** Each message is assigned a unique UUID by the client or Gateway. The database uses a unique constraint on this ID to prevent duplicate messages in case of network retries.

## Table
![alt text](https://github.com/dedihartono801/slack-system-design/blob/master/table.jpg)
