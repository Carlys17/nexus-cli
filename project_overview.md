# Nexus CLI Project Overview

## Project Summary
This is the **Nexus CLI** - a high-performance command-line interface for contributing proofs to the Nexus network, which is a global distributed prover network for verifiable computation. The project is written in Rust and aims to power the "Verifiable Internet."

## Project Structure

### Root Directory
```
├── .git/                    # Git repository
├── .github/                 # CI/CD workflows and issue templates
├── assets/                  # Project assets (images)
├── clients/                 # Client implementations
├── programs/                # Example programs
├── proto/                   # Protocol Buffer definitions
├── public/                  # Firebase hosting files
├── tests/                   # Test files
├── firebase.json            # Firebase hosting configuration
├── README.md               # Main documentation
├── CONTRIBUTING.md         # Contribution guidelines
├── LICENSE-APACHE          # Apache 2.0 license
└── LICENSE-MIT             # MIT license
```

### Key Technologies
- **Language**: Rust (edition 2024, rustc 1.85+)
- **CLI Framework**: Clap for command-line parsing
- **UI**: Ratatui for terminal user interface
- **Async Runtime**: Tokio
- **Networking**: Reqwest with rustls-tls
- **Serialization**: Serde JSON, Postcard, Prost (Protocol Buffers)
- **Cryptography**: Ed25519-dalek for signing
- **Hosting**: Firebase hosting for distribution

## Main Features

### Core Functionality
1. **Proof Generation**: Generate and submit proofs to the Nexus network
2. **User Registration**: Register users with wallet addresses
3. **Node Management**: Register and manage prover nodes
4. **Multi-threaded Proving**: Configurable number of worker threads (1-8)
5. **Terminal UI**: Interactive dashboard for monitoring
6. **Headless Mode**: Command-line only operation

### CLI Commands
- `nexus-cli start [--node-id] [--headless] [--max-threads]` - Start proving
- `nexus-cli register-user --wallet-address <ADDRESS>` - Register user
- `nexus-cli register-node [--node-id]` - Register/link a node
- `nexus-cli logout` - Clear credentials

### Installation Methods
1. **Precompiled Binary (Recommended)**:
   ```bash
   curl https://cli.nexus.xyz/ | sh
   ```
2. **Non-Interactive Installation**:
   ```bash
   curl -sSf https://cli.nexus.xyz/ -o install.sh
   NONINTERACTIVE=1 ./install.sh
   ```

## Source Code Structure

### Main Components (`clients/cli/src/`)
- `main.rs` (249 lines) - Main entry point and CLI handling
- `prover.rs` (240 lines) - Core proving logic
- `prover_runtime.rs` (181 lines) - Worker management
- `register.rs` (253 lines) - User/node registration
- `config.rs` (301 lines) - Configuration management
- `orchestrator/` - Network communication
- `ui/` - Terminal user interface
- `workers/` - Worker thread management

### Key Dependencies
- **nexus-sdk**: Core Nexus zkVM functionality
- **clap**: Command-line parsing
- **tokio**: Async runtime
- **ratatui**: Terminal UI
- **reqwest**: HTTP client
- **ed25519-dalek**: Cryptographic signing
- **serde**: Serialization

## Network Information

### Testnet History
- **Testnet 0**: October 8-28, 2024
- **Testnet I**: December 9-13, 2024
- **Testnet II**: February 18-22, 2025
- **Devnet**: February 22 - June 20, 2025
- **Testnet III**: Currently ongoing

### Configuration
- Config stored in `~/.nexus/config.json`
- Supports different environments (production, staging, development)
- Node ID and user credentials management

## Development Setup

### Prerequisites
- Rust (latest stable)
- Protobuf compiler
- Git

### Building
```bash
# Standard build
cargo build

# With proto compilation
cargo build --features build_proto

# Run
cargo run
```

### Code Quality
- `cargo fmt --check` - Code formatting
- `cargo clippy -- -D warnings` - Linting
- `cargo audit` - Security vulnerabilities

## CI/CD & Distribution

### GitHub Workflows
- `ci.yml` - Continuous integration
- `release.yml` - Automated releases with binaries
- Firebase hosting workflows for deployment

### Release Process
1. Update version in Cargo.toml
2. Create annotated git tag: `git tag -a v0.1.2 -m "Release v0.1.2"`
3. Push tag: `git push origin v0.1.2`
4. Automated workflow builds binaries and Docker image

### Distribution
- Firebase hosting serves installation script at https://cli.nexus.xyz/
- Precompiled binaries for multiple platforms
- Docker image builds

## Example Programs
Located in `clients/cli/examples/src/bin/`:
- `fib.rs` - Fibonacci sequence
- `fact.rs` - Factorial calculation
- `keccak.rs` - Keccak hashing
- `palindromes.rs` - Palindrome detection
- `galeshapley.rs` - Gale-Shapley algorithm
- `lambda_calculus.rs` - Lambda calculus interpreter

## Contributing
- Dual-licensed under Apache 2.0 and MIT
- Active community on Discord
- Comprehensive contribution guidelines
- Code of conduct following Rust standards

## Current Status
- Version 0.9.0
- Active development with ongoing testnet
- Production-ready with optimized release builds
- Cross-platform support (Linux, macOS, Windows)

## Links
- **Website**: https://nexus.xyz/
- **Beta Network**: https://beta.nexus.xyz/
- **App**: https://app.nexus.xyz
- **Documentation**: https://docs.nexus.xyz/
- **Discord**: Discord community available
- **GitHub**: https://github.com/nexus-xyz/nexus-cli