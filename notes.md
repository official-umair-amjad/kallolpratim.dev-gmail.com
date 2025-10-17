# Database Design Notes

## 1. Design Goals and Approach

The objective is to create a scalable, flexible, and robust chat system supporting private (1:1), group, and public conversations, with features for message delivery/read tracking and file sharing, capable of handling billions of users.

**Core Entities:**

- **User:** Represents platform participants.
- **Conversation:** Models chats (private, group, public).
- **ConversationMember:** Tracks user membership, roles, and join/leave timestamps.
- **Message:** Stores messages sent within conversations.
- **MessageDeliveryStatus and MessageReadStatus:** Track delivery and read confirmations for each recipient.
- **FileMetadata:** Manages metadata for multimedia and file attachments.

These entities address all requirements: 1:1 chats, group chats, public channels, message lifecycle tracking, and file support.

---

## 2. Key Design Decisions

**Conversation Types**  
A type field (private, group, public) in the conversation table differentiates chat behaviors within a unified model, simplifying logic and ensuring extensibility.

**Membership Management**  
The conversation_member table tracks user participation with joined_at and left_at timestamps. This enables precise control over message visibility (e.g., users only see messages from their active membership period) and supports user removal or rejoining.

**Message Visibility**  
Message access is restricted using joined_at and left_at filters in queries, avoiding the need for message deletion or archiving while maintaining data integrity.

**File Handling**  
File metadata is stored in a separate file_metadata table to decouple it from the message table, enabling efficient querying and scalability. Actual files are stored in an external object storage system (e.g., AWS S3, Google Cloud Storage).

**Identifiers**  
UUIDs are used as primary keys for all tables to ensure global uniqueness and facilitate sharding in distributed systems.

**Message Status Tracking**  
Separate message_delivery_status and message_read_status tables support multiple recipients (critical for group chats) and enable granular tracking without bloating the message table.

---

## 3. SQL vs. NoSQL Evaluation

### SQL Approach (Primary Choice)
A relational database (SQL) was chosen for the core schema due to:

- **Structured Relationships:** Clear relationships between users, conversations, and memberships benefit from SQL’s relational model.
- **ACID Compliance:** Ensures consistency for critical operations (e.g., joining/leaving conversations, updating statuses).
- **Expressive Queries:** SQL’s join capabilities and constraints simplify complex queries (e.g., filtering messages by membership).
- **Maintainability:** Referential integrity (foreign keys) enforces data correctness, ideal for early-stage development.

### NoSQL Considerations
A NoSQL database (e.g., MongoDB, Cassandra, DynamoDB) could be considered for high-volume data like messages and statuses due to:

- **Write Scalability:** NoSQL excels at handling high-throughput, write-heavy workloads (e.g., billions of messages).
- **Flexible Schema:** Message documents could embed delivery/read statuses for faster reads in certain use cases.
- **Horizontal Scaling:** NoSQL’s native sharding simplifies partitioning for massive datasets.

### Recommended Architecture
A hybrid approach optimizes for both consistency and scale:

- **SQL:** Store user, conversation, conversation_member, and file_metadata for strong consistency and relational integrity.
- **NoSQL:** Store message, message_delivery_status, and message_read_status for high write throughput and scalability.
- **Object Storage:** Use S3 , Google cloud storage or equivalent for file storage, with metadata in SQL for efficient querying.

---

## 4. Weaknesses and Mitigations

| Weakness | Impact | Mitigation |
|-----------|---------|------------|
| Large message table | Query performance degrades with billions of rows. | Shard by conversation_id or timestamp; consider NoSQL for messages or use time-based partitioning. |
| High write load for statuses | Delivery/read updates create write amplification. | Use asynchronous writes via message queues (e.g., Kafka, RabbitMQ) and eventual consistency for non-critical updates. |
| Complex SQL queries | Joins across large tables slow down at scale. | Implement caching (Redis, Memcached) for frequent queries; use materialized views or denormalized tables for performance. |
| Storage costs | Historical messages increase storage demands. | Archive old messages to cold storage (e.g., S3 Glacier) with queryable indexes; implement retention policies. |

---

## 5. Scalability and Extensibility

**Horizontal Scaling**  
UUID-based keys enable sharding across database clusters. Time-based partitioning for messages reduces query latency. Caching layers (e.g., Redis) and message queues handle high-frequency events (e.g., typing indicators, read receipts).

**Search and Analytics**  
A dedicated search index (e.g., Elasticsearch, OpenSearch) supports full-text message search and analytics without impacting transactional performance.

**File Management**  
External object storage (S3, GCS) scales file storage independently. The file_metadata table supports future features like thumbnails or multiple attachments per message.

**Event-Driven Architecture**  
Use message queues for asynchronous processing of status updates, notifications, and analytics to reduce database load and improve responsiveness.

---

## 6. Conclusion

The design balances data integrity, scalability, and flexibility. A relational SQL core ensures consistency for user and conversation management, while a NoSQL layer for messages and statuses supports high-throughput writes. External object storage handles files efficiently. This hybrid architecture supports both early-stage development and long-term scaling to billions of users, with clear paths for optimization (caching, sharding, archiving).
