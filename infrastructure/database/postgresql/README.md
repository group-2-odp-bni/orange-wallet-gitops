# PostgreSQL 16 - StatefulSet

## Overview

Production-ready PostgreSQL 16 deployment on Kubernetes with:
- StatefulSet for stable storage and DNS
- Persistent volumes (30GB SSD)
- Automated daily backups to GCS
- Health checks and monitoring
- Optimized configuration for 2GB RAM
- Runs on dedicated stateful node

## Deployment

### Prerequisites

1. **Secrets must exist** (created via External Secrets):
   ```bash
   kubectl get secret postgres-credentials -n database
   ```

2. **Backup service account** configured:
   ```bash
   # Verify GCP service account exists
   gcloud iam service-accounts describe k8s-backup-sa@PROJECT.iam.gserviceaccount.com
   ```

### Deploy

```bash
# Apply with kustomize
kubectl apply -k .

# Or individual files
kubectl apply -f configmap.yaml
kubectl apply -f statefulset.yaml
kubectl apply -f service.yaml
kubectl apply -f backup-cronjob.yaml
```

### Verify

```bash
# Check pod status
kubectl get pods -n database -l app=postgresql

# Check StatefulSet
kubectl get statefulset -n database

# Check PVC
kubectl get pvc -n database

# View logs
kubectl logs -n database postgresql-0 -f

# Check backup cronjob
kubectl get cronjob -n database
```

## Configuration

### Resource Allocation

- **CPU**: 500m request, 2000m limit
- **Memory**: 1Gi request, 2Gi limit
- **Storage**: 30Gi persistent disk (SSD)

### PostgreSQL Settings

Optimized for 2GB RAM:
- `shared_buffers = 512MB` (25% of RAM)
- `effective_cache_size = 1GB` (50% of RAM)
- `work_mem = 10MB`
- `max_connections = 100`

### Node Placement

Runs on stateful worker node only:
- **Toleration**: `stateful=true:NoSchedule`
- **NodeSelector**: `workload-type=stateful`
- **PriorityClass**: `database-high`

## Usage

### Connection Strings

**From within cluster:**
```
# Headless service (StatefulSet)
postgresql-0.postgresql.database.svc.cluster.local:5432

# Client service (load balanced)
postgresql-client.database.svc.cluster.local:5432
```

**Connection details:**
- Host: `postgresql-client.database.svc.cluster.local`
- Port: `5432`
- Database: `orange_wallet_db`
- User: `orange_wallet` or `postgres`
- Password: From secret `postgres-credentials`

**Spring Boot application.yml:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgresql-client.database.svc.cluster.local:5432/orange_wallet_db
    username: orange_wallet
    password: ${POSTGRES_PASSWORD}
```

### Database Access

```bash
# Connect via kubectl
kubectl exec -it -n database postgresql-0 -- psql -U postgres -d orange_wallet_db

# Port forward for local access
kubectl port-forward -n database svc/postgresql-client 5432:5432

# Connect from local machine
psql -h localhost -U postgres -d orange_wallet_db
```

## Backup & Restore

### Automated Backups

- **Schedule**: Daily at 2 AM Jakarta time
- **Location**: `gs://orange-wallet-backups/postgresql/`
- **Retention**: 30 days
- **Format**: Compressed SQL dump (`.sql.gz`)

### Manual Backup

```bash
# Trigger backup job manually
kubectl create job -n database \
  --from=cronjob/postgresql-backup \
  postgresql-backup-manual-$(date +%Y%m%d)

# View backup logs
kubectl logs -n database job/postgresql-backup-manual-YYYYMMDD
```

### List Backups

```bash
# List all backups
gsutil ls gs://orange-wallet-backups/postgresql/

# Get backup details
gsutil ls -l gs://orange-wallet-backups/postgresql/
```

### Restore from Backup

```bash
# 1. Download backup
gsutil cp gs://orange-wallet-backups/postgresql/postgresql-YYYYMMDD-HHMMSS.sql.gz .

# 2. Extract
gunzip postgresql-YYYYMMDD-HHMMSS.sql.gz

# 3. Copy to pod
kubectl cp -n database postgresql-YYYYMMDD-HHMMSS.sql postgresql-0:/tmp/

# 4. Restore
kubectl exec -it -n database postgresql-0 -- bash
psql -U postgres -d orange_wallet_db < /tmp/postgresql-YYYYMMDD-HHMMSS.sql
```

## Monitoring

### Health Checks

- **Liveness**: `pg_isready` every 10s
- **Readiness**: `pg_isready` every 5s
- **Startup**: `pg_isready` every 10s (5 min max)

### Metrics

PostgreSQL metrics exposed via `pg_stat_statements`:

```sql
-- View slow queries
SELECT
  query,
  calls,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Database size
SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname))
FROM pg_database;

-- Table sizes
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Logs

```bash
# View real-time logs
kubectl logs -n database postgresql-0 -f

# View backup job logs
kubectl logs -n database -l app=postgresql-backup

# View last 100 lines
kubectl logs -n database postgresql-0 --tail=100
```

## Troubleshooting

### Pod Not Starting

```bash
# Describe pod
kubectl describe pod -n database postgresql-0

# Check events
kubectl get events -n database --sort-by='.lastTimestamp'

# Check PVC status
kubectl get pvc -n database
```

### Connection Issues

```bash
# Test connection from debug pod
kubectl run -it --rm debug --image=postgres:16-alpine --restart=Never -- \
  psql -h postgresql-client.database.svc.cluster.local -U postgres -d orange_wallet_db

# Check service endpoints
kubectl get endpoints -n database postgresql-client
```

### Backup Failures

```bash
# Check backup job status
kubectl get jobs -n database -l app=postgresql-backup

# View failed job logs
kubectl logs -n database job/postgresql-backup-TIMESTAMP

# Check GCS bucket permissions
kubectl exec -n database postgresql-0 -- gsutil ls gs://orange-wallet-backups/
```

### Performance Issues

```bash
# Check resource usage
kubectl top pod -n database postgresql-0

# Check database connections
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# Check slow queries
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -d orange_wallet_db -c \
  "SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 5;"
```

## Scaling

### Vertical Scaling

Edit StatefulSet resources:
```bash
kubectl edit statefulset -n database postgresql

# Update resources section:
resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 3000m
    memory: 4Gi
```

### Storage Expansion

```bash
# Expand PVC (supported by GCP)
kubectl patch pvc -n database postgres-data-postgresql-0 \
  -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Verify expansion
kubectl get pvc -n database
```

### High Availability (Future)

For HA setup:
1. Increase replicas to 3
2. Configure streaming replication
3. Use Patroni or Stolon for failover
4. Use pgpool for load balancing

## Security

- Runs as non-root user (UID 999)
- File system group set to postgres
- Secrets managed via External Secrets Operator
- Network policies restrict access (see ../../../security/network-policies/)
- SSL/TLS connections supported (configure in postgresql.conf)

## Maintenance

### Vacuum

```bash
# Manual vacuum
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -d orange_wallet_db -c "VACUUM ANALYZE;"

# Check autovacuum status
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -d orange_wallet_db -c \
  "SELECT * FROM pg_stat_user_tables WHERE n_dead_tup > 1000;"
```

### Analyze

```bash
# Update statistics
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -d orange_wallet_db -c "ANALYZE;"
```

### Reindex

```bash
# Reindex database
kubectl exec -n database postgresql-0 -- \
  psql -U postgres -d orange_wallet_db -c "REINDEX DATABASE orange_wallet_db;"
```

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/16/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [PostgreSQL Tuning](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)
