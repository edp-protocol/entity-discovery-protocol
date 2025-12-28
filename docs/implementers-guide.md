# Implementers Guide

This guide provides implementation guidance for the Entity Discovery Protocol (EDP).

## Audiences

1. **Registry Operators** — Build services that index entities and provide resolution APIs
2. **MCP Providers** — Register entities you serve with EDP registries
3. **Agent Developers** — Query EDP registries to resolve entities to MCPs

---

## For Registry Operators

A registry indexes entities and provides resolution APIs for AI agents.

### Core Responsibilities

1. **Crawl Entity Cards** from `.well-known/entity-card.json`
2. **Accept Provider Registrations** for bulk entity imports
3. **Enrich with Metadata** from external sources (Schema.org, Google Business, etc.)
4. **Provide Resolution APIs** for agents to query
5. **Verify Associations** at the appropriate level

### Recommended API Endpoints

```
GET  /v1/resolve?query={search}&location={loc}    # Search by name/keywords
GET  /v1/resolve/domain/{domain}                  # Lookup by domain
GET  /v1/nearby?lat={lat}&lng={lng}&radius={m}    # Geo search
POST /v1/provider/register                        # Bulk provider registration
```

### Metadata Enrichment

Entity Cards only contain MCP associations. Registries should enrich entries with business metadata from:

- **Schema.org / JSON-LD** on the entity's website
- **Google Business Profile** (via API)
- **Social media** (Open Graph tags)
- **Provider registration** data

### Verification Implementation

**Level 0**: Provider Registration only — trust the provider

**Level 1**: Entity Card exists at `/.well-known/entity-card.json` — proves domain control

**Level 2**: Both sources match — Provider Registration + Entity Card agree on the relationship

```python
def determine_verification_level(entity, provider_data, entity_card):
    has_provider_claim = entity in provider_data
    has_entity_card = entity_card is not None
    
    if has_provider_claim and has_entity_card:
        # Check if they match
        if entity_card_matches_provider(entity_card, provider_data):
            return 2  # Dual attestation
    
    if has_entity_card:
        return 1  # Entity claim only
    
    if has_provider_claim:
        return 0  # Provider claim only
    
    return None  # Not indexed
```

**Optional JWT verification** (for Level 2 with crypto):

```python
import jwt

def verify_jwt_signature(entity_domain, mcp_entry, provider_public_key):
    signature = mcp_entry["verification"]["signature"]
    payload = jwt.decode(signature, provider_public_key, algorithms=["ES256"])
    
    assert payload["iss"] == mcp_entry["provider"]
    assert payload["sub"] == entity_domain
    assert payload["exp"] > time.time()
    
    return True
```

---

## For MCP Providers

As an MCP provider, you serve entities and can register them with EDP registries.

### Option 1: Bulk Registration

Register all your entities with EDP registries:

```json
POST /v1/provider/register

{
  "provider": {
    "id": "your-provider-id",
    "name": "Your Provider Name",
    "endpoint": "https://mcp.yourprovider.com",
    "public_key": "-----BEGIN PUBLIC KEY-----\n..."
  },
  "entities": [
    {
      "entity_id": "entity-123",
      "name": "Business Name",
      "domain": "business.com",
      "location": { "city": "Paris", "country": "FR" },
      "capabilities": ["reservations"]
    }
  ]
}
```

### Option 2: Help Entities Publish Cards

Generate Entity Card data for your customers to publish:

```json
{
  "provider": "your-provider-id",
  "endpoint": "https://mcp.yourprovider.com",
  "entity_id": "entity-123",
  "capabilities": ["reservations"],
  "verification": {
    "method": "signed_jwt",
    "signature": "<generated-jwt>",
    "issued_at": "2025-12-28T00:00:00Z",
    "expires_at": "2026-12-28T00:00:00Z"
  }
}
```

### Signing Entity Claims (Level 2)

```bash
# Generate ES256 key pair
openssl ecparam -genkey -name prime256v1 -noout -out private.pem
openssl ec -in private.pem -pubout -out public.pem
```

```python
import jwt
from datetime import datetime, timedelta

def sign_entity_claim(entity_domain, entity_id, capabilities):
    payload = {
        "iss": "your-provider-id",
        "sub": entity_domain,
        "entity_id": entity_id,
        "capabilities": capabilities,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + timedelta(days=365)
    }
    return jwt.encode(payload, private_key, algorithm="ES256")
```

---

## For Agent Developers

Agents query EDP registries to resolve entities to MCPs.

### Basic Resolution Flow

```python
def resolve_and_call(entity_name, location, action):
    # 1. Query EDP registry
    response = requests.get(
        "https://registry.example.com/v1/resolve",
        params={"query": entity_name, "location": location}
    )
    result = response.json()
    
    if not result["results"]:
        return None
    
    # 2. Select appropriate MCP by capability
    entity = result["results"][0]
    mcp = select_mcp_for_action(entity["mcps"], action)
    
    # 3. Call MCP
    return call_mcp(
        endpoint=mcp["endpoint"],
        entity_id=mcp["entity_id"],
        action=action
    )
```

### Verification Considerations

| Action Type | Recommended Min Level |
|-------------|----------------------|
| Information queries | 0 |
| Bookings | 1 |
| Payments | 2 (with JWT recommended) |

### Direct Entity Card Fetch

If you know the entity's domain:

```python
def fetch_entity_card(domain):
    url = f"https://{domain}/.well-known/entity-card.json"
    response = requests.get(url, timeout=5)
    if response.status_code == 200:
        return response.json()
    return None
```

---

## Testing

### Validate Entity Cards

```bash
# Using ajv-cli
ajv validate -s schemas/entity-card.schema.json -d my-entity-card.json
```

---

## Security Checklist

### For Registries
- [ ] Validate all Entity Card schemas
- [ ] Verify JWT signatures for Level 2
- [ ] Rate limit API endpoints

### For Providers
- [ ] Rotate signing keys annually
- [ ] Only sign claims for legitimate customers

### For Agents
- [ ] Check verification levels before sensitive actions
- [ ] Validate MCP endpoints are HTTPS
