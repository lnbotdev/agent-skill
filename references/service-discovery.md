# Service Discovery via satring.com

Agents need to find L402-enabled APIs to spend sats on. [satring.com](https://satring.com) is an open directory of Lightning-paywalled services, built for programmatic access.

Base URL: `https://satring.com/api/v1`

## Endpoints

### List services

```
GET /api/v1/services?page=1&page_size=20&category=ai-ml&sort=rating
```

Query parameters:

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | Page number (default 1) |
| `page_size` | int | Results per page (default 20) |
| `category` | string | Filter by category slug |
| `sort` | string | `newest`, `rating`, `price_asc`, `price_desc` |

Response:

```json
{
  "services": [
    {
      "id": 1,
      "name": "Example AI API",
      "slug": "example-ai-api",
      "url": "https://api.example.com",
      "description": "AI inference behind an L402 paywall",
      "pricing_sats": 100,
      "pricing_model": "per_request",
      "protocol": "L402",
      "owner_name": "Example Corp",
      "logo_url": "",
      "avg_rating": 4.5,
      "rating_count": 12,
      "domain_verified": true,
      "categories": [
        { "id": 1, "name": "AI & ML", "slug": "ai-ml", "description": "..." }
      ],
      "created_at": "2025-01-15T12:00:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "page_size": 20
}
```

### Search services

```
GET /api/v1/search?q=image+generation
```

Returns the same `ServiceListOut` format filtered by search query.

### Get service details

```
GET /api/v1/services/{slug}
```

Returns a single `ServiceOut` object with full details.

### Get service ratings

```
GET /api/v1/services/{slug}/ratings
```

Returns an array of ratings with score (1-5), comment, reviewer name, and timestamp.

### Categories

Available category slugs: `ai-ml`, `data`, `finance`, `identity`, `media`, `search`, `social`, `storage`, `tools`.

## CLI usage

```bash
# Browse all services
curl -s https://satring.com/api/v1/services | jq '.services[] | {name, url, pricing_sats, avg_rating}'

# Search for a specific capability
curl -s "https://satring.com/api/v1/search?q=image" | jq '.services[] | {name, url, pricing_sats}'

# Get details for a specific service
curl -s https://satring.com/api/v1/services/example-ai-api | jq

# Filter by category
curl -s "https://satring.com/api/v1/services?category=ai-ml&sort=rating" | jq '.services[] | {name, url, pricing_sats}'
```

## Full workflow: discover, pay, access

```bash
# 1. Find a service
SERVICE=$(curl -s "https://satring.com/api/v1/search?q=translation" | jq -r '.services[0].url')

# 2. Hit the L402-gated endpoint, get the 402 challenge
CHALLENGE=$(curl -si "$SERVICE/api/translate" | grep -i www-authenticate | cut -d' ' -f2-)

# 3. Pay the challenge with ln.bot
AUTH=$(lnbot l402 pay "$CHALLENGE")

# 4. Access the service with the L402 token
curl -H "Authorization: $AUTH" "$SERVICE/api/translate" -d '{"text": "hello", "target": "es"}'
```

```typescript
import { LnBot } from "@lnbot/sdk";

const ln = new LnBot({ apiKey: "key_..." });

// 1. Discover services from satring.com
const res = await fetch("https://satring.com/api/v1/search?q=translation");
const { services } = await res.json();
const service = services[0];

// 2. Request the L402-gated endpoint
const gatedRes = await fetch(`${service.url}/api/translate`, { method: "POST" });

if (gatedRes.status === 402) {
  const challenge = gatedRes.headers.get("www-authenticate")!;

  // 3. Pay the L402 challenge
  const result = await ln.l402.pay({ wwwAuthenticate: challenge });

  // 4. Retry with authorization
  const finalRes = await fetch(`${service.url}/api/translate`, {
    method: "POST",
    headers: { Authorization: result.authorization! },
    body: JSON.stringify({ text: "hello", target: "es" }),
  });
}
```

```python
import httpx
from lnbot import LnBot

ln = LnBot(api_key="key_...")

# 1. Discover services from satring.com
services = httpx.get("https://satring.com/api/v1/search?q=translation").json()["services"]
service = services[0]

# 2. Request the L402-gated endpoint
response = httpx.post(f"{service['url']}/api/translate")

if response.status_code == 402:
    challenge = response.headers["www-authenticate"]

    # 3. Pay the L402 challenge
    result = ln.l402.pay(www_authenticate=challenge)

    # 4. Retry with authorization
    final = httpx.post(
        f"{service['url']}/api/translate",
        headers={"Authorization": result.authorization},
        json={"text": "hello", "target": "es"},
    )
```
