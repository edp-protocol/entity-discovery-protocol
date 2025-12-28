# Entity Discovery Protocol (EDP)

> **A standard for AI agents to discover which MCPs serve a given entity.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1.0-blue.svg)](SPEC.md)

## The Problem

The Model Context Protocol (MCP) enables AI agents to interact with services. But there's a missing layer:

- **MCP answers**: "How do I talk to this server?"
- **Server Cards (SEP-1649) answer**: "What can this MCP server do?"
- **Entity Discovery Protocol answers**: "Which MCPs serve this business?"

A restaurant doesn't run its own MCP server—it *delegates to* third-party MCP providers (booking, delivery, payments) to become actionable by AI agents. When an agent wants to help a user book a table at "Le Petit Zinc", it needs to discover that this restaurant delegates reservations to a booking provider.

```
User: "Book a table at Le Petit Zinc for tonight"
Agent: → Which MCP handles Le Petit Zinc?
       → Query EDP Registry: resolve("Le Petit Zinc", "Paris")
       → Response: Use mcp.booking-provider.com with entity_id "lpz-paris-75006"
       → Agent calls Booking Provider MCP → Booking complete
```

## How It Works

### 1. Entity Cards

Entities publish an `entity-card.json` declaring which MCPs they delegate to:

```
https://lepetitzinc.fr/.well-known/entity-card.json
```

```json
{
  "schema_version": "0.1.0",
  "domain": "lepetitzinc.fr",
  "mcps": [
    {
      "provider": "booking-provider",
      "endpoint": "https://mcp.booking-provider.com",
      "entity_id": "lpz-paris-75006",
      "capabilities": ["reservations", "menu", "availability"]
    }
  ]
}
```

Entity Cards are intentionally minimal—they only declare MCP associations. Business metadata (name, address, hours) lives on the website itself (via Schema.org, etc.) and is enriched by registries.

### 2. EDP Registries

Registries crawl Entity Cards and provide resolution APIs for agents. They enrich entries with business metadata from external sources (Schema.org, Google Business, etc.).

### 3. Verification Levels

| Level | Source | Trust |
|-------|--------|-------|
| **0** | Provider Registration only | Trust provider |
| **1** | Entity Card only | Trust business |
| **2** | Both match | Mutual agreement |

Level 2 can optionally include a cryptographic signature (JWT) for non-repudiation in sensitive use cases (payments, legal).

## Specification

📄 **[Full Specification (SPEC.md)](SPEC.md)**

The specification covers:
- Entity Card format and schema
- MCP Provider registration
- Verification levels and cryptographic proofs
- Resolution API endpoints
- Security considerations

## Schemas

- [entity-card.schema.json](schemas/entity-card.schema.json) — Validate Entity Cards
- [provider-registration.schema.json](schemas/provider-registration.schema.json) — Validate provider bulk registrations

## Examples

- [minimal.json](examples/minimal.json) — Simplest possible Entity Card
- [with-verification.json](examples/with-verification.json) — Entity Card with Level 2 verification
- [multi-mcp.json](examples/multi-mcp.json) — Entity with multiple MCP providers
- [provider-registration.json](examples/provider-registration.json) — Bulk provider registration

## Relationship to MCP

EDP is a **complementary** protocol to MCP, born from [discussions in the MCP community](https://discord.com/channels/1358869848138059966/1452619498749165660). It addresses a layer MCP intentionally doesn't cover: entity-to-MCP discovery.

This is not a fork or competition—MCP defines communication, EDP defines discovery. We welcome feedback from the MCP community and are open to integration if there's interest.

```
┌─────────────────────────────────────────────────────────────────┐
│  MCP (Model Context Protocol)                                   │
│  "How agents communicate with servers"                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ describes servers via
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Server Cards (SEP-1649)                                        │
│  "What an MCP server can do"                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ discovered via
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Entity Discovery Protocol (EDP)                                │
│  "Which MCP serves this business"                               │
└─────────────────────────────────────────────────────────────────┘
```

## Implementations

EDP is a protocol specification. To use it, you need:

- **A Registry**: Indexes entities and provides resolution APIs
- **MCP Providers**: Register their served entities
- **Entities**: Publish Entity Cards on their domains

See [docs/implementers-guide.md](docs/implementers-guide.md) for implementation guidance.

*If you've implemented EDP, open a PR to add your implementation here.*

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License — see [LICENSE](LICENSE)

## Links

- [Full Specification](SPEC.md)
- [MCP Protocol](https://modelcontextprotocol.io)
- [SEP-1649: Server Cards](https://github.com/modelcontextprotocol/specification/discussions/1649)
