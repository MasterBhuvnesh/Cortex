# BITLY

Bitly is a URL shortening service that converts long web links into shorter, easy-to-share URLs while also providing analytics to track clicks and engagement.

## 1. Requirements
## Functional Requirements

|No.| Requirement |
|---|-------------|
| 1 | Users can create short URLs from original URLs, with optional custom aliases and expiration dates. |
| 2 | Users can access the original URL by browsing the short URL. |

## Non-Functional Requirements

|No. | Requirement |
|---|-------------|
| 1 | Shortened URLs must be unique. |
| 2 | Redirection latency must be under 100 milliseconds. |
| 3 | Availability is prioritized over consistency (AP system from CAP Theorem). |
| 4 | The system must support 1 billion URLs and 100 million daily active users (DAU). |

## 2. Core Entities
- **Original URL**: The full-length target URL.
- **Shortened URL**: The generated or custom alias that redirects to the original URL.
- **User**: The creator of the shortened URL.
## 3. API Specifications
### 3.1 Shorten a URL
**Endpoint:** `POST /short_url`
**Request Body:**

```json
{
    "long_url": "https://www.app.openstudio.tech/studio",
    "custom_alias": "openstudio",
    "expiration_date": "2025-11-11"
}
```
**Response:**

```json
{
    "short_url": "https://os.ly/openstudio"
}
```
### 3.2 Redirection
**Endpoint:** `GET /{short_code}`
**Behavior:** Returns an HTTP 302 redirect to the original URL.

## 4. High-Level Design Overview
The system comprises several key components:

- **Client**: Initiates URL shortening and redirection requests.
- **Server**: Handles business logic, validation, and coordination.
- **Database**: Stores URL mappings and metadata.
- **Cache**: An in-memory store (e.g., Redis) for fast redirection lookups.
- **Counter Service**: A centralized atomic counter for generating unique short codes.
**Key Workflows:**

1. **URL Shortening:** 
    - Client sends a `POST /short_url`  request. 
    - Server validates the URL, checks for existing entries, generates a unique short code, and saves the mapping to the database. 
    - The short URL is returned to the client.

2. **Redirection:** 
    - Client requests `GET /{short_code}` . 
    - Server first checks the cache for the mapping. 
    - If not found, it queries the database, returns a 302 redirect, and caches the result for future requests.


## 5. Deep Dives
### 5.1 Ensuring Unique Short URLs
**Constraints:** Generate unique, compact, and efficient short codes.

**Approaches Considered:**

1. **Using the First N Characters of the Long URL:** 
    - Violates uniqueness as similar URLs produce identical prefixes.
    - Cons: clearly violates unique constraint.

2. **Random Number Generation or Hash Functions:** 
    - Base62 encoding of a random number or hash output. 
    - Collision probability increases with scale (Birthday Problem). 
    - Requires longer codes or additional database checks to ensure uniqueness, impacting latency and efficiency.
    - Example: 
        - input_url = "https://github.com/masterbhuvnesh"
        - random_number = Math.random()
        - short_code_encoded = base62_encode(random_number)
        - short_code=short_code_encoded[:8]

    - Cons: As urls increase, the chance of collision increases.
    - To avoid this, we need high entropy, which means longer short codes, which is against our constraints.

3. **Unique Counter with Base62 Encoding (Recommended & Used):** 
    - Maintain a global atomic counter (e.g., in Redis). 
    - Increment for each new URL and encode the counter value in Base62. 
    - Guarantees uniqueness and compactness. 
    - Example: A 6-character Base62 code can represent up to ~56.8 billion unique URLs (62⁶). For 1 billion URLs, this remains efficient.
    - Concern: In distributed env for synchronization.


### 5.2 Ensuring Fast Redirections
**Challenge:** Database full-table scans become a bottleneck at scale.

**Solutions:**

1. **Database Indexing:** 
    - Use a hash index (B-Tree) on the short-URL column for O(1) lookups.

2. **In-Memory Cache (Redis):** 
    - Store frequently accessed short URL mappings. 
    - Provides microsecond-level read latency. 
    - Requires a cache invalidation strategy for updates/deletions and warming for initial requests.

3. **CDNs and Edge Computing:** 
    - Deploy redirect logic to edge servers (e.g., Cloudflare Workers, AWS Lambda ). 
    - Reduces latency by serving redirects geographically closer to users. 
    - Adds complexity in configuration, debugging, and cost.
    - Cons: 
        - Cache invalidation & consistency issues become complex.
        - Setting up edge computing is configuration & cost overhead.
        - Debugging and monitoring are way easier in our servers than in distributed edge environments.
        - Trading cost and complexity for performance.


**Latency Comparison:**

- Memory (Redis): ~0.0001 ms
- SSD: ~0.1 ms
- HDD: ~10 ms

**Throughput Analysis:**

For 100M DAU with 5 redirects per day: 

- Average: ~5,800 redirects/second 
- Peak (100x): ~580,000 redirects/second

A database alone cannot handle peak traffic; caching is essential.


<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../application assets/bitly.flow.dark.png">
  <source media="(prefers-color-scheme: light)" srcset="../application assets/bitly.flow.png">
  <img alt="Bitly Flow Integration" src="../application assets/bitly.flow.png">
</picture>

### 5.3 Scaling to 1B URLs and 100M DAU
**Storage Estimation:** 

- `short_url_code` : 8 bytes 
- `original_url` : 100 bytes 
- `created_time` : 8 bytes 
- `custom_alias` : 100 bytes 
- `expiration_date` : 8 bytes 
- **Total per row:** ~224 bytes + metadata ≈ 500 bytes 
- **For 1B rows:** ~500 GB
A single modern database can manage this volume. If needed, **sharding **can be implemented.
**Write Load:**
Assuming 100K new URLs per day ≈ 1 write/second, easily handled by databases like PostgreSQL.

**System Resilience:**

- **Database Failure:** Implement replication (primary-replica) and regular backups.
- **Service Scaling:** Separate read and write services for independent horizontal scaling.
- **Microservices Architecture:** 
    - API Gateway routes requests. 
    - Write Service handles URL creation. 
    - Read Service manages redirection lookups.

**Final Design Components:**

1. **Client** 
2. **API Gateway** 
3. **Write Service** (with Redis counter for atomic increments) 
4. **Read Service** (with Redis cache for mappings) 
5. **Database** (with indexing and optional sharding) 
6. **CDN/Edge** (optional for geographic latency reduction)


<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../application assets/bitly.aws.dark.png">
  <source media="(prefers-color-scheme: light)" srcset="../application assets/bitly.aws.png">
  <img alt="Bitly AWS Integration" src="../application assets/bitly.aws.png">
</picture>


## 6. Additional Considerations
- **Redirect Type (302 vs. 301):**
Use **302 (Temporary Redirect)** to ensure all requests pass through our server for analytics and control. A 301 (Permanent Redirect) may be cached by browsers, bypassing our service.
- **Cache Invalidation:** Implement strategies (e.g., TTL, write-through) to maintain consistency between cache and database.
- **Monitoring and Debugging:** Essential for distributed systems, especially when using edge computing.
## 7. Conclusion
This design outlines a scalable, high-availability URL shortening service. By leveraging a unique counter for short code generation, in-memory caching for fast redirections, and a microservices architecture for scalability, the system meets the functional and non-functional requirements. Future enhancements could include advanced analytics, user dashboards, and enhanced security features.

