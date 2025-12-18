# Server Infrastructure Documentation

This repository contains comprehensive documentation and setup guides for a self-hosted server infrastructure.

## Server Specifications

- **CPU**: High-performance CPU
- **RAM**: 32GB ECC
- **Storage**: 2TB
- **OS**: Fedora 43 Server
- **Domains**:
  - antoinelb.fr
  - vaultofsuraya.com
- **DNS Management**: Cloudflare

## Infrastructure Components

### Core Services

1. **Coolify** - Application deployment and hosting platform
2. **Remote Development Environment** - Cloud-based coding solution
3. **VPN** - Secure remote access and networking
4. **Reverse Proxy** - Automatic SSL and traffic routing
5. **Monitoring & Backups** - System observability and data protection

## Documentation Structure

- [`docs/01-vpn-solutions.md`](docs/01-vpn-solutions.md) - VPN comparison (Tailscale vs Headscale)
- [`docs/02-reverse-proxy.md`](docs/02-reverse-proxy.md) - Reverse proxy solutions (Caddy, Traefik, Nginx)
- [`docs/03-coolify-setup.md`](docs/03-coolify-setup.md) - Coolify installation and configuration
- [`docs/04-remote-coding.md`](docs/04-remote-coding.md) - Remote development environment options
- [`docs/05-monitoring-backups.md`](docs/05-monitoring-backups.md) - Monitoring and backup strategies
- [`docs/06-security-hardening.md`](docs/06-security-hardening.md) - Security best practices for Fedora 43
- [`docs/07-architecture.md`](docs/07-architecture.md) - Overall system architecture

## Quick Start

1. Review the architecture document to understand the overall setup
2. Follow the security hardening guide first
3. Set up VPN for secure access
4. Install and configure the reverse proxy
5. Deploy Coolify
6. Set up monitoring and backups
7. Configure your remote development environment

## Prerequisites

- Root access to Fedora 43 server
- Cloudflare account with DNS management
- GitHub account (for git operations)
- Basic understanding of Linux system administration

## Contributing

This is a personal infrastructure setup, but feel free to adapt these guides for your own needs.

## License

Documentation is provided as-is for personal use.
