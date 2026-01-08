# URL Shortener – Short Design Document

## 1. Overview

A URL Shortener converts long URLs into short, unique links and redirects users efficiently. The system is **read-heavy**, requires **low latency**, and must **scale horizontally**.

---

## 2. Requirements

### Functional

* Shorten long URLs
* Redirect short URL → original URL
* Optional custom alias
* Optional expiration
* Track click count (basic analytics)

### Non-Functional

* Low latency (<20 ms redirects)
* High availability
* Scalable to millions/billions of URLs

---

## 3. High-Level Architecture

Client → Load Balancer → URL Shortener Service → Cache (Redis) → Database

* Redis for fast lookups
* Database for durability
* Queue for async analytics

---

## 4. API Design

### Create Short URL

```
POST /api/v1/shorten
```

```json
{
  "longUrl": "https://example.com",
  "customAlias": "myurl",
  "expireAt": "2026-01-01T00:00:00Z"
}
```

### Redirect

```
GET /{shortCode} → 302 Redirect
```

---

## 5. Short Code Generation

### Why Base62 Encoding

Base62 uses the following characters:

```
a–z (26) + A–Z (26) + 0–9 (10) = 62 characters
```

**Reasons for choosing Base62:**

* **Shorter URLs:** Higher base results in fewer characters for the same numeric ID
* **URL-safe:** Uses only alphanumeric characters, no encoding required
* **No collisions:** When combined with a unique ID generator
* **Easy to implement:** Simple math-based encoding/decoding

**Example:**

```
Numeric ID: 1,000,000
Base10  → 1000000 (7 chars)
Base62  → 4c92    (4 chars)
```

### Why NOT Base64

Base64 uses the following characters:

```
A–Z a–z 0–9 + / =
```

**Problems with Base64:**

* **Not URL-safe:** Contains `+`, `/`, `=` which need URL encoding
* **Longer URLs:** Padding (`=`) increases length
* **Messy redirects:** Requires decoding before redirect
* **Higher processing overhead**

**Example Issue:**

```
Base64 output: aB+/=
URL encoded : aB%2B%2F%3D
```

Because of these issues, Base64 is avoided in URL shorteners.

### ID Generation

* Generate a unique numeric ID (Auto-increment or Snowflake)

* Encode the ID using Base62 to generate the short code

* Generate a unique numeric ID (Auto-increment or Snowflake)

* Encode the ID using Base62 to generate the short code

---

## 6. Data Model

| Field           | Description     |
| --------------- | --------------- |
| short_code (PK) | Unique code     |
| long_url        | Original URL    |
| expire_at       | Expiration time |
| click_count     | Number of hits  |

---

## 7. Redirect Flow

1. Request comes with short code
2. Check Redis cache
3. If miss → fetch from DB and cache it
4. Return 302 redirect

---

## 8. Caching Strategy

* Redis key: `short_code`
* Value: `long_url`
* TTL based on expiration

---

## 9. Analytics

* Increment click count asynchronously
* Use Queue (Kafka / PubSub)
* Avoid blocking redirects

---

## 10. Expiration Handling

* Redis TTL for auto-expiry
* Periodic DB cleanup job

---

## 11. Scalability & Reliability

* Horizontal service scaling
* DB sharding by short code
* CDN for faster global redirects

---

## 12. Summary

A scalable URL Shortener using **Base62 encoding**, **Redis caching**, and **async analytics** to deliver fast, reliable redirects at massive scale.


