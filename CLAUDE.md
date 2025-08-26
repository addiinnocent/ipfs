# IPFS Pinning Platform - Claude Code Context

## Project Overview

This project is a comprehensive IPFS (InterPlanetary File System) pinning platform, similar to Pinata, built as a monorepo architecture. It provides file hosting, pinning services, and gateway access through a distributed IPFS infrastructure with enterprise-grade reliability and monitoring.

## Quick Start Commands

```bash
# Setup development environment
npm install
cd infrastructure/ipfs && docker-compose up -d && cd ../..
docker-compose up -d

# Development
npm run dev              # Start all services
npm run dev:api         # Backend only
npm run dev:web         # Frontend only

# IPFS Management
npm run ipfs:status     # Check cluster health
npm run ipfs:backup     # Backup pin data
npm run ipfs:logs       # View IPFS logs

# Testing & Quality
npm run test            # Run all tests
npm run lint           # Lint codebase
npm run type-check     # TypeScript validation

# Production
npm run build          # Build for production
npm run docker:build   # Build Docker images
npm run deploy         # Deploy to production
```

## Architecture Overview

### Core Services
- **API Service** (`apps/api`): RESTful API with authentication, file management, and pin operations
- **Web Application** (`apps/web`): Next.js dashboard for file management and account administration
- **IPFS Manager** (`services/ipfs-manager`): Kubo node management and cluster coordination
- **File Processor** (`services/file-processor`): File validation, optimization, and metadata extraction
- **Notification Service** (`services/notification`): Webhook and email notification system

### IPFS Infrastructure
- **Kubo Nodes**: Official IPFS implementation for content storage and retrieval
- **IPFS Cluster**: Multi-node coordination and pin replication management
- **Custom Gateway**: High-performance content delivery with CDN integration
- **Monitoring Stack**: Prometheus, Grafana, and custom IPFS-specific metrics

### Shared Libraries
- **@packages/shared**: Common types, utilities, and constants
- **@packages/api-client**: Type-safe API client library
- **@packages/db**: Database models, migrations, and ORM utilities

## Development Environment

### Prerequisites
- Node.js 18+
- Docker & Docker Compose
- PostgreSQL 15+
- Redis 7+

### IPFS Development Stack
```yaml
# infrastructure/ipfs/docker-compose.yml
services:
  ipfs-node-1:
    image: ipfs/kubo:v0.36.0
    ports: ["4001:4001", "5001:5001", "8080:8080"]
  
  ipfs-cluster:
    image: ipfs/ipfs-cluster:v1.0.8
    ports: ["9094:9094", "9095:9095", "9096:9096"]
  
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
```

### Key Configuration Files
- `docker-compose.yml`: Development services orchestration
- `infrastructure/ipfs/configs/kubo-config.json`: IPFS node configuration
- `infrastructure/ipfs/configs/cluster-config.json`: Cluster coordination settings
- `infrastructure/kubernetes/`: Production Kubernetes manifests

## Database Schema

### Core Entities
```sql
-- Users with API key management
users (id, email, password_hash, api_key, plan, storage_limit)

-- File metadata and IPFS mapping
files (id, user_id, ipfs_hash, original_name, content_type, size, metadata)

-- Pin management with cluster tracking
pins (id, file_id, user_id, ipfs_hash, status, pin_date, replication_factor)

-- Multi-node pin status tracking
pin_nodes (id, pin_id, node_id, status, pinned_at, last_checked)
```

## API Endpoints

### Core Operations
```http
POST   /api/v1/auth/login           # User authentication
POST   /api/v1/files/upload         # File upload and pinning
GET    /api/v1/files                # List user files
DELETE /api/v1/files/:hash          # Unpin and remove file
GET    /api/v1/pins                 # List active pins
POST   /api/v1/pins/:hash           # Pin existing IPFS content
GET    /api/v1/pins/:hash/status    # Check pin status across nodes
```

### Gateway Access
```http
GET    /ipfs/:hash                  # Content retrieval
GET    /ipfs/:hash/:filename        # Named file access  
HEAD   /ipfs/:hash                  # Metadata only
```

## Monitoring & Observability

### IPFS-Specific Metrics
```yaml
# Key metrics tracked in Prometheus
ipfs_datastore_size_bytes          # Node storage usage
ipfs_peers_connected              # Network connectivity
ipfs_pins_total                   # Total pinned objects
ipfs_gateway_requests_total       # Gateway traffic
ipfs_cluster_pins_status          # Pin replication health
```

### Custom Dashboards
- **Node Health**: CPU, memory, disk, peer connections
- **Pin Status**: Replication health, pin queue, failure rates
- **Gateway Performance**: Response times, cache hit rates, bandwidth
- **Business Metrics**: User uploads, storage usage, API usage

## Security & Compliance

### Authentication & Authorization
- JWT-based API authentication
- API key management with rate limiting
- Role-based access control (RBAC)
- Multi-factor authentication support

### Data Protection
- Encryption at rest (database, file metadata)
- TLS 1.3 for all communications
- Content validation and virus scanning
- GDPR compliance features (data export, deletion)

### Infrastructure Security
- Private IPFS cluster networks
- Firewall rules and network segmentation
- Regular security audits and penetration testing
- SOC2 Type II compliance preparation

## Related Documentation

- **[ROADMAP.md](./ROADMAP.md)**: Detailed technical implementation phases, milestones, and resource planning
- **[BUSINESS.md](./BUSINESS.md)**: Business model, pricing strategy, competitive analysis, and market positioning
- **[API Documentation](./docs/api/openapi.yaml)**: Complete OpenAPI specification
- **[Architecture Guide](./docs/architecture/)**: Detailed system architecture and design decisions
- **[Deployment Guide](./docs/deployment/)**: Production deployment and infrastructure setup

## Development Workflow

### 1. Feature Development
```bash
# Create feature branch
git checkout -b feature/pin-optimization

# Make changes with proper testing
npm run test
npm run lint

# Commit with conventional commits
git commit -m "feat(pins): improve cluster pin distribution algorithm"
```

### 2. Testing Strategy
- **Unit Tests**: Service logic and utilities
- **Integration Tests**: API endpoints and database operations
- **E2E Tests**: Full user workflows
- **Load Tests**: IPFS cluster performance under stress

### 3. Deployment Pipeline
```bash
# Staging deployment
npm run build:staging
npm run test:e2e
npm run deploy:staging

# Production deployment (after approval)
npm run build:production
npm run deploy:production
```

## Performance Targets

### API Performance
- 95th percentile response time: <200ms
- Throughput: 1000+ requests/second
- Availability: 99.99% uptime SLA

### IPFS Operations
- Pin success rate: >99.5%
- Content retrieval: <500ms first byte
- Cluster sync time: <30 seconds
- Gateway cache hit rate: >85%

## Troubleshooting

### Common Issues
```bash
# IPFS node connectivity issues
docker exec ipfs-node-1 ipfs swarm peers

# Cluster pin status debugging  
curl http://localhost:9094/allocations

# Database connection issues
npm run db:health

# API service logs
docker-compose logs api

# Gateway performance debugging
curl -I http://localhost:8080/ipfs/QmHash
```

### Health Checks
- Node status: `GET /health/ipfs`
- Database: `GET /health/db` 
- Redis: `GET /health/cache`
- Cluster: `GET /health/cluster`

---

This project aims to provide enterprise-grade IPFS pinning services with the reliability and features needed for production workloads. See ROADMAP.md for implementation phases and BUSINESS.md for commercial strategy.