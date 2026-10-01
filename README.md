# URL Shortener

## Overview

URL Shortener is a backend service that converts long URLs into compact Base62 short keys and redirects users to the original URLs. It is built with Spring Boot and uses PostgreSQL for persistent data and Redis for caching, rate limiting, and efficient redirect handling.

## Tech Stack

- **Java 17**
- **Spring Boot 4**
- **PostgreSQL** - persistent storage for URL mappings and click events
- **Redis** - URL cache and rate-limit counters
- **Hibernate / Spring Data JPA** - ORM and repository layer
- **Lettuce** - Redis client through Spring Data Redis
- **Docker** - local PostgreSQL and Redis containers

## Features

- **Base62 URL shortening** - generates a compact short key from the stored URL mapping ID.
- **302 redirects** - resolves a short key and redirects to the original URL.
- **Optional URL expiry** - accepts an expiry timestamp and stops serving expired URLs.
- **HTTP/HTTPS validation** - accepts only URLs with a valid host and an `http` or `https` scheme.
- **Centralized exception handling** - maps application exceptions to consistent HTTP error responses.
- **Redis cache-aside strategy** - caches short-key lookups and loads the database only after a cache miss.
- **Atomic click-count updates** - increments the stored total with a database `UPDATE` operation.
- **Redis-based rate limiting** - limits URL creation requests with a fixed-window counter per IP address.
- **Scheduled URL expiry handling** - removes expired URLs from Redis and marks them inactive in PostgreSQL.
- **Asynchronous click analytics** - records click events in a background task so redirects do not wait for analytics writes.
- **Analytics endpoint** - reports total clicks, recent click counts, and a daily breakdown.
- **Docker support** - runs PostgreSQL and Redis locally in containers.

## Architecture

### 1. URL creation

```text
POST /api/shorten
  -> validate HTTP/HTTPS URL and expiry
  -> save URL mapping in PostgreSQL
  -> encode the database ID as a Base62 short key
  -> save the short key
  -> cache short key -> long URL in Redis
  -> return the short URL
```

### 2. Redirect with a Redis cache hit

```text
GET /{shortKey}
  -> Redis HIT
  -> start asynchronous click analytics
  -> return 302 FOUND
```

The cached value provides the destination without a database lookup. Analytics records a `click_event` and atomically increments the total click count in the background.

### 3. Redirect with a Redis cache miss

```text
GET /{shortKey}
  -> Redis MISS
  -> find an active mapping in PostgreSQL
  -> check expiry
  -> cache the destination in Redis
  -> start asynchronous click analytics
  -> return 302 FOUND
```

If the mapping does not exist, the service returns a not-found error. If it has expired, the service evicts the Redis key and returns an expired-URL error.

### 4. URL expiry

```text
Scheduled job (every 5 minutes by default)
  -> find expired active mappings
  -> evict each short key from Redis
  -> mark expired mappings inactive in PostgreSQL
```

When an expiry is provided, the Redis TTL is calculated from `expiresAt`. URLs without an expiry use the configured default cache TTL. The service also checks expiry during a database-backed redirect, so an expired URL is not redirected while waiting for the scheduled cleanup.

### 5. Rate limiting

```text
POST /api/shorten
  -> Redis INCR rate:{ip}:{windowSlot}
  -> set the window TTL on the first request
  -> allow the request or return 429 when the limit is exceeded
```

The default limit is 5 URL-creation requests per 60-second window per IP. Redirect traffic is not rate limited.

## API Reference

### Create a short URL

```http
POST /api/shorten
Content-Type: application/json
```

Request:

```json
{
  "longUrl": "https://www.example.com",
  "expiresAt": "2026-03-01T00:00:00"
}
```

`expiresAt` is optional. Response `201 CREATED`:

```json
{
  "shortUrl": "http://localhost:8080/aB3"
}
```

### Redirect to the original URL

```http
GET /{shortKey}
```

- `302 FOUND` for an active short URL
- `404 NOT FOUND` when the short key does not exist
- `410 GONE` when the short URL has expired

### View analytics

```http
GET /api/analytics/{shortKey}
```

Response `200 OK`:

```json
{
  "shortKey": "aB3",
  "longUrl": "https://www.example.com",
  "createdAt": "2026-02-24T06:12:35",
  "expiresAt": null,
  "isActive": true,
  "totalClicks": 42,
  "clicksLast24Hours": 10,
  "clicksLast7Days": 35,
  "clicksPerDay": [
    { "date": "2026-02-24", "count": 10 },
    { "date": "2026-02-23", "count": 25 }
  ]
}
```

## Database Schema

```postgresql
CREATE TABLE url_mapping (
    id          BIGSERIAL PRIMARY KEY,
    short_key   VARCHAR(10) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    click_count BIGINT DEFAULT 0,
    created_at  TIMESTAMP NOT NULL,
    expires_at  TIMESTAMP,
    is_active   BOOLEAN DEFAULT true
);

CREATE TABLE click_event (
    id         BIGSERIAL PRIMARY KEY,
    short_key  VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_click_event_short_key ON click_event(short_key);
CREATE INDEX idx_click_event_clicked_at ON click_event(clicked_at);
```

- **`url_mapping`** stores the current short-to-long URL mapping, its lifecycle fields, and the total click counter.
- **`click_event`** stores timestamped clicks used for 24-hour, 7-day, and daily analytics queries.

## Project Structure

```text
src/main/java/com/urlshortener/url_shortener/
|-- controller/                         # HTTP API and redirect controllers
|   |-- UrlShortenerController.java     # POST /api/shorten
|   |-- RedirectController.java          # GET /{shortKey}
|   `-- AnalyticsController.java        # GET /api/analytics/{shortKey}
|-- service/                             # URL, cache, and analytics services
|   |-- UrlShortenerService.java
|   |-- UrlShortenerServiceImpl.java
|   |-- CacheService.java
|   |-- RedisCacheServiceImpl.java
|   |-- AnalyticsService.java
|   `-- AnalyticsServiceImpl.java        # Asynchronous click recording
|-- schedular/                           # Scheduled URL expiry processing
|   `-- UrlExpirySchedular.java
|-- interceptor/                         # Request interception and rate limiting
|   `-- RateLimitInterceptor.java
|-- repository/                          # Spring Data JPA repositories
|   |-- UrlMappingRepository.java
|   `-- ClickEventRepository.java
|-- model/                               # JPA entities
|   |-- UrlMapping.java
|   `-- ClickEvent.java
|-- dto/                                 # Request and response objects
|   |-- ShortenUrlRequest.java
|   |-- ShortenUrlResponse.java
|   `-- UrlAnalyticsResponse.java
|-- exception/                           # Custom exceptions and error handling
|   |-- GlobalExceptionHandler.java
|   |-- UrlNotFoundException.java
|   |-- UrlExpiredException.java
|   |-- InvalidUrlException.java
|   `-- TooManyRequestsException.java
|-- config/                              # Redis and MVC configuration
|   |-- RedisConfig.java
|   `-- WebMvcConfig.java
`-- util/                                # Shared encoding utilities
  `-- Base62Encoder.java
```

## Key Engineering Decisions

**Cache-aside caching** - Redirects first check Redis. On a miss, the service reads PostgreSQL and then stores the destination in Redis, keeping cache population explicit and easy to reason about.

**Redis TTL based on URL expiry** - A URL with `expiresAt` receives a Redis TTL based on the time remaining until expiry. URLs without an expiry use the configured default cache TTL.

**Atomic click-count update** - Click totals are incremented with `click_count = click_count + 1` in the database, avoiding a read-modify-write race between requests.

**Asynchronous analytics** - Click event insertion and total-count updates run through `@Async`, allowing the redirect response to return without waiting for analytics persistence.

**Expiry correctness** - The scheduler evicts Redis before deactivating the database record. Redirects also check expiry after a database lookup, so an expired URL is not served during cleanup.

**Rate limiting only on URL creation** - The interceptor limits `POST /api/shorten` to 5 requests per 60 seconds per IP by default. Redirects remain available for normal high-volume use.

**Fail-open rate limiting** - If Redis cannot provide a rate-limit count, the request is allowed. This keeps URL creation available when the rate-limit dependency is unavailable.

## Running Locally

**Prerequisites:** Java 17, Maven, and Docker.

Start PostgreSQL:

```bash
docker run -d --name url_shortener_postgres \
  -e POSTGRES_DB=url_shortener \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=your_password \
  -p 5432:5432 postgres:15
```

Start Redis:

```bash
docker run -d --name my-redis-container -p 6379:6379 redis:7-alpine
```

Configure the application in `src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/url_shortener
spring.datasource.username=postgres
spring.datasource.password=your_password
spring.data.redis.host=localhost
spring.data.redis.port=6379
```

Run the Spring Boot application:

```bash
./mvnw spring-boot:run
```

On Windows, use `mvnw.cmd spring-boot:run`.