# Redis 7 - StatefulSet

## Overview

Production-ready Redis 7 deployment for caching and session management:
- StatefulSet with persistent storage (10GB)
- Password authentication
- AOF + RDB persistence
- LRU eviction policy
- Health checks and monitoring
- Optimized for 512MB memory

## Deployment

### Prerequisites

Secret must exist:
```bash
kubectl get secret redis-credentials -n database
```

### Deploy

```bash
kubectl apply -k .
```

### Verify

```bash
kubectl get pods -n database -l app=redis
kubectl logs -n database redis-0 -f
```

## Usage

### Connection

**Host**: `redis-client.database.svc.cluster.local`
**Port**: `6379`
**Password**: From secret `redis-credentials`

**Spring Boot:**
```yaml
spring:
  redis:
    host: redis-client.database.svc.cluster.local
    port: 6379
    password: ${REDIS_PASSWORD}
```

### Testing

```bash
# Port forward
kubectl port-forward -n database svc/redis-client 6379:6379

# Connect
redis-cli -h localhost -a <password>

# Test
127.0.0.1:6379> SET test "Hello"
127.0.0.1:6379> GET test
"Hello"
```

## Configuration

### Memory

- **Max Memory**: 400MB
- **Eviction Policy**: `allkeys-lru` (least recently used)
- **Persistence**: AOF + RDB
- **AOF Sync**: `everysec`

### Persistence

**RDB Snapshots:**
- Every 900s if 1 key changed
- Every 300s if 10 keys changed
- Every 60s if 10,000 keys changed

**AOF (Append Only File):**
- Enabled with `everysec` fsync
- Auto-rewrite at 100% growth, min 64MB

## Monitoring

### Metrics

```bash
# Connect to Redis
kubectl exec -it -n database redis-0 -- redis-cli -a <password>

# Server info
INFO server

# Memory usage
INFO memory

# Stats
INFO stats

# Slow queries
SLOWLOG GET 10

# Client list
CLIENT LIST
```

### Common Commands

```bash
# Key count
DBSIZE

# Memory usage
MEMORY USAGE <key>

# Flush all (DANGEROUS)
FLUSHALL

# Flush database
FLUSHDB

# Get config
CONFIG GET *
```

## Troubleshooting

### Connection Refused

```bash
kubectl describe pod -n database redis-0
kubectl logs -n database redis-0

# Check service endpoints
kubectl get endpoints -n database redis-client
```

### High Memory

```bash
# Check memory
kubectl exec -n database redis-0 -- redis-cli -a <password> INFO memory

# Check largest keys
kubectl exec -n database redis-0 -- redis-cli -a <password> \
  --bigkeys
```

### Performance

```bash
# Check slow log
kubectl exec -n database redis-0 -- redis-cli -a <password> SLOWLOG GET 20

# Latency
kubectl exec -n database redis-0 -- redis-cli -a <password> --latency

# Check resource usage
kubectl top pod -n database redis-0
```

## Scaling

### Vertical Scaling

Update resources:
```yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

Update maxmemory in ConfigMap:
```
maxmemory 800mb
```

### Horizontal Scaling (Cluster Mode - Future)

For production HA:
1. Deploy Redis Cluster (6 nodes minimum)
2. Use Redis Sentinel for failover
3. Or use Redis Cluster mode with sharding

## Security

- Password authentication required
- Runs as non-root user
- Network policies restrict access
- No dangerous commands exposed

## References

- [Redis Documentation](https://redis.io/documentation)
- [Redis Persistence](https://redis.io/topics/persistence)
- [Redis Configuration](https://redis.io/topics/config)
