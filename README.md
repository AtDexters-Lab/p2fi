# p2fi

An open architecture for privacy-preserving access to cloud-deployed frontier AI.

## Overview

Current access to frontier AI models requires identity-linked accounts, giving providers access to user identities, metadata, and plaintext data. p2fi shifts this from "privacy by policy" to "privacy by design" — mathematically and architecturally guaranteeing that providers learn nothing about the user's identity, network location, or (eventually) the semantic content of their data.

## Architecture Documents

| Document | Status | Description |
|----------|--------|-------------|
| [Problem Statement](docs/problem-statement.md) | Final | Defines the core architectural conflict and the three-layer objective |
| [L1 & L2 Architecture](docs/architecture-l1-l2.md) | Final | Anonymous authorization (ZK-Ticket Protocol) and network unlinkability (OHTTP / onion routing) |

### Layers

The architecture is organized into three layers:

1. **Layer 1 — Anonymous Authorization:** Proving the right to consume resources without revealing identity. Solved via blind BBS+ credential issuance, ZK range proofs, and indexed nullifiers mediated by an independent Authorization Escrow.
2. **Layer 2 — Network Unlinkability:** Severing the connection between the user's network location and their requests. Solved via OHTTP (RFC 9458) or multi-hop onion routing through a federated relay network.
3. **Layer 3 — Data-in-Use Confidentiality:** Ensuring prompts and completions remain encrypted during inference. *Not yet specified.*

## Contributing

This is a documentation-only repository. Contributions to the architecture — new layer designs, threat analysis, alternative approaches, or refinements to existing documents — are welcome.

## License

See [LICENSE](LICENSE) for details.
