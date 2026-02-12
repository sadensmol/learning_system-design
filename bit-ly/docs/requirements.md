# Requirements: URL Shortener Service

## Business Goals

A URL shortening service that converts long URLs into short, shareable links. When users visit the short URL, they're redirected to the original URL. The system provides analytics on link usage.

## Key Actors

| Actor | Description | Primary Goals |
|-------|-------------|---------------|
| Anonymous User | Visitor without account | Quickly create short links |
| Registered User | Account holder | Manage URLs, view analytics, use custom aliases |
| API Consumer | Developer using API | Programmatic URL shortening |
| System Admin | Operations team | Monitor health, manage abuse |

## Constraints

### Technical
- Short URLs: 7 characters (3.5 trillion combinations)
- Immutable: once created, short URLs don't change
- Cooldown on deleted URLs (prevent hijacking)
- Read:Write ratio ~100:1 (read-heavy)

### Scale Targets

| Level | URLs | Redirects/sec |
|-------|------|---------------|
| Startup | 10M | 1K |
| Growth | 100M | 10K |
| Enterprise | 1B+ | 100K |

## Success Criteria

| Metric | Target |
|--------|--------|
| Redirect Availability | 99.9% |
| Redirect Latency (p99) | < 100ms |
| URL Creation Latency | < 500ms |
| Cache Hit Rate | > 95% |

## External Systems

| System | Purpose | Integration |
|--------|---------|-------------|
| CDN | Edge caching | HTTP proxy |
| Safe Browsing API | Malware detection | REST API |
| GeoIP Service | Location from IP | Library |
| Message Queue | Async analytics | Event streaming |

## Open Questions / Hotspots

- ? Key generation: Hash-based vs Counter-based
- ? Database choice at scale: SQL vs NoSQL transition point
- ? Custom domain support: SSL certificate management
- ? Data retention: Analytics retention policy
