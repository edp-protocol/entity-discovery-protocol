# Entity Discovery Protocol Specification

**Version**: 0.1.0 (Draft)  
**Status**: Draft  
**Authors**: Yoann Arzu  
**Last Updated**: 2025-12-28

## Abstract

The Entity Discovery Protocol (EDP) defines a standard way for AI agents to discover which MCP (Model Context Protocol) servers can be used to interact with a given real-world entity (business, organization, or service provider).

While MCP defines how agents communicate with servers, and Server Cards (SEP-1649) describe what MCP servers can do, EDP answers a different question: "Which MCP servers serve this entity?"

## 1. Terminology

| Term | Definition |
|------|------------|
| **Entity** | A real-world business, organization, or service that can be interacted with via MCPs |
| **MCP Provider** | An organization that operates MCP servers on behalf of entities |
| **Entity Card** | A JSON document declaring which MCPs an entity delegates to |
| **EDP Registry** | A service that indexes entities and provides resolution APIs |
| **Resolution** | The process of finding which MCPs serve a given entity |
| **Verification Level** | A measure of trust in the entity-MCP association |

## 2. Entity Card Format

### 2.1 Design Principle

Entity Cards are intentionally **minimal**. They only declare MCP associations, not business metadata.

Business information (name, address, hours, etc.) should live on the entity's website using existing standards (Schema.org, Open Graph, etc.). EDP registries enrich entries by crawling these sources.

This separation ensures:
- **Single responsibility**: Entity Cards do one thing well
- **No sync issues**: Business info stays current on the source website
- **Low friction**: Easy to publish and maintain

### 2.2 Location

Entity Cards SHOULD be published at:

```
https://{entity-domain}/.well-known/entity-card.json
```

This follows [RFC 8615](https://tools.ietf.org/html/rfc8615) for well-known URIs.

### 2.3 Schema

```json
{
  "schema_version": "0.1.0",
  "domain": "string (required)",
  "mcps": [
    {
      "provider": "string (required)",
      "endpoint": "string (required, URL)",
      "entity_id": "string (optional)",
      "capabilities": ["string (optional)"],
      "priority": "number (optional, default: 0)",
      "verification": {
        "method": "string (optional)",
        "signature": "string (optional)",
        "issued_at": "string (optional, ISO 8601)",
        "expires_at": "string (optional, ISO 8601)"
      }
    }
  ]
}
```

### 2.4 Domain Validation

The `domain` field MUST match the domain hosting the Entity Card.

For example, if the Entity Card is hosted at:
```
https://lepetitzinc.fr/.well-known/entity-card.json
```

Then the `domain` field MUST be `lepetitzinc.fr`.

Registries SHOULD reject Entity Cards where the domain does not match the hosting domain. This prevents impersonation attacks where a malicious site claims to be another business.

### 2.5 Field Definitions

#### Root Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema_version` | string | Yes | Version of the Entity Card schema (e.g., "0.1.0") |
| `domain` | string | Yes | The domain publishing this Entity Card |
| `mcps` | array | Yes | List of MCP associations |

#### MCP Entry Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | string | Yes | Identifier of the MCP provider |
| `endpoint` | string | Yes | MCP server endpoint URL (must be HTTPS) |
| `entity_id` | string | No | Entity's identifier at this provider |
| `capabilities` | array | No | List of capability identifiers |
| `priority` | number | No | Priority for capability conflicts (higher = preferred) |
| `verification` | object | No | Verification information |

### 2.6 Capabilities

Capabilities describe what actions are possible via an MCP. Recommended values:

**Booking & Scheduling**
- `reservations` — Book appointments or tables
- `availability` — Check available slots
- `cancellation` — Cancel bookings

**Commerce**
- `ordering` — Place orders
- `payments` — Process payments
- `catalog` — Browse products/services

**Information**
- `menu` — View menus/offerings
- `hours` — Operating hours
- `contact` — Contact information

**Communication**
- `messaging` — Send messages
- `notifications` — Receive updates

Capabilities are extensible. Providers may define custom capabilities using a namespaced format: `provider:capability` (e.g., `acme:custom-feature`).

## 3. MCP Provider Registration

### 3.1 Overview

MCP providers can register entities they serve with EDP registries, enabling bulk discovery without requiring each entity to publish its own Entity Card.

### 3.2 Registration Payload

```json
{
  "provider": {
    "id": "string (required)",
    "name": "string (required)",
    "endpoint": "string (required, URL)",
    "public_key": "string (optional, PEM)",
    "capabilities": ["string"]
  },
  "entities": [
    {
      "entity_id": "string (required)",
      "domain": "string (optional)",
      "name": "string (required)",
      "category": "string (optional)",
      "location": {
        "city": "string (optional)",
        "country": "string (optional, ISO 3166-1 alpha-2)",
        "coordinates": {
          "lat": "number",
          "lng": "number"
        }
      },
      "capabilities": ["string"]
    }
  ],
  "signature": "string (optional)"
}
```

Note: Provider registration includes entity metadata because the provider is the authoritative source for entities they serve. This is different from Entity Cards, which only declare associations.

### 3.3 Trust Model

**Important**: Provider Registration is based on trust. A malicious provider could falsely claim to serve entities it doesn't actually serve. No technical mechanism can prevent this.

Mitigations:
- Registries SHOULD only accept registrations from verified, trusted providers (business partnerships)
- For higher assurance, registries SHOULD cross-reference with Entity Cards published by businesses
- Provider fraud is ultimately a legal/contractual issue, not a technical one

This is similar to Certificate Authorities in TLS: if a CA lies, the system fails. The protection is reputational and legal, not cryptographic.

## 4. Verification Levels

### 4.1 Overview

| Level | Source | What it proves |
|-------|--------|----------------|
| 0 | Provider Registration only | Provider claims to serve this entity |
| 1 | Entity Card only | Business claims to delegate to this MCP |
| 2 | Both match | Mutual agreement between business and provider |

### 4.2 Level 0: Provider Claim

The entity appears in the registry via Provider Registration only. No Entity Card exists.

**Trust model**: Trust the provider entirely.

**Use case**: Bulk onboarding, low-risk interactions, bootstrapping.

**Risk**: Provider could falsely claim entities.

### 4.3 Level 1: Entity Claim

The business publishes an Entity Card at `/.well-known/entity-card.json`.

**Trust model**: Trust the business. Publishing to `.well-known/` proves domain control—no additional verification needed.

**Use case**: Businesses without provider partnerships, self-hosted MCPs.

**Risk**: Business could claim to delegate to an MCP that doesn't actually serve them (MCP will reject unauthorized requests).

### 4.4 Level 2: Dual Attestation

Both sources agree:
- Provider Registration declares the entity, AND
- Entity Card published by business points to the same provider

**Trust model**: Both parties independently confirm the relationship. An attacker would need to compromise both sides.

**Use case**: Production, bookings, sensitive actions.

**How registries verify**:
1. Provider registers entity with `domain: "lepetitzinc.fr"`
2. Registry fetches `https://lepetitzinc.fr/.well-known/entity-card.json`
3. Entity Card lists the same provider → Level 2 confirmed

### 4.5 Optional: Cryptographic Signature

For additional assurance (non-repudiation, legal evidence), the provider can sign a JWT that the business publishes in their Entity Card.

This is **optional** for Level 2 but recommended for:
- Payment processing
- Legal compliance
- Audit trails

#### JWT Format

```json
{
  "header": {
    "alg": "ES256",
    "typ": "JWT",
    "kid": "provider-key-2025"
  },
  "payload": {
    "iss": "provider-id",
    "sub": "entity-domain.com",
    "entity_id": "entity-id-at-provider",
    "capabilities": ["reservations", "menu"],
    "iat": 1735344000,
    "exp": 1766880000
  }
}
```

#### Entity Card with Signature

```json
{
  "schema_version": "0.1.0",
  "domain": "lepetitzinc.fr",
  "mcps": [
    {
      "provider": "booking-provider",
      "endpoint": "https://mcp.booking-provider.com",
      "entity_id": "lpz-paris-75006",
      "verification": {
        "method": "signed_jwt",
        "signature": "eyJhbGciOiJFUzI1NiIs...",
        "issued_at": "2025-12-28T00:00:00Z",
        "expires_at": "2026-12-28T00:00:00Z"
      }
    }
  ]
}
```

#### Verification Process

The registry:
1. Fetches the provider's public key (from registry or JWKS endpoint)
2. Verifies the JWT signature
3. Checks `iss` matches the declared provider
4. Checks `sub` matches the entity's domain
5. Checks expiration

## 5. Resolution API

EDP registries SHOULD implement these endpoints:

### 5.1 Resolve by Query

```http
GET /v1/resolve?query={search_query}&location={location}
```

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Search query (name, keywords) |
| `location` | string | No | Location context (city, coordinates) |
| `capabilities` | string | No | Filter by capabilities (comma-separated) |
| `min_verification` | number | No | Minimum verification level (0-2) |

**Response**:
```json
{
  "results": [
    {
      "entity": {
        "name": "Le Petit Zinc",
        "domain": "lepetitzinc.fr",
        "category": "restaurant",
        "location": {
          "address": "12 rue de Rennes, 75006 Paris",
          "coordinates": { "lat": 48.8534, "lng": 2.3328 }
        },
        "verification_level": 2
      },
      "mcps": [
        {
          "provider": "booking-provider",
          "endpoint": "https://mcp.booking-provider.com",
          "entity_id": "lpz-paris-75006",
          "capabilities": ["reservations", "menu"],
          "verification": {
            "level": 2,
            "method": "signed_jwt",
            "valid": true,
            "expires_at": "2026-12-28T00:00:00Z"
          }
        }
      ],
      "confidence": 0.95
    }
  ]
}
```

Note: The `entity` object in the response contains enriched metadata from the registry, not from the Entity Card itself.

### 5.2 Resolve by Domain

```http
GET /v1/resolve/domain/{domain}
```

Returns the entity and MCPs for a specific domain.

### 5.3 Find Nearby

```http
GET /v1/nearby?lat={latitude}&lng={longitude}&radius={meters}
```

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `lat` | number | Yes | Latitude |
| `lng` | number | Yes | Longitude |
| `radius` | number | No | Radius in meters (default: 1000) |
| `capabilities` | string | No | Filter by capabilities |

## 6. Security Considerations

### 6.1 Entity Impersonation

An attacker could publish an Entity Card claiming to be a legitimate business.

**Mitigations**:
- Verification levels (Section 4)
- MCP-side authorization (the MCP rejects unauthorized requests)
- Registry moderation

### 6.2 MCP Impersonation

An attacker could claim a fake MCP endpoint.

**Mitigations**:
- Bidirectional signatures (Level 2)
- Provider registration with EDP registries
- HTTPS only for endpoints

### 6.3 Stale Data

Entity Cards may become outdated.

**Mitigations**:
- Regular crawling by registries
- Provider-side updates via registration API
- Expiration on signed claims

### 6.4 Privacy

Entity Cards are public by design (they're published on websites).

**Recommendations**:
- Only declare public MCP associations
- Sensitive entity_ids can be opaque tokens

## 7. Implementation Notes

### 7.1 For Entities

1. Create `/.well-known/entity-card.json`
2. List MCPs you delegate to
3. (Optional) Obtain signed claims from providers for Level 2 verification

### 7.2 For MCP Providers

1. Register with EDP registries
2. Provide a public key for signature verification
3. Generate signed JWTs for your customers
4. Keep entity lists updated via registration API

### 7.3 For EDP Registries

1. Crawl `.well-known/entity-card.json` files
2. Accept provider registrations
3. Enrich entries with business metadata (Schema.org, Google Business, etc.)
4. Implement verification for all levels
5. Provide resolution APIs

### 7.4 For AI Agents

1. Query EDP registries to resolve entities
2. Check verification levels based on action sensitivity
3. Cache results appropriately
4. Handle multiple MCPs by capability matching

## Appendix A: Examples

### A.1 Minimal Entity Card

```json
{
  "schema_version": "0.1.0",
  "domain": "mybusiness.com",
  "mcps": [
    {
      "provider": "booking-provider",
      "endpoint": "https://mcp.booking-provider.com",
      "entity_id": "mybusiness-123"
    }
  ]
}
```

### A.2 Entity Card with Multiple MCPs

```json
{
  "schema_version": "0.1.0",
  "domain": "lepetitzinc.fr",
  "mcps": [
    {
      "provider": "booking-provider",
      "endpoint": "https://mcp.booking-provider.com",
      "entity_id": "lpz-paris-75006",
      "capabilities": ["reservations", "availability", "menu"],
      "priority": 10
    },
    {
      "provider": "delivery-provider",
      "endpoint": "https://mcp.delivery-provider.com",
      "entity_id": "del-lpz-75006",
      "capabilities": ["ordering", "delivery_tracking"],
      "priority": 5
    }
  ]
}
```

### A.3 Entity Card with Level 2 Verification

```json
{
  "schema_version": "0.1.0",
  "domain": "lepetitzinc.fr",
  "mcps": [
    {
      "provider": "booking-provider",
      "endpoint": "https://mcp.booking-provider.com",
      "entity_id": "lpz-paris-75006",
      "capabilities": ["reservations", "menu"],
      "verification": {
        "method": "signed_jwt",
        "signature": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...",
        "issued_at": "2025-12-28T00:00:00Z",
        "expires_at": "2026-12-28T00:00:00Z"
      }
    }
  ]
}
```

## Appendix B: Changelog

### v0.1.0 (2025-12-28)
- Initial draft specification
- Minimal Entity Card format (no business metadata)
- MCP Provider registration
- Verification levels (0-2)
- Resolution API
