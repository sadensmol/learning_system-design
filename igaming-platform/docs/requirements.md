# Multi-Region iGaming Platform — Requirements

## Business Context

iGaming platform (similar to PokerStars, DraftKings) that needs to serve players globally. Currently single-region (AWS), needs to expand to 3-5 regions while maintaining a globally consistent player ledger. Architecture follows proven patterns from major iGaming operators.

## Actors

- **Player**: plays games from any region, can travel/switch regions, has a single ledger account
- **Game Service**: runs game sessions locally in the nearest region
- **Ledger Service**: manages player balances, transactions — must be strongly consistent globally
- **Platform Operator**: deploys, monitors, and maintains the system across regions

## Key Constraints

1. **Strong consistency for ledger**: no double-spend, player balance must be accurate regardless of which region they connect to
2. **Region-local game sessions**: game logic runs in the player's nearest region
3. **Single ledger identity**: player A in region A and region B sees the same balance
4. **Redis is cache + distributed locking only**: not a primary data store

## Infrastructure

- **Compute**: Kubernetes (AWS EKS) per region
- **Database**: PostgreSQL (AWS RDS / Aurora)
- **Cache**: Redis (custom install within each K8s cluster)
- **CDN / Frontend Hosting**: AWS Amplify (hosts static JS clients + videos, CloudFront under the hood)
- **Cloud provider**: AWS

## Non-Functional Requirements

- **Latency**: game sessions must be low-latency (region-local); ledger operations can tolerate moderate latency for strong consistency
- **Availability**: system should survive partial failures; full region outage should not lose data
- **Scalability**: 5-15 microservices, 3-5 regions
- **Maintainability**: single codebase, consistent deployment across regions
- **Observability**: centralized logging, metrics, tracing across all regions

## Hotspots (Decisions Needed)

- Database topology for strong consistency across regions (single-writer vs multi-writer)
- How players are routed to the correct region
- How ledger service handles cross-region requests
- Deployment strategy (GitOps, progressive rollout)
- Failover strategy when a region goes down
