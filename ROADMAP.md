# IPFS Pinning Platform - Technical Roadmap

## Executive Summary

This roadmap outlines the 18-week technical implementation plan for building a production-ready IPFS pinning platform. The approach progresses from single-node prototyping to enterprise-grade multi-region clusters, with clear milestones, resource requirements, and success metrics.

## Implementation Phases Overview

| Phase | Duration | Focus | Key Deliverables |
|-------|----------|-------|------------------|
| **Phase 1** | Weeks 1-3 | Foundation & Prototyping | Single-node IPFS setup, basic API integration |
| **Phase 2** | Weeks 4-6 | Production-Ready Single Node | Robust configuration, monitoring, comprehensive API |
| **Phase 3** | Weeks 7-10 | Multi-Node Cluster | IPFS Cluster implementation, high availability |
| **Phase 4** | Weeks 11-14 | Scaling & Optimization | Performance tuning, advanced features |
| **Phase 5** | Weeks 15-18 | Enterprise Readiness | Multi-region, compliance, security hardening |

---

## Phase 1: Foundation & Prototyping (Weeks 1-3)

### Week 1: IPFS Fundamentals & Local Setup

**Objectives:**
- Establish deep understanding of IPFS concepts and operations
- Set up local development environment with Kubo
- Document content addressing and pinning mechanisms

**Technical Tasks:**
```bash
# Local Kubo installation and configuration
brew install ipfs
ipfs init
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["*"]'
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["PUT", "GET", "POST"]'
ipfs daemon

# Basic operations testing
ipfs add example.txt
ipfs pin add QmHash
ipfs cat QmHash
ipfs pin ls --type=recursive
```

**Research & Documentation:**
- IPFS HTTP RPC API comprehensive analysis
- Content Addressing vs Traditional URLs
- DHT (Distributed Hash Table) behavior study
- Gateway functionality and limitations
- Pin lifecycle management understanding

**Deliverables:**
- Local IPFS development environment
- IPFS operations cheat sheet
- Content addressing proof-of-concept
- Basic pin/unpin operations documentation

### Week 2: Containerized Single Node Setup

**Objectives:**
- Create reproducible Docker-based IPFS environment
- Establish baseline configuration for development
- Implement basic health monitoring

**Technical Implementation:**
```yaml
# infrastructure/ipfs/docker-compose.yml
version: '3.8'
services:
  ipfs:
    image: ipfs/kubo:v0.36.0
    container_name: ipfs-node-dev
    environment:
      - IPFS_PROFILE=server
    ports:
      - "4001:4001"     # P2P swarm
      - "5001:5001"     # HTTP API
      - "8080:8080"     # HTTP Gateway
    volumes:
      - ./data:/data/ipfs
      - ./configs/kubo-config.json:/data/ipfs/config
    restart: unless-stopped
```

**Configuration Management:**
```json
{
  "API": {
    "HTTPHeaders": {
      "Access-Control-Allow-Origin": ["*"],
      "Access-Control-Allow-Methods": ["PUT", "GET", "POST"]
    }
  },
  "Addresses": {
    "Swarm": ["/ip4/0.0.0.0/tcp/4001"],
    "API": "/ip4/0.0.0.0/tcp/5001",
    "Gateway": "/ip4/0.0.0.0/tcp/8080"
  },
  "Datastore": {
    "StorageMax": "10GB",
    "StorageGCWatermark": 90,
    "GCPeriod": "1h"
  }
}
```

**Health Monitoring Scripts:**
```bash
#!/bin/bash
# infrastructure/ipfs/scripts/health-check.sh
curl -f http://localhost:5001/api/v0/id > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "IPFS Node: HEALTHY"
    curl -s http://localhost:5001/api/v0/stats/repo
else
    echo "IPFS Node: UNHEALTHY"
    exit 1
fi
```

**Deliverables:**
- Containerized IPFS development environment
- Basic configuration management system
- Health check automation scripts
- Docker Compose orchestration setup

### Week 3: API Integration Prototype

**Objectives:**
- Build initial Node.js service for IPFS interaction
- Implement basic file upload and pin management
- Establish database schema for metadata storage

**IPFS Manager Service Structure:**
```
services/ipfs-manager/
├── src/
│   ├── controllers/
│   │   ├── upload.controller.ts
│   │   ├── pin.controller.ts
│   │   └── health.controller.ts
│   ├── services/
│   │   ├── ipfs.service.ts
│   │   ├── pin.service.ts
│   │   └── metadata.service.ts
│   ├── utils/
│   │   ├── ipfs-client.ts
│   │   ├── validators.ts
│   │   └── logger.ts
│   └── app.ts
├── package.json
└── Dockerfile
```

**Core API Implementation:**
```typescript
// services/ipfs-manager/src/services/ipfs.service.ts
import { create, IPFSHTTPClient } from 'ipfs-http-client'

export class IPFSService {
  private client: IPFSHTTPClient

  constructor() {
    this.client = create({
      host: process.env.IPFS_HOST || 'localhost',
      port: 5001,
      protocol: 'http'
    })
  }

  async addFile(file: Buffer, options?: any) {
    try {
      const result = await this.client.add(file, {
        pin: true,
        wrapWithDirectory: false,
        ...options
      })
      return result
    } catch (error) {
      throw new Error(`Failed to add file to IPFS: ${error.message}`)
    }
  }

  async pinHash(hash: string) {
    try {
      await this.client.pin.add(hash)
      return { hash, status: 'pinned' }
    } catch (error) {
      throw new Error(`Failed to pin hash ${hash}: ${error.message}`)
    }
  }

  async getStats() {
    try {
      const [id, stats, version] = await Promise.all([
        this.client.id(),
        this.client.stats.repo(),
        this.client.version()
      ])
      return { id, stats, version }
    } catch (error) {
      throw new Error(`Failed to get IPFS stats: ${error.message}`)
    }
  }
}
```

**Database Schema (PostgreSQL):**
```sql
-- Core tables for metadata management
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ipfs_hash VARCHAR(255) UNIQUE NOT NULL,
    original_name VARCHAR(500) NOT NULL,
    content_type VARCHAR(100),
    size_bytes BIGINT NOT NULL,
    upload_timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    metadata JSONB,
    INDEX(ipfs_hash),
    INDEX(upload_timestamp)
);

CREATE TABLE pins (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_id UUID REFERENCES files(id) ON DELETE CASCADE,
    ipfs_hash VARCHAR(255) NOT NULL,
    pin_timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pinning',
    metadata JSONB,
    INDEX(ipfs_hash),
    INDEX(status),
    INDEX(pin_timestamp)
);
```

**Deliverables:**
- Basic IPFS Manager service with core operations
- Database schema and migration scripts
- API endpoints for upload/pin/unpin operations
- Error handling and logging framework
- Unit tests for core functionality

---

## Phase 2: Production-Ready Single Node (Weeks 4-6)

### Week 4: Robust Node Configuration & Security

**Objectives:**
- Optimize IPFS configuration for production workloads
- Implement security hardening measures
- Establish backup and recovery procedures

**Production Kubo Configuration:**
```json
{
  "API": {
    "HTTPHeaders": {
      "Access-Control-Allow-Origin": ["https://yourdomain.com"],
      "Access-Control-Allow-Methods": ["GET", "POST"],
      "Access-Control-Allow-Credentials": ["true"]
    }
  },
  "Addresses": {
    "Swarm": [
      "/ip4/0.0.0.0/tcp/4001",
      "/ip6/::/tcp/4001",
      "/ip4/0.0.0.0/udp/4001/quic-v1",
      "/ip6/::/udp/4001/quic-v1"
    ],
    "Announce": [],
    "AppendAnnounce": [],
    "NoAnnounce": [
      "/ip4/10.0.0.0/ipcidr/8",
      "/ip4/172.16.0.0/ipcidr/12",
      "/ip4/192.168.0.0/ipcidr/16"
    ],
    "API": "/ip4/127.0.0.1/tcp/5001",
    "Gateway": "/ip4/127.0.0.1/tcp/8080"
  },
  "Discovery": {
    "MDNS": {
      "Enabled": false
    }
  },
  "Datastore": {
    "StorageMax": "100GB",
    "StorageGCWatermark": 85,
    "GCPeriod": "30m",
    "HashOnRead": true,
    "BloomFilterSize": 0
  },
  "Gateway": {
    "PathPrefixes": ["/ipfs", "/ipns"],
    "HTTPHeaders": {
      "Access-Control-Allow-Origin": ["*"],
      "Access-Control-Allow-Methods": ["GET"],
      "Cache-Control": ["public, max-age=29030400, immutable"]
    }
  },
  "Swarm": {
    "ConnMgr": {
      "LowWater": 600,
      "HighWater": 900,
      "GracePeriod": "20s"
    }
  }
}
```

**Security Hardening:**
```bash
# infrastructure/ipfs/security/firewall-rules.sh
#!/bin/bash

# Allow IPFS P2P traffic
ufw allow 4001/tcp
ufw allow 4001/udp

# Restrict API access to localhost only
ufw deny 5001/tcp
iptables -A INPUT -p tcp --dport 5001 -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 5001 -j DROP

# Allow Gateway access with rate limiting
ufw limit 8080/tcp

# Enable firewall
ufw --force enable
```

**Backup Strategy:**
```bash
#!/bin/bash
# infrastructure/ipfs/scripts/backup-datastore.sh

BACKUP_DIR="/backups/ipfs/$(date +%Y%m%d_%H%M%S)"
IPFS_DATA_DIR="/data/ipfs"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Stop IPFS service
docker stop ipfs-node

# Create compressed backup of datastore
tar -czf "$BACKUP_DIR/datastore.tar.gz" -C "$IPFS_DATA_DIR" blocks datastore

# Backup configuration
cp "$IPFS_DATA_DIR/config" "$BACKUP_DIR/config.json"

# Export pin list
ipfs pin ls --type=recursive > "$BACKUP_DIR/pinlist.txt"

# Restart IPFS service
docker start ipfs-node

# Upload to S3 (optional)
aws s3 cp "$BACKUP_DIR" s3://your-backup-bucket/ipfs/ --recursive

echo "Backup completed: $BACKUP_DIR"
```

**Deliverables:**
- Production-optimized IPFS configuration
- Security hardening implementation
- Automated backup and recovery procedures
- Performance tuning documentation
- Security audit checklist

### Week 5: Comprehensive API Services

**Objectives:**
- Implement complete file management API
- Add advanced pin operations and batch processing
- Implement rate limiting and authentication
- Add comprehensive error handling and logging

**Enhanced API Structure:**
```
apps/api/
├── src/
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── files.controller.ts
│   │   ├── pins.controller.ts
│   │   └── health.controller.ts
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   ├── rate-limit.middleware.ts
│   │   ├── validation.middleware.ts
│   │   └── error-handler.middleware.ts
│   ├── services/
│   │   ├── file.service.ts
│   │   ├── pin.service.ts
│   │   ├── user.service.ts
│   │   └── notification.service.ts
│   ├── models/
│   │   ├── User.ts
│   │   ├── File.ts
│   │   └── Pin.ts
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── validators.ts
│   │   └── constants.ts
│   └── app.ts
```

**Complete File Management API:**
```typescript
// apps/api/src/controllers/files.controller.ts
import { Request, Response } from 'express'
import { FileService } from '../services/file.service'
import { PinService } from '../services/pin.service'

export class FilesController {
  constructor(
    private fileService: FileService,
    private pinService: PinService
  ) {}

  async uploadFile(req: Request, res: Response) {
    try {
      const file = req.file
      const userId = req.user.id
      
      // Validate file
      this.validateFile(file)
      
      // Upload to IPFS
      const ipfsResult = await this.fileService.uploadToIPFS(file.buffer)
      
      // Save metadata
      const fileRecord = await this.fileService.create({
        userId,
        ipfsHash: ipfsResult.cid.toString(),
        originalName: file.originalname,
        contentType: file.mimetype,
        size: file.size,
        metadata: {
          uploadedAt: new Date(),
          userAgent: req.get('User-Agent')
        }
      })
      
      // Create pin record
      await this.pinService.createPin({
        fileId: fileRecord.id,
        userId,
        ipfsHash: ipfsResult.cid.toString(),
        status: 'pinned'
      })
      
      res.status(201).json({
        success: true,
        data: {
          id: fileRecord.id,
          hash: ipfsResult.cid.toString(),
          name: file.originalname,
          size: file.size,
          pinStatus: 'pinned'
        }
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }

  async listFiles(req: Request, res: Response) {
    try {
      const userId = req.user.id
      const { page = 1, limit = 20, status } = req.query
      
      const files = await this.fileService.findByUser(userId, {
        page: Number(page),
        limit: Number(limit),
        status
      })
      
      res.json({
        success: true,
        data: files.data,
        pagination: {
          page: files.page,
          limit: files.limit,
          total: files.total,
          pages: Math.ceil(files.total / files.limit)
        }
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }

  async deleteFile(req: Request, res: Response) {
    try {
      const { hash } = req.params
      const userId = req.user.id
      
      // Verify ownership
      const file = await this.fileService.findByHash(hash)
      if (!file || file.userId !== userId) {
        return res.status(404).json({
          success: false,
          error: 'File not found'
        })
      }
      
      // Unpin from IPFS
      await this.pinService.unpinHash(hash)
      
      // Delete from database
      await this.fileService.delete(file.id)
      
      res.json({
        success: true,
        message: 'File deleted successfully'
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }
}
```

**Rate Limiting & Authentication:**
```typescript
// apps/api/src/middleware/rate-limit.middleware.ts
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'
import { Redis } from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

export const createRateLimiter = (options: {
  windowMs: number
  max: number
  message: string
}) => {
  return rateLimit({
    store: new RedisStore({
      client: redis,
      prefix: 'rl:',
    }),
    windowMs: options.windowMs,
    max: options.max,
    message: {
      success: false,
      error: options.message
    },
    standardHeaders: true,
    legacyHeaders: false,
  })
}

// Usage
export const uploadRateLimit = createRateLimiter({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10, // 10 uploads per window
  message: 'Too many uploads, please try again later'
})

export const apiRateLimit = createRateLimiter({
  windowMs: 15 * 60 * 1000,
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later'
})
```

**Deliverables:**
- Complete RESTful API with CRUD operations
- Authentication and authorization system
- Rate limiting and request validation
- Comprehensive error handling
- API documentation (OpenAPI/Swagger)
- Integration tests for all endpoints

### Week 6: Monitoring & Alerting Infrastructure

**Objectives:**
- Implement comprehensive monitoring with Prometheus
- Create Grafana dashboards for IPFS metrics
- Set up alerting rules for critical events
- Implement structured logging and log aggregation

**Prometheus Configuration:**
```yaml
# infrastructure/monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'ipfs-node'
    static_configs:
      - targets: ['ipfs-node:5001']
    metrics_path: /debug/metrics/prometheus
    scrape_interval: 30s
    
  - job_name: 'api-service'
    static_configs:
      - targets: ['api:3000']
    metrics_path: /metrics
    scrape_interval: 15s
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

**IPFS-Specific Alert Rules:**
```yaml
# infrastructure/monitoring/prometheus/alert_rules.yml
groups:
  - name: ipfs_alerts
    rules:
      - alert: IPFSNodeDown
        expr: up{job="ipfs-node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "IPFS node is down"
          description: "IPFS node has been down for more than 1 minute"
          
      - alert: IPFSDatastoreFull
        expr: ipfs_datastore_usage_bytes / ipfs_datastore_size_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPFS datastore is 90% full"
          description: "Datastore usage is {{ $value }}% of capacity"
          
      - alert: IPFSPeerCountLow
        expr: ipfs_p2p_peers < 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "IPFS peer count is low"
          description: "Only {{ $value }} peers connected"
          
      - alert: IPFSPinQueueHigh
        expr: ipfs_pins_queued > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPFS pin queue is high"
          description: "{{ $value }} pins are queued for processing"
```

**Grafana Dashboard Configuration:**
```json
{
  "dashboard": {
    "title": "IPFS Node Overview",
    "panels": [
      {
        "title": "Node Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"ipfs-node\"}",
            "legendFormat": "Node Status"
          }
        ]
      },
      {
        "title": "Datastore Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "ipfs_datastore_usage_bytes",
            "legendFormat": "Used Bytes"
          },
          {
            "expr": "ipfs_datastore_size_bytes",
            "legendFormat": "Total Bytes"
          }
        ]
      },
      {
        "title": "Connected Peers",
        "type": "graph",
        "targets": [
          {
            "expr": "ipfs_p2p_peers",
            "legendFormat": "Peer Count"
          }
        ]
      },
      {
        "title": "Gateway Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(ipfs_gateway_requests_total[5m])",
            "legendFormat": "Requests/sec"
          }
        ]
      }
    ]
  }
}
```

**Structured Logging Implementation:**
```typescript
// apps/api/src/utils/logger.ts
import winston from 'winston'
import { ElasticsearchTransport } from 'winston-elasticsearch'

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'ipfs-api',
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    new winston.transports.File({
      filename: 'logs/combined.log'
    })
  ]
})

// Elasticsearch integration (production)
if (process.env.ELASTICSEARCH_URL) {
  logger.add(new ElasticsearchTransport({
    level: 'info',
    clientOpts: {
      node: process.env.ELASTICSEARCH_URL
    },
    index: 'ipfs-api-logs'
  }))
}

export default logger
```

**Deliverables:**
- Complete Prometheus monitoring setup
- IPFS-specific Grafana dashboards
- Comprehensive alerting rules
- Structured logging implementation
- Log aggregation and search capabilities
- Monitoring runbooks and procedures

---

## Phase 3: Multi-Node Cluster Implementation (Weeks 7-10)

### Week 7: IPFS Cluster Architecture & Planning

**Objectives:**
- Design multi-node cluster architecture
- Evaluate IPFS Cluster consensus mechanisms
- Plan network topology and security model
- Create cluster management automation

**Cluster Architecture Design:**
```yaml
# infrastructure/ipfs/cluster-compose.yml
version: '3.8'
services:
  ipfs-node-1:
    image: ipfs/kubo:v0.36.0
    container_name: ipfs-node-1
    environment:
      - IPFS_PROFILE=server
    volumes:
      - ./data/node1:/data/ipfs
      - ./configs/kubo-config-node1.json:/data/ipfs/config
    ports:
      - "4001:4001"
      - "5001:5001"
      - "8080:8080"
    networks:
      - ipfs-cluster
      
  ipfs-node-2:
    image: ipfs/kubo:v0.36.0
    container_name: ipfs-node-2
    environment:
      - IPFS_PROFILE=server
    volumes:
      - ./data/node2:/data/ipfs
      - ./configs/kubo-config-node2.json:/data/ipfs/config
    ports:
      - "4002:4001"
      - "5002:5001"
      - "8081:8080"
    networks:
      - ipfs-cluster
      
  ipfs-node-3:
    image: ipfs/kubo:v0.36.0
    container_name: ipfs-node-3
    environment:
      - IPFS_PROFILE=server
    volumes:
      - ./data/node3:/data/ipfs
      - ./configs/kubo-config-node3.json:/data/ipfs/config
    ports:
      - "4003:4001"
      - "5003:5001"
      - "8082:8080"
    networks:
      - ipfs-cluster

  cluster-node-1:
    image: ipfs/ipfs-cluster:v1.0.8
    container_name: cluster-node-1
    environment:
      - CLUSTER_PEERNAME=cluster-node-1
      - CLUSTER_SECRET=${CLUSTER_SECRET}
      - CLUSTER_IPFSHTTP_NODEMULTIADDRESS=/dns4/ipfs-node-1/tcp/5001
    volumes:
      - ./configs/cluster-config-1.json:/data/ipfs-cluster/service.json
    ports:
      - "9094:9094"  # HTTP API
      - "9095:9095"  # P2P
      - "9096:9096"  # Prometheus
    depends_on:
      - ipfs-node-1
    networks:
      - ipfs-cluster

  cluster-node-2:
    image: ipfs/ipfs-cluster:v1.0.8
    container_name: cluster-node-2
    environment:
      - CLUSTER_PEERNAME=cluster-node-2
      - CLUSTER_SECRET=${CLUSTER_SECRET}
      - CLUSTER_IPFSHTTP_NODEMULTIADDRESS=/dns4/ipfs-node-2/tcp/5001
    volumes:
      - ./configs/cluster-config-2.json:/data/ipfs-cluster/service.json
    ports:
      - "9097:9094"
      - "9098:9095"
      - "9099:9096"
    depends_on:
      - ipfs-node-2
      - cluster-node-1
    networks:
      - ipfs-cluster

  cluster-node-3:
    image: ipfs/ipfs-cluster:v1.0.8
    container_name: cluster-node-3
    environment:
      - CLUSTER_PEERNAME=cluster-node-3
      - CLUSTER_SECRET=${CLUSTER_SECRET}
      - CLUSTER_IPFSHTTP_NODEMULTIADDRESS=/dns4/ipfs-node-3/tcp/5001
    volumes:
      - ./configs/cluster-config-3.json:/data/ipfs-cluster/service.json
    ports:
      - "9100:9094"
      - "9101:9095"
      - "9102:9096"
    depends_on:
      - ipfs-node-3
      - cluster-node-1
    networks:
      - ipfs-cluster

  nginx-lb:
    image: nginx:alpine
    container_name: ipfs-gateway-lb
    volumes:
      - ./configs/nginx-cluster.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - ipfs-node-1
      - ipfs-node-2
      - ipfs-node-3
    networks:
      - ipfs-cluster

networks:
  ipfs-cluster:
    driver: bridge
```

**IPFS Cluster Configuration:**
```json
{
  "cluster": {
    "secret": "${CLUSTER_SECRET}",
    "peers": [],
    "bootstrap": [],
    "leave_on_shutdown": false,
    "listen_multiaddress": "/ip4/0.0.0.0/tcp/9096",
    "state_sync_interval": "1m0s",
    "ipfs_sync_interval": "2m10s",
    "replication_factor_min": 2,
    "replication_factor_max": 3,
    "monitor_ping_interval": "15s"
  },
  "consensus": {
    "crdt": {
      "cluster_name": "ipfs-cluster",
      "trusted_peers": ["*"],
      "batching": {
        "max_batch_size": 0,
        "max_batch_age": "0s"
      },
      "repair_interval": "1h0m0s"
    }
  },
  "api": {
    "ipfsproxy": {
      "listen_multiaddress": "/ip4/0.0.0.0/tcp/9095",
      "node_multiaddress": "/dns4/ipfs-node-1/tcp/5001",
      "log_level": "info",
      "extract_headers_extra": [],
      "extract_headers_path": "/api/v0/version",
      "extract_headers_ttl": "5m"
    },
    "restapi": {
      "http_listen_multiaddress": "/ip4/0.0.0.0/tcp/9094",
      "read_timeout": "30s",
      "read_header_timeout": "5s",
      "write_timeout": "60s",
      "idle_timeout": "120s",
      "max_header_bytes": 4096,
      "cors_allowed_origins": ["*"],
      "cors_allowed_methods": ["GET", "POST", "PUT", "DELETE"],
      "cors_allowed_headers": ["Content-Type", "Authorization"],
      "cors_exposed_headers": ["Content-Type", "X-Stream-Output"],
      "cors_allow_credentials": true,
      "cors_max_age": "0s"
    }
  },
  "ipfs_connector": {
    "ipfshttp": {
      "node_multiaddress": "/dns4/ipfs-node-1/tcp/5001",
      "connect_swarms_delay": "30s",
      "ipfs_request_timeout": "5m0s",
      "pin_timeout": "2m0s",
      "unpin_timeout": "3h0m0s",
      "repogc_timeout": "24h0m0s",
      "informer_trigger_interval": 0
    }
  },
  "pin_tracker": {
    "stateless": {
      "concurrent_pins": 10,
      "priority_pin_max_age": "24h0m0s",
      "priority_pin_max_retries": 5
    }
  },
  "monitor": {
    "pubsubmon": {
      "check_interval": "15s",
      "failure_threshold": 3
    }
  },
  "allocator": {
    "balanced": {
      "allocate_by": ["tag:group", "freespace"]
    }
  },
  "informer": {
    "disk": {
      "metric_ttl": "30s",
      "metric_type": "freespace"
    },
    "tags": {
      "metric_ttl": "30s",
      "tags": {
        "group": "default"
      }
    }
  }
}
```

**Cluster Management Scripts:**
```bash
#!/bin/bash
# infrastructure/ipfs/scripts/init-cluster.sh

set -e

echo "Initializing IPFS Cluster..."

# Generate cluster secret
CLUSTER_SECRET=$(docker run --rm ipfs/ipfs-cluster:latest ipfs-cluster-service init --consensus crdt | grep secret | cut -d'"' -f4)
echo "CLUSTER_SECRET=${CLUSTER_SECRET}" > .env

# Initialize first node
echo "Starting bootstrap node..."
docker-compose -f cluster-compose.yml up -d cluster-node-1

# Wait for bootstrap node to be ready
sleep 30

# Start remaining nodes
echo "Starting cluster nodes..."
docker-compose -f cluster-compose.yml up -d

# Wait for all nodes to join
sleep 60

# Verify cluster health
echo "Verifying cluster health..."
curl -s http://localhost:9094/peers | jq .

echo "IPFS Cluster initialization complete!"
echo "Cluster API: http://localhost:9094"
echo "Prometheus metrics: http://localhost:9096/metrics"
```

**Deliverables:**
- Complete cluster architecture design
- Multi-node Docker Compose configuration
- IPFS Cluster configuration templates
- Cluster initialization automation
- Network topology documentation

### Week 8: Cluster Implementation & Testing

**Objectives:**
- Deploy functional 3-node IPFS cluster
- Implement cluster health monitoring
- Test pin replication and failover scenarios
- Create cluster management tooling

**Cluster Health Monitoring:**
```typescript
// services/ipfs-manager/src/services/cluster.service.ts
import axios from 'axios'
import { logger } from '../utils/logger'

export interface ClusterPeer {
  id: string
  name: string
  addresses: string[]
  version: string
  commit: string
  error?: string
}

export interface PinStatus {
  cid: string
  name: string
  allocations: Array<{
    peer: string
    status: 'pinned' | 'pinning' | 'unpinned' | 'pin_error' | 'unpin_error'
    timestamp: string
    error?: string
  }>
  replication_factor_min: number
  replication_factor_max: number
}

export class ClusterService {
  private clusterEndpoints: string[]
  
  constructor() {
    this.clusterEndpoints = [
      'http://localhost:9094',
      'http://localhost:9097',
      'http://localhost:9100'
    ]
  }
  
  async getClusterPeers(): Promise<ClusterPeer[]> {
    try {
      // Try each cluster API endpoint until one succeeds
      for (const endpoint of this.clusterEndpoints) {
        try {
          const response = await axios.get(`${endpoint}/peers`, {
            timeout: 5000
          })
          return response.data
        } catch (error) {
          logger.warn(`Cluster endpoint ${endpoint} failed: ${error.message}`)
          continue
        }
      }
      throw new Error('All cluster endpoints unavailable')
    } catch (error) {
      logger.error('Failed to get cluster peers', error)
      throw error
    }
  }
  
  async pinToCluster(hash: string, options: {
    name?: string
    replication_factor_min?: number
    replication_factor_max?: number
    metadata?: Record<string, any>
  } = {}): Promise<PinStatus> {
    try {
      const payload = {
        cid: hash,
        name: options.name || hash,
        replication_factor_min: options.replication_factor_min || 2,
        replication_factor_max: options.replication_factor_max || 3,
        metadata: options.metadata || {}
      }
      
      for (const endpoint of this.clusterEndpoints) {
        try {
          const response = await axios.post(`${endpoint}/pins`, payload, {
            timeout: 30000,
            headers: {
              'Content-Type': 'application/json'
            }
          })
          
          logger.info(`Successfully pinned ${hash} to cluster`)
          return response.data
        } catch (error) {
          logger.warn(`Pin request to ${endpoint} failed: ${error.message}`)
          continue
        }
      }
      
      throw new Error('All cluster endpoints unavailable for pinning')
    } catch (error) {
      logger.error(`Failed to pin ${hash} to cluster`, error)
      throw error
    }
  }
  
  async getPinStatus(hash: string): Promise<PinStatus> {
    try {
      for (const endpoint of this.clusterEndpoints) {
        try {
          const response = await axios.get(`${endpoint}/pins/${hash}`, {
            timeout: 5000
          })
          return response.data
        } catch (error) {
          if (error.response?.status === 404) {
            throw new Error(`Pin ${hash} not found in cluster`)
          }
          continue
        }
      }
      throw new Error('All cluster endpoints unavailable')
    } catch (error) {
      logger.error(`Failed to get pin status for ${hash}`, error)
      throw error
    }
  }
  
  async unpinFromCluster(hash: string): Promise<void> {
    try {
      for (const endpoint of this.clusterEndpoints) {
        try {
          await axios.delete(`${endpoint}/pins/${hash}`, {
            timeout: 30000
          })
          logger.info(`Successfully unpinned ${hash} from cluster`)
          return
        } catch (error) {
          logger.warn(`Unpin request to ${endpoint} failed: ${error.message}`)
          continue
        }
      }
      throw new Error('All cluster endpoints unavailable for unpinning')
    } catch (error) {
      logger.error(`Failed to unpin ${hash} from cluster`, error)
      throw error
    }
  }
  
  async getClusterStatus(): Promise<{
    healthy: boolean
    peers: ClusterPeer[]
    totalPeers: number
    healthyPeers: number
    consensus: 'healthy' | 'degraded' | 'unhealthy'
  }> {
    try {
      const peers = await this.getClusterPeers()
      const healthyPeers = peers.filter(peer => !peer.error).length
      const totalPeers = peers.length
      
      let consensus: 'healthy' | 'degraded' | 'unhealthy'
      if (healthyPeers === totalPeers && totalPeers >= 3) {
        consensus = 'healthy'
      } else if (healthyPeers >= Math.ceil(totalPeers / 2)) {
        consensus = 'degraded'
      } else {
        consensus = 'unhealthy'
      }
      
      return {
        healthy: consensus !== 'unhealthy',
        peers,
        totalPeers,
        healthyPeers,
        consensus
      }
    } catch (error) {
      logger.error('Failed to get cluster status', error)
      return {
        healthy: false,
        peers: [],
        totalPeers: 0,
        healthyPeers: 0,
        consensus: 'unhealthy'
      }
    }
  }
}
```

**Failover Testing Script:**
```bash
#!/bin/bash
# infrastructure/ipfs/scripts/test-cluster-failover.sh

set -e

echo "Starting cluster failover test..."

# Test file
TEST_FILE="test-$(date +%s).txt"
echo "This is a test file for cluster failover testing" > "/tmp/${TEST_FILE}"

# Upload file to cluster
echo "Uploading test file..."
HASH=$(curl -X POST -F "file=@/tmp/${TEST_FILE}" http://localhost:9094/add | jq -r '.Hash')
echo "File hash: $HASH"

# Verify pin on all nodes
echo "Verifying pin status..."
curl -s "http://localhost:9094/pins/${HASH}" | jq .

# Stop one node
echo "Stopping cluster-node-2..."
docker stop cluster-node-2 ipfs-node-2

sleep 30

# Verify pin still accessible
echo "Testing file retrieval with one node down..."
curl -s "http://localhost:8080/ipfs/${HASH}" > "/tmp/retrieved-${TEST_FILE}"

if diff "/tmp/${TEST_FILE}" "/tmp/retrieved-${TEST_FILE}"; then
    echo "✅ File retrieval successful with one node down"
else
    echo "❌ File retrieval failed with one node down"
    exit 1
fi

# Stop second node
echo "Stopping cluster-node-3..."
docker stop cluster-node-3 ipfs-node-3

sleep 30

# Verify pin still accessible (should work with majority)
echo "Testing file retrieval with two nodes down..."
curl -s "http://localhost:8081/ipfs/${HASH}" > "/tmp/retrieved2-${TEST_FILE}"

if diff "/tmp/${TEST_FILE}" "/tmp/retrieved2-${TEST_FILE}"; then
    echo "✅ File retrieval successful with two nodes down"
else
    echo "❌ File retrieval failed with two nodes down"
    exit 1
fi

# Restart nodes
echo "Restarting nodes..."
docker start ipfs-node-2 cluster-node-2
docker start ipfs-node-3 cluster-node-3

sleep 60

# Verify cluster health
echo "Verifying cluster health after restart..."
curl -s http://localhost:9094/peers | jq .

# Clean up
rm "/tmp/${TEST_FILE}" "/tmp/retrieved-${TEST_FILE}" "/tmp/retrieved2-${TEST_FILE}"

echo "✅ Cluster failover test completed successfully"
```

**Load Balancer Configuration:**
```nginx
# infrastructure/ipfs/configs/nginx-cluster.conf
events {
    worker_connections 1024;
}

http {
    upstream ipfs_gateway {
        least_conn;
        server ipfs-node-1:8080 max_fails=3 fail_timeout=30s;
        server ipfs-node-2:8080 max_fails=3 fail_timeout=30s;
        server ipfs-node-3:8080 max_fails=3 fail_timeout=30s;
    }
    
    upstream cluster_api {
        least_conn;
        server cluster-node-1:9094 max_fails=3 fail_timeout=30s;
        server cluster-node-2:9094 max_fails=3 fail_timeout=30s;
        server cluster-node-3:9094 max_fails=3 fail_timeout=30s;
    }
    
    # Gateway load balancer
    server {
        listen 80;
        server_name _;
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # IPFS content routing
        location /ipfs/ {
            proxy_pass http://ipfs_gateway;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Caching for immutable content
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header Access-Control-Allow-Origin *;
        }
        
        # Cluster API proxy (internal only)
        location /cluster/ {
            allow 10.0.0.0/8;
            allow 172.16.0.0/12;
            allow 192.168.0.0/16;
            deny all;
            
            proxy_pass http://cluster_api/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

**Deliverables:**
- Functional 3-node IPFS cluster
- Cluster health monitoring system
- Failover testing suite
- Load balancer configuration
- Pin replication verification tools

### Week 9: API Integration with Cluster

**Objectives:**
- Adapt API services for multi-node cluster operations
- Implement cluster-aware pin management
- Add comprehensive cluster monitoring
- Optimize performance for clustered environment

**Enhanced Pin Service for Cluster:**
```typescript
// services/ipfs-manager/src/services/enhanced-pin.service.ts
import { ClusterService } from './cluster.service'
import { PinModel } from '../models/Pin'
import { logger } from '../utils/logger'

export interface ClusterPinOptions {
  replication_factor_min: number
  replication_factor_max: number
  name?: string
  metadata?: Record<string, any>
  priority?: 'low' | 'normal' | 'high'
}

export interface PinStatusReport {
  hash: string
  status: 'healthy' | 'degraded' | 'unhealthy'
  total_replicas: number
  healthy_replicas: number
  failed_replicas: number
  nodes: Array<{
    node_id: string
    status: string
    last_seen: Date
    error?: string
  }>
}

export class EnhancedPinService {
  constructor(
    private clusterService: ClusterService,
    private pinModel: PinModel
  ) {}

  async createClusterPin(
    hash: string,
    userId: string,
    options: ClusterPinOptions
  ): Promise<{
    id: string
    hash: string
    status: string
    replication_status: PinStatusReport
  }> {
    try {
      // Create pin record in database
      const pin = await this.pinModel.create({
        userId,
        ipfsHash: hash,
        name: options.name || hash,
        status: 'pinning',
        replicationFactorMin: options.replication_factor_min,
        replicationFactorMax: options.replication_factor_max,
        priority: options.priority || 'normal',
        metadata: options.metadata || {},
        createdAt: new Date()
      })

      // Pin to cluster
      const clusterPin = await this.clusterService.pinToCluster(hash, {
        name: options.name,
        replication_factor_min: options.replication_factor_min,
        replication_factor_max: options.replication_factor_max,
        metadata: {
          ...options.metadata,
          user_id: userId,
          pin_id: pin.id,
          priority: options.priority
        }
      })

      // Monitor pin status
      this.scheduleStatusCheck(pin.id, hash)

      const statusReport = await this.generateStatusReport(hash)

      return {
        id: pin.id,
        hash,
        status: 'pinning',
        replication_status: statusReport
      }
    } catch (error) {
      logger.error(`Failed to create cluster pin for ${hash}`, error)
      throw error
    }
  }

  async generateStatusReport(hash: string): Promise<PinStatusReport> {
    try {
      const pinStatus = await this.clusterService.getPinStatus(hash)
      
      const nodes = pinStatus.allocations.map(allocation => ({
        node_id: allocation.peer,
        status: allocation.status,
        last_seen: new Date(allocation.timestamp),
        error: allocation.error
      }))

      const healthy_replicas = nodes.filter(n => n.status === 'pinned').length
      const failed_replicas = nodes.filter(n => n.status.includes('error')).length
      const total_replicas = nodes.length

      let status: 'healthy' | 'degraded' | 'unhealthy'
      if (healthy_replicas >= pinStatus.replication_factor_min) {
        if (healthy_replicas === pinStatus.replication_factor_max && failed_replicas === 0) {
          status = 'healthy'
        } else {
          status = 'degraded'
        }
      } else {
        status = 'unhealthy'
      }

      return {
        hash,
        status,
        total_replicas,
        healthy_replicas,
        failed_replicas,
        nodes
      }
    } catch (error) {
      logger.error(`Failed to generate status report for ${hash}`, error)
      return {
        hash,
        status: 'unhealthy',
        total_replicas: 0,
        healthy_replicas: 0,
        failed_replicas: 0,
        nodes: []
      }
    }
  }

  async scheduleStatusCheck(pinId: string, hash: string): Promise<void> {
    // Use a job queue (Redis/Bull) for production
    setTimeout(async () => {
      try {
        const statusReport = await this.generateStatusReport(hash)
        
        // Update pin status in database
        await this.pinModel.update(pinId, {
          status: statusReport.status === 'healthy' ? 'pinned' : 'degraded',
          replicationStatus: statusReport,
          lastChecked: new Date()
        })

        // Schedule next check if not healthy
        if (statusReport.status !== 'healthy') {
          this.scheduleStatusCheck(pinId, hash)
        }
      } catch (error) {
        logger.error(`Status check failed for pin ${pinId}`, error)
        // Retry with exponential backoff
        setTimeout(() => this.scheduleStatusCheck(pinId, hash), 30000)
      }
    }, 10000) // Initial check after 10 seconds
  }

  async getClusterHealth(): Promise<{
    overall_health: 'healthy' | 'degraded' | 'unhealthy'
    cluster_status: any
    pin_distribution: Array<{
      node_id: string
      total_pins: number
      healthy_pins: number
      failed_pins: number
    }>
  }> {
    try {
      const clusterStatus = await this.clusterService.getClusterStatus()
      
      // Get pin distribution across nodes
      const allPins = await this.pinModel.findAll({ 
        status: { $in: ['pinned', 'pinning', 'degraded'] } 
      })
      
      const pinDistribution = new Map<string, {
        total_pins: number
        healthy_pins: number
        failed_pins: number
      }>()

      // Initialize with cluster peers
      clusterStatus.peers.forEach(peer => {
        pinDistribution.set(peer.id, {
          total_pins: 0,
          healthy_pins: 0,
          failed_pins: 0
        })
      })

      // Count pins per node
      for (const pin of allPins) {
        if (pin.replicationStatus?.nodes) {
          pin.replicationStatus.nodes.forEach(node => {
            const stats = pinDistribution.get(node.node_id) || {
              total_pins: 0,
              healthy_pins: 0,
              failed_pins: 0
            }
            
            stats.total_pins++
            if (node.status === 'pinned') {
              stats.healthy_pins++
            } else if (node.status.includes('error')) {
              stats.failed_pins++
            }
            
            pinDistribution.set(node.node_id, stats)
          })
        }
      }

      return {
        overall_health: clusterStatus.consensus,
        cluster_status: clusterStatus,
        pin_distribution: Array.from(pinDistribution.entries()).map(([node_id, stats]) => ({
          node_id,
          ...stats
        }))
      }
    } catch (error) {
      logger.error('Failed to get cluster health', error)
      return {
        overall_health: 'unhealthy',
        cluster_status: null,
        pin_distribution: []
      }
    }
  }

  async optimizePinDistribution(): Promise<{
    rebalanced_pins: number
    moved_pins: Array<{
      hash: string
      from_nodes: string[]
      to_nodes: string[]
    }>
  }> {
    // Implementation for automatic pin rebalancing
    // This would analyze current distribution and move pins
    // to balance load across cluster nodes
    try {
      const health = await this.getClusterHealth()
      const rebalanced_pins = 0
      const moved_pins = []

      // Logic to identify over/under-utilized nodes
      // and rebalance pins accordingly
      
      logger.info(`Pin optimization completed: ${rebalanced_pins} pins rebalanced`)
      
      return { rebalanced_pins, moved_pins }
    } catch (error) {
      logger.error('Pin optimization failed', error)
      throw error
    }
  }
}
```

**Cluster-Aware API Endpoints:**
```typescript
// apps/api/src/controllers/cluster.controller.ts
import { Request, Response } from 'express'
import { EnhancedPinService } from '../services/enhanced-pin.service'

export class ClusterController {
  constructor(private enhancedPinService: EnhancedPinService) {}

  async pinWithReplication(req: Request, res: Response) {
    try {
      const { hash } = req.params
      const userId = req.user.id
      const {
        replication_factor_min = 2,
        replication_factor_max = 3,
        name,
        priority = 'normal'
      } = req.body

      const result = await this.enhancedPinService.createClusterPin(hash, userId, {
        replication_factor_min,
        replication_factor_max,
        name,
        priority,
        metadata: {
          created_via: 'api',
          user_agent: req.get('User-Agent')
        }
      })

      res.status(201).json({
        success: true,
        data: result
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }

  async getPinStatus(req: Request, res: Response) {
    try {
      const { hash } = req.params
      
      const statusReport = await this.enhancedPinService.generateStatusReport(hash)
      
      res.json({
        success: true,
        data: statusReport
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }

  async getClusterHealth(req: Request, res: Response) {
    try {
      const health = await this.enhancedPinService.getClusterHealth()
      
      res.json({
        success: true,
        data: health
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }

  async optimizeCluster(req: Request, res: Response) {
    try {
      // Only allow admins to trigger optimization
      if (req.user.role !== 'admin') {
        return res.status(403).json({
          success: false,
          error: 'Insufficient permissions'
        })
      }

      const result = await this.enhancedPinService.optimizePinDistribution()
      
      res.json({
        success: true,
        data: result
      })
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message
      })
    }
  }
}
```

**Deliverables:**
- Cluster-integrated pin management system
- Advanced pin status tracking
- Cluster health monitoring APIs
- Pin distribution optimization tools
- Performance testing results

### Week 10: Production Deployment & Testing

**Objectives:**
- Deploy cluster to production environment
- Implement Infrastructure as Code
- Complete end-to-end testing
- Create operational procedures

**Kubernetes Deployment:**
```yaml
# infrastructure/kubernetes/production/ipfs-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ipfs-cluster
  namespace: ipfs
spec:
  serviceName: ipfs-cluster
  replicas: 3
  selector:
    matchLabels:
      app: ipfs-cluster
  template:
    metadata:
      labels:
        app: ipfs-cluster
    spec:
      containers:
      - name: ipfs
        image: ipfs/kubo:v0.36.0
        ports:
        - containerPort: 4001
          name: swarm
        - containerPort: 5001
          name: api
        - containerPort: 8080
          name: gateway
        env:
        - name: IPFS_PROFILE
          value: "server"
        volumeMounts:
        - name: ipfs-data
          mountPath: /data/ipfs
        - name: config
          mountPath: /data/ipfs/config
          subPath: kubo-config.json
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
            
      - name: cluster
        image: ipfs/ipfs-cluster:v1.0.8
        ports:
        - containerPort: 9094
          name: api
        - containerPort: 9095
          name: proxy
        - containerPort: 9096
          name: metrics
        env:
        - name: CLUSTER_PEERNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_SECRET
          valueFrom:
            secretKeyRef:
              name: cluster-secret
              key: secret
        - name: CLUSTER_IPFSHTTP_NODEMULTIADDRESS
          value: "/ip4/127.0.0.1/tcp/5001"
        volumeMounts:
        - name: cluster-data
          mountPath: /data/ipfs-cluster
        - name: config
          mountPath: /data/ipfs-cluster/service.json
          subPath: cluster-config.json
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
            
      volumes:
      - name: config
        configMap:
          name: ipfs-config
          
  volumeClaimTemplates:
  - metadata:
      name: ipfs-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
  - metadata:
      name: cluster-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: ipfs-cluster-api
  namespace: ipfs
spec:
  selector:
    app: ipfs-cluster
  ports:
  - name: cluster-api
    port: 9094
    targetPort: 9094
  - name: ipfs-api
    port: 5001
    targetPort: 5001
  - name: gateway
    port: 8080
    targetPort: 8080
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: ipfs-gateway-lb
  namespace: ipfs
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: ipfs-cluster
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

**End-to-End Testing Suite:**
```bash
#!/bin/bash
# infrastructure/testing/e2e-cluster-test.sh

set -e

echo "Starting end-to-end cluster testing..."

# Test configuration
API_ENDPOINT="https://api.yourdomain.com"
GATEWAY_ENDPOINT="https://gateway.yourdomain.com"
TEST_FILES_DIR="/tmp/ipfs-test-files"
RESULTS_DIR="/tmp/test-results"

# Setup
mkdir -p "$TEST_FILES_DIR" "$RESULTS_DIR"

# Generate test files of various sizes
echo "Generating test files..."
echo "Small text file" > "$TEST_FILES_DIR/small.txt"
dd if=/dev/urandom of="$TEST_FILES_DIR/medium.bin" bs=1M count=10
dd if=/dev/urandom of="$TEST_FILES_DIR/large.bin" bs=1M count=100

# Test 1: File Upload and Pin
echo "Test 1: File upload and cluster pinning..."
for file in "$TEST_FILES_DIR"/*; do
    filename=$(basename "$file")
    echo "Uploading $filename..."
    
    # Upload via API
    response=$(curl -s -X POST \
        -H "Authorization: Bearer $API_TOKEN" \
        -F "file=@$file" \
        -F "replication_factor_min=2" \
        -F "replication_factor_max=3" \
        "$API_ENDPOINT/api/v1/files/upload")
    
    hash=$(echo "$response" | jq -r '.data.hash')
    echo "Hash: $hash"
    
    # Verify pin status
    sleep 10
    status_response=$(curl -s \
        -H "Authorization: Bearer $API_TOKEN" \
        "$API_ENDPOINT/api/v1/pins/$hash/status")
    
    pin_status=$(echo "$status_response" | jq -r '.data.status')
    healthy_replicas=$(echo "$status_response" | jq -r '.data.healthy_replicas')
    
    if [[ "$pin_status" == "healthy" && "$healthy_replicas" -ge 2 ]]; then
        echo "✅ $filename pinned successfully with $healthy_replicas replicas"
    else
        echo "❌ $filename pin failed: status=$pin_status, replicas=$healthy_replicas"
        exit 1
    fi
    
    # Test retrieval
    echo "Testing retrieval..."
    curl -s "$GATEWAY_ENDPOINT/ipfs/$hash" > "$RESULTS_DIR/$filename"
    
    if cmp -s "$file" "$RESULTS_DIR/$filename"; then
        echo "✅ $filename retrieved successfully"
    else
        echo "❌ $filename retrieval failed - content mismatch"
        exit 1
    fi
    
    echo "Hash: $hash" >> "$RESULTS_DIR/uploaded_hashes.txt"
done

# Test 2: Cluster Resilience
echo "Test 2: Cluster resilience testing..."

# Get random hash from uploaded files
test_hash=$(head -n1 "$RESULTS_DIR/uploaded_hashes.txt" | cut -d' ' -f2)

# Simulate node failure by scaling down one replica
echo "Simulating node failure..."
kubectl scale statefulset ipfs-cluster --replicas=2 -n ipfs
sleep 30

# Test if content is still accessible
echo "Testing content availability with reduced nodes..."
curl -s "$GATEWAY_ENDPOINT/ipfs/$test_hash" > "$RESULTS_DIR/resilience_test.bin"

if [[ -f "$RESULTS_DIR/resilience_test.bin" && -s "$RESULTS_DIR/resilience_test.bin" ]]; then
    echo "✅ Content still accessible with reduced cluster"
else
    echo "❌ Content not accessible with reduced cluster"
    exit 1
fi

# Restore full cluster
kubectl scale statefulset ipfs-cluster --replicas=3 -n ipfs
sleep 60

# Test 3: Performance Testing
echo "Test 3: Performance testing..."

# Upload performance test
start_time=$(date +%s)
for i in {1..10}; do
    echo "Performance test file $i" > "$TEST_FILES_DIR/perf_$i.txt"
    curl -s -X POST \
        -H "Authorization: Bearer $API_TOKEN" \
        -F "file=@$TEST_FILES_DIR/perf_$i.txt" \
        "$API_ENDPOINT/api/v1/files/upload" > /dev/null
done
end_time=$(date +%s)

upload_time=$((end_time - start_time))
echo "✅ 10 file uploads completed in ${upload_time}s (avg: $((upload_time/10))s per file)"

# Retrieval performance test
start_time=$(date +%s)
for hash in $(grep "Hash:" "$RESULTS_DIR/uploaded_hashes.txt" | head -10 | cut -d' ' -f2); do
    curl -s "$GATEWAY_ENDPOINT/ipfs/$hash" > /dev/null
done
end_time=$(date +%s)

retrieval_time=$((end_time - start_time))
echo "✅ 10 file retrievals completed in ${retrieval_time}s (avg: $((retrieval_time/10))s per file)"

# Test 4: Cluster Health
echo "Test 4: Cluster health verification..."
health_response=$(curl -s \
    -H "Authorization: Bearer $API_TOKEN" \
    "$API_ENDPOINT/api/v1/cluster/health")

overall_health=$(echo "$health_response" | jq -r '.data.overall_health')
healthy_peers=$(echo "$health_response" | jq -r '.data.cluster_status.healthyPeers')
total_peers=$(echo "$health_response" | jq -r '.data.cluster_status.totalPeers')

if [[ "$overall_health" == "healthy" && "$healthy_peers" -eq "$total_peers" && "$total_peers" -eq 3 ]]; then
    echo "✅ Cluster health: $overall_health ($healthy_peers/$total_peers nodes)"
else
    echo "❌ Cluster health issues: $overall_health ($healthy_peers/$total_peers nodes)"
    exit 1
fi

# Cleanup
echo "Cleaning up test files..."
rm -rf "$TEST_FILES_DIR" "$RESULTS_DIR"

echo "🎉 All end-to-end tests passed successfully!"
echo "Cluster is production-ready."
```

**Operational Procedures:**
```bash
# infrastructure/operations/runbooks/cluster-operations.md

## IPFS Cluster Operations Runbook

### Daily Health Checks
```bash
# Check cluster status
kubectl get pods -n ipfs
kubectl logs -n ipfs statefulset/ipfs-cluster -c cluster --tail=100

# Verify cluster consensus
curl -s https://api.yourdomain.com/api/v1/cluster/health | jq .

# Check storage usage
kubectl exec -n ipfs ipfs-cluster-0 -c ipfs -- df -h /data/ipfs

# Verify pin distribution
curl -s https://cluster-api.yourdomain.com/allocations | jq '.[] | {cid: .cid, peers: [.allocations[].peer]}'
```

### Incident Response

**Node Failure:**
1. Identify failed node: `kubectl get pods -n ipfs`
2. Check node logs: `kubectl logs ipfs-cluster-X -c ipfs`
3. Restart if needed: `kubectl delete pod ipfs-cluster-X`
4. Verify cluster recovery: `curl cluster-api/peers`

**Pin Failures:**
1. Identify failed pins: `curl cluster-api/pins | grep error`
2. Check pin status: `curl cluster-api/pins/QmHash`
3. Retry pin: `curl -X POST cluster-api/pins/QmHash`
4. Manual intervention if needed

**Performance Issues:**
1. Check resource usage: `kubectl top pods -n ipfs`
2. Verify storage: `df -h` on nodes
3. Check gateway response times
4. Scale cluster if needed: `kubectl scale statefulset ipfs-cluster --replicas=5`

### Maintenance Procedures

**Cluster Upgrade:**
```bash
# Rolling update
kubectl patch statefulset ipfs-cluster -p '{"spec":{"template":{"spec":{"containers":[{"name":"cluster","image":"ipfs/ipfs-cluster:v1.0.9"}]}}}}'

# Monitor rollout
kubectl rollout status statefulset/ipfs-cluster -n ipfs
```

**Backup Procedures:**
```bash
# Automated daily backup
kubectl create job cluster-backup-$(date +%Y%m%d) --from=cronjob/cluster-backup -n ipfs

# Manual backup
kubectl exec ipfs-cluster-0 -c ipfs -- ipfs pin ls --type=recursive > pins-backup-$(date +%Y%m%d).txt
```
```

**Deliverables:**
- Production-ready Kubernetes deployment
- Comprehensive end-to-end testing suite
- Operational runbooks and procedures
- Performance benchmarking results
- Production monitoring and alerting
- Disaster recovery documentation

---

## Phase 4: Scaling & Optimization (Weeks 11-14)

### Week 11: Performance Optimization & CDN Integration

**Objectives:**
- Implement advanced caching strategies for IPFS gateways
- Integrate CDN for global content delivery
- Optimize database queries and API performance
- Implement content deduplication improvements

**CDN Integration Architecture:**
```yaml
# infrastructure/cdn/cloudflare-config.yaml
# Cloudflare Workers for intelligent IPFS routing
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  
  // Parse IPFS hash from path
  const pathMatch = url.pathname.match(/^\/ipfs\/([a-zA-Z0-9]+)/)
  if (!pathMatch) {
    return new Response('Invalid IPFS path', { status: 400 })
  }
  
  const hash = pathMatch[1]
  
  // Try multiple IPFS gateways in order of preference
  const gateways = [
    'https://gateway-eu.yourdomain.com',
    'https://gateway-us.yourdomain.com',
    'https://gateway-asia.yourdomain.com',
    'https://ipfs.io'  // Fallback
  ]
  
  // Add cache headers for immutable content
  const cacheHeaders = {
    'Cache-Control': 'public, max-age=31536000, immutable',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, HEAD, OPTIONS',
    'Access-Control-Allow-Headers': 'Range, If-None-Match'
  }
  
  for (const gateway of gateways) {
    try {
      const response = await fetch(`${gateway}/ipfs/${hash}`, {
        headers: request.headers,
        cf: {
          // Cloudflare-specific caching
          cacheEverything: true,
          cacheTtl: 31536000, // 1 year
        }
      })
      
      if (response.ok) {
        // Add security and performance headers
        const newResponse = new Response(response.body, {
          status: response.status,
          statusText: response.statusText,
          headers: {
            ...response.headers,
            ...cacheHeaders,
            'X-IPFS-Gateway': gateway,
            'X-Content-Type-Options': 'nosniff',
            'X-Frame-Options': 'DENY'
          }
        })
        
        return newResponse
      }
    } catch (error) {
      console.error(`Gateway ${gateway} failed:`, error)
      continue
    }
  }
  
  return new Response('Content not available', { status: 504 })
}
```

**Advanced Gateway Caching:**
```nginx
# infrastructure/nginx/advanced-gateway.conf
http {
    # Cache zones
    proxy_cache_path /var/cache/nginx/ipfs 
        levels=1:2 
        keys_zone=ipfs_cache:100m 
        max_size=50g 
        inactive=365d 
        use_temp_path=off;
    
    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=gateway:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=upload:10m rate=1r/s;
    
    # Upstream health checking
    upstream ipfs_cluster_healthy {
        least_conn;
        server ipfs-node-1:8080 max_fails=2 fail_timeout=30s;
        server ipfs-node-2:8080 max_fails=2 fail_timeout=30s backup;
        server ipfs-node-3:8080 max_fails=2 fail_timeout=30s backup;
        
        # Health check (nginx plus)
        # health_check interval=5s fails=3 passes=2;
    }
    
    # Geographic load balancing
    geo $closest_gateway {
        default us;
        # EU ranges
        2.0.0.0/8 eu;
        31.0.0.0/8 eu;
        # Asia ranges  
        1.0.0.0/8 asia;
        14.0.0.0/8 asia;
    }
    
    map $closest_gateway $gateway_upstream {
        us ipfs_cluster_us;
        eu ipfs_cluster_eu;
        asia ipfs_cluster_asia;
        default ipfs_cluster_healthy;
    }
    
    server {
        listen 80;
        server_name gateway.yourdomain.com;
        
        # Security headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
        add_header Referrer-Policy "strict-origin-when-cross-origin";
        
        # IPFS content serving
        location ~ ^/ipfs/([a-zA-Z0-9]+)$ {
            limit_req zone=gateway burst=20 nodelay;
            
            # Enable caching for IPFS content (immutable)
            proxy_cache ipfs_cache;
            proxy_cache_key "$scheme$proxy_host$uri";
            proxy_cache_valid 200 365d;
            proxy_cache_valid 404 1h;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_lock on;
            
            # Cache status headers
            add_header X-Cache-Status $upstream_cache_status;
            add_header X-IPFS-Hash $1;
            
            # CORS headers
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS";
            add_header Access-Control-Max-Age 86400;
            
            # Immutable content caching
            add_header Cache-Control "public, max-age=31536000, immutable";
            
            # Route to closest gateway
            proxy_pass http://$gateway_upstream;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeout configuration
            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 60s;
            
            # Buffer configuration for large files
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            proxy_max_temp_file_size 1024m;
        }
        
        # Named file access with content disposition
        location ~ ^/ipfs/([a-zA-Z0-9]+)/(.+)$ {
            limit_req zone=gateway burst=20 nodelay;
            
            proxy_cache ipfs_cache;
            proxy_cache_key "$scheme$proxy_host$uri";
            proxy_cache_valid 200 365d;
            
            add_header Content-Disposition "inline; filename=\"$2\"";
            add_header X-IPFS-Hash $1;
            add_header X-Filename $2;
            
            proxy_pass http://$gateway_upstream/ipfs/$1;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # Cache stats (admin only)
        location /cache-stats {
            allow 10.0.0.0/8;
            deny all;
            
            access_log off;
            proxy_cache_bypass 1;
            add_header Content-Type application/json;
            return 200 '{"cache_size": "$proxy_cache_path_size", "hits": "$upstream_cache_status"}';
        }
    }
}
```

**Database Query Optimization:**
```typescript
// packages/db/src/models/optimized-file.model.ts
import { Model, QueryInterface, DataTypes, Op } from 'sequelize'

export interface FileAttributes {
  id: string
  userId: string
  ipfsHash: string
  originalName: string
  contentType: string
  size: number
  uploadedAt: Date
  accessCount: number
  lastAccessed: Date
  tags: string[]
  metadata: Record<string, any>
}

export class OptimizedFileModel extends Model<FileAttributes> {
  // Optimized queries with proper indexing
  
  static async findPopularFiles(limit = 50): Promise<FileAttributes[]> {
    return this.findAll({
      attributes: [
        'id', 'ipfsHash', 'originalName', 'contentType', 
        'size', 'accessCount', 'lastAccessed'
      ],
      where: {
        accessCount: { [Op.gt]: 10 },
        lastAccessed: { [Op.gt]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
      },
      order: [
        ['accessCount', 'DESC'],
        ['lastAccessed', 'DESC']
      ],
      limit,
      // Use read replica for analytics
      useMaster: false
    })
  }
  
  static async findByUserPaginated(
    userId: string, 
    options: {
      page: number
      limit: number
      sortBy?: 'uploadedAt' | 'size' | 'accessCount'
      sortOrder?: 'ASC' | 'DESC'
      contentType?: string
      search?: string
    }
  ): Promise<{
    files: FileAttributes[]
    total: number
    page: number
    limit: number
  }> {
    const offset = (options.page - 1) * options.limit
    const sortBy = options.sortBy || 'uploadedAt'
    const sortOrder = options.sortOrder || 'DESC'
    
    const whereClause: any = { userId }
    
    if (options.contentType) {
      whereClause.contentType = { [Op.like]: `${options.contentType}%` }
    }
    
    if (options.search) {
      whereClause.originalName = { [Op.iLike]: `%${options.search}%` }
    }
    
    const { rows: files, count: total } = await this.findAndCountAll({
      where: whereClause,
      order: [[sortBy, sortOrder]],
      limit: options.limit,
      offset,
      // Include only necessary fields
      attributes: [
        'id', 'ipfsHash', 'originalName', 'contentType',
        'size', 'uploadedAt', 'accessCount', 'lastAccessed'
      ]
    })
    
    return {
      files: files as FileAttributes[],
      total,
      page: options.page,
      limit: options.limit
    }
  }
  
  static async updateAccessStats(ipfsHash: string): Promise<void> {
    // Batch update using raw query for better performance
    await this.sequelize.query(
      'UPDATE files SET access_count = access_count + 1, last_accessed = NOW() WHERE ipfs_hash = :hash',
      {
        replacements: { hash: ipfsHash },
        type: QueryInterface.QueryTypes.UPDATE
      }
    )
  }
  
  static async getStorageStatsByUser(userId: string): Promise<{
    totalFiles: number
    totalSize: number
    sizeByType: Array<{ contentType: string, count: number, totalSize: number }>
  }> {
    const results = await this.sequelize.query(`
      SELECT 
        COUNT(*) as total_files,
        SUM(size) as total_size,
        SPLIT_PART(content_type, '/', 1) as content_category,
        COUNT(*) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'image') as image_count,
        SUM(size) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'image') as image_size,
        COUNT(*) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'video') as video_count,
        SUM(size) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'video') as video_size,
        COUNT(*) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'application') as document_count,
        SUM(size) FILTER (WHERE SPLIT_PART(content_type, '/', 1) = 'application') as document_size
      FROM files 
      WHERE user_id = :userId
      GROUP BY user_id
    `, {
      replacements: { userId },
      type: QueryInterface.QueryTypes.SELECT
    })
    
    if (results.length === 0) {
      return { totalFiles: 0, totalSize: 0, sizeByType: [] }
    }
    
    const result = results[0] as any
    
    return {
      totalFiles: parseInt(result.total_files),
      totalSize: parseInt(result.total_size),
      sizeByType: [
        { contentType: 'image', count: result.image_count || 0, totalSize: result.image_size || 0 },
        { contentType: 'video', count: result.video_count || 0, totalSize: result.video_size || 0 },
        { contentType: 'document', count: result.document_count || 0, totalSize: result.document_size || 0 }
      ]
    }
  }
}

// Database migrations for optimization
export const up = async (queryInterface: QueryInterface) => {
  // Add indexes for common queries
  await queryInterface.addIndex('files', ['user_id', 'uploaded_at'], {
    name: 'idx_files_user_uploaded'
  })
  
  await queryInterface.addIndex('files', ['ipfs_hash'], {
    name: 'idx_files_ipfs_hash',
    unique: true
  })
  
  await queryInterface.addIndex('files', ['content_type'], {
    name: 'idx_files_content_type'
  })
  
  await queryInterface.addIndex('files', ['access_count', 'last_accessed'], {
    name: 'idx_files_popularity'
  })
  
  // Add partial index for active files
  await queryInterface.sequelize.query(`
    CREATE INDEX CONCURRENTLY idx_files_active 
    ON files (user_id, uploaded_at DESC) 
    WHERE last_accessed > NOW() - INTERVAL '30 days'
  `)
  
  // Add GIN index for metadata search
  await queryInterface.sequelize.query(`
    CREATE INDEX CONCURRENTLY idx_files_metadata_gin
    ON files USING gin (metadata)
  `)
}
```

**Content Deduplication Service:**
```typescript
// services/file-processor/src/services/deduplication.service.ts
import crypto from 'crypto'
import { IPFSService } from './ipfs.service'
import { FileModel } from '../models/File'
import { logger } from '../utils/logger'

export interface DuplicationAnalysis {
  isDuplicate: boolean
  existingFileId?: string
  existingHash?: string
  similarFiles: Array<{
    fileId: string
    hash: string
    similarity: number
    reason: string
  }>
}

export class DeduplicationService {
  constructor(
    private ipfsService: IPFSService,
    private fileModel: FileModel
  ) {}
  
  async analyzeDuplication(
    fileBuffer: Buffer,
    userId: string,
    metadata: {
      originalName: string
      contentType: string
      size: number
    }
  ): Promise<DuplicationAnalysis> {
    try {
      // 1. Check for exact content hash match
      const contentHash = this.calculateContentHash(fileBuffer)
      const exactMatch = await this.fileModel.findByContentHash(contentHash)
      
      if (exactMatch) {
        logger.info(`Exact duplicate found for user ${userId}: ${exactMatch.ipfsHash}`)
        return {
          isDuplicate: true,
          existingFileId: exactMatch.id,
          existingHash: exactMatch.ipfsHash,
          similarFiles: []
        }
      }
      
      // 2. Check for IPFS content addressing (should be identical to content hash)
      // But verify by attempting to add to IPFS and checking CID
      const ipfsResult = await this.ipfsService.addFile(fileBuffer, { onlyHash: true })
      const ipfsHash = ipfsResult.cid.toString()
      
      const ipfsMatch = await this.fileModel.findByHash(ipfsHash)
      if (ipfsMatch) {
        logger.info(`IPFS hash duplicate found for user ${userId}: ${ipfsHash}`)
        return {
          isDuplicate: true,
          existingFileId: ipfsMatch.id,
          existingHash: ipfsMatch.ipfsHash,
          similarFiles: []
        }
      }
      
      // 3. Check for similar files (same name, similar size)
      const similarFiles = await this.findSimilarFiles(userId, metadata)
      
      // 4. For images, check perceptual hash
      const perceptualSimilarity = await this.checkPerceptualSimilarity(
        fileBuffer, 
        metadata.contentType
      )
      
      return {
        isDuplicate: false,
        similarFiles: [...similarFiles, ...perceptualSimilarity]
      }
    } catch (error) {
      logger.error('Deduplication analysis failed', error)
      // Don't fail the upload, just log and continue
      return { isDuplicate: false, similarFiles: [] }
    }
  }
  
  private calculateContentHash(buffer: Buffer): string {
    return crypto.createHash('sha256').update(buffer).digest('hex')
  }
  
  private async findSimilarFiles(
    userId: string, 
    metadata: { originalName: string; contentType: string; size: number }
  ): Promise<Array<{ fileId: string; hash: string; similarity: number; reason: string }>> {
    const similarFiles = []
    
    // Same filename
    const sameNameFiles = await this.fileModel.findAll({
      where: {
        userId,
        originalName: metadata.originalName,
        contentType: metadata.contentType
      },
      attributes: ['id', 'ipfsHash', 'size', 'uploadedAt']
    })
    
    for (const file of sameNameFiles) {
      const sizeDiff = Math.abs(file.size - metadata.size)
      const similarity = Math.max(0, 100 - (sizeDiff / metadata.size) * 100)
      
      if (similarity > 90) {
        similarFiles.push({
          fileId: file.id,
          hash: file.ipfsHash,
          similarity: Math.round(similarity),
          reason: `Same filename, ${similarity.toFixed(1)}% size similarity`
        })
      }
    }
    
    // Similar size files of same type
    const sizeThreshold = metadata.size * 0.05 // 5% size variance
    const similarSizeFiles = await this.fileModel.findAll({
      where: {
        userId,
        contentType: metadata.contentType,
        size: {
          [Op.between]: [metadata.size - sizeThreshold, metadata.size + sizeThreshold]
        }
      },
      limit: 5,
      attributes: ['id', 'ipfsHash', 'originalName', 'size']
    })
    
    for (const file of similarSizeFiles) {
      const nameSimilarity = this.calculateStringSimilarity(
        metadata.originalName, 
        file.originalName
      )
      
      if (nameSimilarity > 0.7) {
        similarFiles.push({
          fileId: file.id,
          hash: file.ipfsHash,
          similarity: Math.round(nameSimilarity * 100),
          reason: `Similar filename and size`
        })
      }
    }
    
    return similarFiles
  }
  
  private calculateStringSimilarity(str1: string, str2: string): number {
    // Levenshtein distance based similarity
    const matrix = []
    const len1 = str1.length
    const len2 = str2.length
    
    for (let i = 0; i <= len2; i++) {
      matrix[i] = [i]
    }
    
    for (let j = 0; j <= len1; j++) {
      matrix[0][j] = j
    }
    
    for (let i = 1; i <= len2; i++) {
      for (let j = 1; j <= len1; j++) {
        if (str2.charAt(i - 1) === str1.charAt(j - 1)) {
          matrix[i][j] = matrix[i - 1][j - 1]
        } else {
          matrix[i][j] = Math.min(
            matrix[i - 1][j - 1] + 1,
            matrix[i][j - 1] + 1,
            matrix[i - 1][j] + 1
          )
        }
      }
    }
    
    const maxLen = Math.max(len1, len2)
    return (maxLen - matrix[len2][len1]) / maxLen
  }
  
  private async checkPerceptualSimilarity(
    buffer: Buffer, 
    contentType: string
  ): Promise<Array<{ fileId: string; hash: string; similarity: number; reason: string }>> {
    if (!contentType.startsWith('image/')) {
      return []
    }
    
    try {
      // Use a library like 'sharp' for image processing and perceptual hashing
      const sharp = require('sharp')
      
      // Generate perceptual hash
      const { data: imageBuffer } = await sharp(buffer)
        .greyscale()
        .resize(8, 8, { fit: 'fill' })
        .raw()
        .toBuffer({ resolveWithObject: true })
      
      const perceptualHash = this.generatePerceptualHash(imageBuffer)
      
      // Find similar perceptual hashes in database
      // This would require storing perceptual hashes in the database
      const similarImages = await this.fileModel.findSimilarPerceptualHashes(
        perceptualHash, 
        5 // Hamming distance threshold
      )
      
      return similarImages.map(img => ({
        fileId: img.id,
        hash: img.ipfsHash,
        similarity: Math.round((1 - img.hammingDistance / 64) * 100),
        reason: `Visually similar image (${img.hammingDistance} bit difference)`
      }))
    } catch (error) {
      logger.warn('Perceptual hash comparison failed', error)
      return []
    }
  }
  
  private generatePerceptualHash(imageData: Buffer): string {
    // Simple average hash implementation
    const pixels = Array.from(imageData)
    const avg = pixels.reduce((sum, pixel) => sum + pixel, 0) / pixels.length
    
    return pixels
      .map(pixel => pixel > avg ? '1' : '0')
      .join('')
  }
  
  async handleDuplicateUpload(
    userId: string,
    existingFileId: string,
    metadata: {
      originalName?: string
      tags?: string[]
      customMetadata?: Record<string, any>
    }
  ): Promise<{ fileId: string; hash: string; action: 'linked' | 'updated' }> {
    // Create a new reference to the existing file instead of storing duplicate
    const existingFile = await this.fileModel.findById(existingFileId)
    
    if (!existingFile) {
      throw new Error('Existing file not found')
    }
    
    // Check if user already has a reference to this file
    const userReference = await this.fileModel.findOne({
      where: {
        userId,
        ipfsHash: existingFile.ipfsHash
      }
    })
    
    if (userReference) {
      // Update existing reference with new metadata
      await this.fileModel.update(userReference.id, {
        originalName: metadata.originalName || userReference.originalName,
        tags: [...new Set([...(userReference.tags || []), ...(metadata.tags || [])])],
        metadata: {
          ...userReference.metadata,
          ...metadata.customMetadata,
          duplicateDetectedAt: new Date(),
          originalUploadAttempt: new Date()
        }
      })
      
      return {
        fileId: userReference.id,
        hash: existingFile.ipfsHash,
        action: 'updated'
      }
    } else {
      // Create new reference for this user
      const newReference = await this.fileModel.create({
        userId,
        ipfsHash: existingFile.ipfsHash,
        originalName: metadata.originalName || existingFile.originalName,
        contentType: existingFile.contentType,
        size: existingFile.size,
        tags: metadata.tags || [],
        metadata: {
          ...metadata.customMetadata,
          linkedToOriginal: existingFileId,
          deduplicatedAt: new Date(),
          originalUploader: existingFile.userId
        }
      })
      
      return {
        fileId: newReference.id,
        hash: existingFile.ipfsHash,
        action: 'linked'
      }
    }
  }
}
```

**Deliverables:**
- CDN integration with intelligent routing
- Advanced gateway caching and optimization  
- Database query optimization and indexing
- Content deduplication system
- Performance benchmarking results
- Cache hit rate analysis

### Week 12: Advanced Features & API Enhancements

**Objectives:**
- Implement custom domain support for gateways
- Create comprehensive webhook system
- Add batch operation capabilities
- Develop advanced analytics and reporting

### Week 13: Operational Excellence & Automation

**Objectives:**
- Implement auto-scaling based on load metrics
- Create advanced monitoring and anomaly detection
- Develop capacity planning tools
- Implement automated performance optimization

### Week 14: Business Features & API Versioning  

**Objectives:**
- Implement user quota and billing integration
- Add advanced authentication (SSO, MFA)
- Create API versioning strategy
- Develop multi-tenancy support

---

## Phase 5: Enterprise Readiness (Weeks 15-18)

### Week 15-16: High Availability & Multi-Region

**Objectives:**
- Deploy multi-region IPFS clusters
- Implement geographic pin distribution
- Create disaster recovery procedures
- Test network partitioning resilience

### Week 17-18: Compliance & Security Hardening

**Objectives:**
- Implement GDPR compliance features
- Complete SOC2 audit preparation
- Add advanced security features
- Create regulatory compliance documentation

---

## Success Metrics & KPIs

### Technical Metrics

**Phase 1-2 (Foundation)**
- Single node uptime: 99.9%
- API response time: <200ms (95th percentile)
- Pin success rate: >99%
- File upload success rate: >99.5%

**Phase 3-4 (Production)**
- Cluster uptime: 99.99%
- Multi-node redundancy: 100% functional
- Pin replication success: >99.5%
- Gateway response time: <100ms additional latency
- Failover time: <30 seconds

**Phase 5 (Enterprise)**
- Multi-region uptime: 99.999%
- Global gateway response: <500ms (95th percentile)
- Zero-downtime deployments: 100% success
- Data consistency: 99.99% across regions

### Business Metrics

**User Experience**
- File upload completion rate: >98%
- Content retrieval success rate: >99.9%
- User satisfaction score: >4.5/5
- Support ticket resolution time: <4 hours

**Operational Efficiency**
- Infrastructure cost per GB stored: <$0.10/month
- Bandwidth cost efficiency: <$0.05/GB transferred
- Admin overhead: <2 hours/week for routine operations
- Automated alert resolution: >80%

### Scale Targets

**Storage Capacity**
- Phase 2: 1TB total capacity, 10K files
- Phase 3: 10TB total capacity, 100K files  
- Phase 4: 100TB total capacity, 1M files
- Phase 5: 1PB+ capacity, 10M+ files

**Performance Benchmarks**
- Concurrent uploads: 100+ simultaneous
- Gateway throughput: 1GB/s aggregate
- API throughput: 1000+ requests/second
- Database query performance: <50ms average

---

## Risk Management

### Technical Risks

**High Impact, High Probability**
- IPFS node instability → Comprehensive monitoring, automated recovery, redundancy
- Network partitioning → Multi-region deployment, conflict resolution procedures
- Database performance degradation → Read replicas, query optimization, caching

**High Impact, Medium Probability**
- Cluster consensus failures → Automated cluster healing, backup consensus mechanisms  
- Gateway overload → Load balancing, auto-scaling, CDN integration
- Data corruption → Integrity checks, automated backup, redundant storage

**Medium Impact, Various Probability**
- API rate limiting issues → Dynamic rate limiting, burst handling
- Storage cost overruns → Usage monitoring, automatic optimization
- Security vulnerabilities → Regular audits, automated scanning, rapid patching

### Mitigation Strategies

**Technical Resilience**
- Multi-region deployment with automatic failover
- Comprehensive monitoring with predictive alerting  
- Automated backup and recovery procedures
- Regular disaster recovery testing
- Performance testing under extreme load

**Operational Preparedness**
- 24/7 monitoring and alerting
- Escalation procedures for critical issues
- Runbooks for common scenarios
- Cross-trained team members
- Vendor relationship management

**Business Continuity**
- Service level agreement compliance
- Customer communication procedures
- Financial reserves for infrastructure scaling
- Legal compliance and audit preparedness
- Competitive analysis and feature parity

---

## Resource Allocation

### Team Requirements by Phase

**Phase 1-2 (Weeks 1-6)**
- 1x Senior Backend Developer (IPFS expertise)
- 1x DevOps Engineer (Docker, monitoring)
- 0.25x Security Consultant (part-time reviews)

**Phase 3-4 (Weeks 7-14)**  
- 2x Backend Developers (1 senior, 1 mid-level)
- 1x DevOps Engineer (Kubernetes, scaling)
- 1x Frontend Developer (dashboard, analytics)
- 0.5x Security Consultant (ongoing reviews)

**Phase 5 (Weeks 15-18)**
- 2x Backend Developers
- 1x Senior DevOps Engineer (multi-region)
- 1x Frontend Developer
- 1x Security Specialist (compliance)
- 0.5x Technical Writer (documentation)

### Infrastructure Costs

**Development Phase (1-2)**
- Cloud infrastructure: $200-300/month
- Monitoring tools: $100/month  
- Development tools: $200/month
- **Total: ~$500/month**

**Production Phase (3-4)**
- Multi-node cluster: $800-1200/month
- CDN and networking: $300-500/month
- Monitoring and logging: $300/month
- Backup and disaster recovery: $200/month
- **Total: ~$1600-2200/month**

**Enterprise Phase (5)**
- Multi-region infrastructure: $3000-5000/month
- Advanced security tools: $500/month
- Compliance and audit tools: $300/month
- Enhanced monitoring: $500/month
- **Total: ~$4300-6300/month**

### Timeline Buffers

**Built-in Contingencies**
- 15% time buffer for each phase
- 2-week integration testing buffer between phases
- 1-week buffer for unexpected issues or scope changes
- Additional 4 weeks for production hardening if needed

**Accelerated Timeline Options**
- Parallel development streams where possible
- Additional team members for critical path items  
- Pre-built components and proven patterns
- Reduced scope for initial MVP if time-critical

This comprehensive roadmap provides a structured approach to building a production-ready IPFS pinning platform with clear milestones, success criteria, and risk mitigation strategies. Each phase builds upon the previous one while maintaining focus on reliability, performance, and scalability.