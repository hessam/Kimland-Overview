# Kimland

**A production shop aggregator that tracks and synchronises products across multiple fashion brands — built for reliability, not demos.**

> This repository contains an architectural overview and sanitised code samples. The full codebase is private.

---

## What it does

Kimland scrapes, classifies, and synchronises product catalogues across 8 fashion brands into a single unified database, then exposes them via a clean REST API to a mobile frontend. The system runs continuously — new products are detected, price changes are tracked, and the WooCommerce layer stays in sync without manual intervention.

The main design constraint was reliability. Scrapers hit external sites that go down, change their markup, or rate-limit aggressively. The system has to handle all of that without corrupting data or requiring a restart.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Mobile Client                     │
└─────────────────────┬───────────────────────────────┘
                      │ REST API
┌─────────────────────▼───────────────────────────────┐
│              API Gateway (Express / TS)              │
│         Auth · Rate Limiting · Validation            │
└──────┬──────────────────────────────┬───────────────┘
       │                              │
┌──────▼──────┐               ┌───────▼──────────────┐
│  Scraping   │               │   WooCommerce Sync   │
│   Engine    │               │   (Adapter Layer)    │
│             │               └───────┬──────────────┘
│ Strategy A  │                       │
│ Strategy B  │               ┌───────▼──────────────┐
│ Strategy N  │               │     PostgreSQL        │
└──────┬──────┘               │  Products · Orders   │
       │                      │  Financials · Rates  │
┌──────▼──────┐               └──────────────────────┘
│    Redis    │
│    Cache    │
└─────────────┘
```

---

## Key Patterns

### Strategy Pattern — Brand Scrapers

Each brand has different markup, pagination, and JavaScript behaviour. A single scraper can't handle all of them cleanly. The Strategy pattern keeps brand-specific logic isolated — adding a new brand means writing one new strategy, nothing else changes.

```typescript
// The contract every brand scraper must fulfil
interface ScrapingStrategy {
  readonly brandId: string;
  fetchProductList(categoryUrl: string): Promise<RawProduct[]>;
  parseProductDetail(html: string): ParsedProduct;
  requiresJavaScript(): boolean;
}

// The engine doesn't know which brand it's running — it just calls the strategy
class ProductScrapingService {
  constructor(private readonly strategies: Map<string, ScrapingStrategy>) {}

  async scrape(brandId: string, categoryUrl: string): Promise<RawProduct[]> {
    const strategy = this.strategies.get(brandId);
    if (!strategy) throw new Error(`No strategy registered for brand: ${brandId}`);
    return strategy.fetchProductList(categoryUrl);
  }
}

// Registration at startup — new brand = new line here
const strategies = new Map<string, ScrapingStrategy>([
  ['brand-a', new BrandAStrategy()],
  ['brand-b', new BrandBStrategy()],
  // add more without touching anything else
]);
```

---

### PostgreSQL Layer — UPSERT & Indexing

Products are re-scraped on a schedule. The database layer needs to handle re-inserts cleanly — updating changed fields, ignoring unchanged rows, never duplicating. Raw SQL with `ON CONFLICT DO UPDATE` handles this without an ORM getting in the way.

```typescript
// UPSERT — insert or update on conflict, ignore if nothing changed
const upsertProduct = async (product: ParsedProduct): Promise<void> => {
  await db.query(`
    INSERT INTO products (sku, slug, brand_id, title, price, image_url, updated_at)
    VALUES ($1, $2, $3, $4, $5, $6, NOW())
    ON CONFLICT (sku)
    DO UPDATE SET
      title      = EXCLUDED.title,
      price      = EXCLUDED.price,
      image_url  = EXCLUDED.image_url,
      updated_at = NOW()
    WHERE
      products.price     IS DISTINCT FROM EXCLUDED.price
      OR products.title  IS DISTINCT FROM EXCLUDED.title
  `, [product.sku, product.slug, product.brandId, product.title, product.price, product.imageUrl]);
};
```

```sql
-- Indexes on the fields hit most by the API
CREATE INDEX idx_products_slug    ON products USING btree (slug);
CREATE INDEX idx_products_sku     ON products USING btree (sku);
CREATE INDEX idx_products_brand   ON products USING btree (brand_id);

-- GIN index for full-text search on product titles
CREATE INDEX idx_products_title_fts ON products USING gin (to_tsvector('english', title));
```

---

### Circuit Breaker — External API Resilience

The WooCommerce sync layer calls an external WordPress API. When that API is slow or down, naive retries pile up and take everything with them. The circuit breaker tracks failure rates and stops calling a service that's already failing — then tries again after a cooldown.

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime = 0;

  constructor(
    private readonly threshold: number = 5,
    private readonly cooldownMs: number = 30_000
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.cooldownMs) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit open — service unavailable');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

---

### Redis Caching Layer

Product listings and classification results are cached in Redis/Upstash. Cache keys are scoped by brand and category so invalidation is surgical — a re-scrape of one brand doesn't flush unrelated data.

```typescript
class ScrapingCache {
  constructor(private readonly redis: Redis, private readonly ttlSeconds: number) {}

  private key(brandId: string, categorySlug: string): string {
    return `scrape:${brandId}:${categorySlug}`;
  }

  async get(brandId: string, categorySlug: string): Promise<RawProduct[] | null> {
    const cached = await this.redis.get(this.key(brandId, categorySlug));
    return cached ? JSON.parse(cached) : null;
  }

  async set(brandId: string, categorySlug: string, products: RawProduct[]): Promise<void> {
    await this.redis.set(
      this.key(brandId, categorySlug),
      JSON.stringify(products),
      'EX',
      this.ttlSeconds
    );
  }

  async invalidate(brandId: string, categorySlug: string): Promise<void> {
    await this.redis.del(this.key(brandId, categorySlug));
  }
}
```

---

### Docker & Nginx

The application runs in Docker with Nginx handling SSL termination, reverse proxying, and rate limiting at the network edge before requests reach Node.

```nginx
# /etc/nginx/sites-available/kimland
server {
    listen 443 ssl http2;
    server_name api.kimland.store;

    ssl_certificate     /etc/letsencrypt/live/api.kimland.store/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.kimland.store/privkey.pem;

    # Rate limiting at Nginx level — before Node even sees the request
    limit_req zone=api_limit burst=20 nodelay;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY dist/ ./dist/

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## Observability

Kimland exposes a `/metrics` endpoint in Prometheus format. Tracked in production:

- Cache hit/miss rates per brand
- P95 and P99 response times per endpoint
- PostgreSQL pool status (active vs idle connections)
- Slow query detection (queries exceeding threshold)
- Scraper success/failure rates per brand
- Circuit breaker state per external dependency

Errors are captured via Sentry with `node-profiling-sentry` for performance tracing on background scraper tasks.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20, TypeScript |
| Framework | Express 5 |
| Database | PostgreSQL (raw SQL, node-pg-migrate) |
| Cache | Redis / Upstash |
| Scraping | Puppeteer (dynamic), Cheerio (static) |
| Auth | JWT + bcrypt |
| Observability | Prometheus, Sentry, Winston |
| Infrastructure | Docker, Nginx, Ubuntu |
| Testing | Jest, Supertest |

---

## Testing

Critical paths have full Jest + Supertest coverage: authentication, product sync logic, UPSERT behaviour, and API route contracts. Background scraper tasks are unit tested with mocked HTTP responses so tests don't hit live brand sites.

---

*Private repository — architectural overview only. Contact [mousavihessam0@email.com] for more details.*
