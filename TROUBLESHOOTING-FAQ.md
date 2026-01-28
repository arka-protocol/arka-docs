# ARKA Engine Troubleshooting & FAQ

**Version:** 1.0.0
**Last Updated:** December 2024

## Table of Contents

1. [Common Issues & Solutions](#common-issues--solutions)
2. [Installation Issues](#installation-issues)
3. [Configuration Issues](#configuration-issues)
4. [Performance Issues](#performance-issues)
5. [Rule Execution Issues](#rule-execution-issues)
6. [API Issues](#api-issues)
7. [Plugin Issues](#plugin-issues)
8. [Database Issues](#database-issues)
9. [Integration Issues](#integration-issues)
10. [Frequently Asked Questions](#frequently-asked-questions)
11. [Error Code Reference](#error-code-reference)
12. [Diagnostic Tools](#diagnostic-tools)
13. [Getting Support](#getting-support)

---

## Common Issues & Solutions

### System Won't Start

**Symptoms:**
- Service fails to start
- Immediate crash after startup
- No response on health endpoint

**Solutions:**

1. **Check logs for errors:**
   ```bash
   arka-cli logs --service core --tail 100
   # or
   docker logs arka-core
   ```

2. **Verify configuration:**
   ```bash
   arka-cli config validate
   ```

3. **Check database connectivity:**
   ```bash
   arka-cli diag database
   ```

4. **Verify port availability:**
   ```bash
   netstat -tlnp | grep 3000
   lsof -i :3000
   ```

5. **Check resource limits:**
   ```bash
   ulimit -a
   free -h
   df -h
   ```

### High Memory Usage

**Symptoms:**
- OOM (Out of Memory) errors
- Slow response times
- Container restarts

**Solutions:**

1. **Check current memory usage:**
   ```bash
   arka-cli diag memory
   ```

2. **Increase memory limits:**
   ```yaml
   # docker-compose.yml
   services:
     arka-core:
       deploy:
         resources:
           limits:
             memory: 8G
   ```

3. **Optimize caching:**
   ```yaml
   # pact.config.yaml
   cache:
     maxSize: 1000
     ttl: 1800
   ```

4. **Reduce connection pool:**
   ```yaml
   database:
     pool:
       max: 25
   ```

### Slow Response Times

**Symptoms:**
- API latency > 1 second
- Timeout errors
- User complaints

**Solutions:**

1. **Check database query performance:**
   ```bash
   arka-cli diag queries --slow
   ```

2. **Enable query caching:**
   ```yaml
   cache:
     queries: true
     queryTtl: 300
   ```

3. **Add database indexes:**
   ```bash
   arka-cli db optimize --analyze --suggest-indexes
   ```

4. **Scale horizontally:**
   ```bash
   kubectl scale deployment arka-core --replicas=5
   ```

---

## Installation Issues

### Q: `pnpm install` fails with permission errors

**Error:**
```
EACCES: permission denied, mkdir '/usr/local/lib/node_modules'
```

**Solution:**
```bash
# Fix npm permissions
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Q: Docker build fails with "no space left on device"

**Error:**
```
Error: ENOSPC: no space left on device
```

**Solution:**
```bash
# Clean up Docker resources
docker system prune -a
docker volume prune

# Check disk space
df -h

# Remove unused images
docker image prune -a
```

### Q: TypeScript compilation errors

**Error:**
```
error TS2307: Cannot find module '@arka/core'
```

**Solution:**
```bash
# Clean and rebuild
rm -rf node_modules
rm -rf dist
pnpm install
pnpm build
```

### Q: Database migration fails

**Error:**
```
Migration failed: relation "rules" already exists
```

**Solution:**
```bash
# Check migration status
arka-cli db migrations status

# Reset and re-run (CAUTION: destroys data)
arka-cli db migrations reset --force
arka-cli db migrations run

# Or fix specific migration
arka-cli db migrations fix --migration 20240101_001
```

---

## Configuration Issues

### Q: Environment variables not being read

**Symptoms:**
- Default values used instead of configured
- Connection failures

**Checks:**
```bash
# Verify env file exists
cat .env

# Check env is loaded
arka-cli config show --include-env

# Verify no typos
grep DATABASE_URL .env
```

**Solution:**
Ensure `.env` file is in the correct directory and properly formatted:
```bash
# Good
DATABASE_URL=postgresql://user:pass@host:5432/db

# Bad (spaces around =)
DATABASE_URL = postgresql://user:pass@host:5432/db

# Bad (quotes in value)
DATABASE_URL="postgresql://user:pass@host:5432/db"
```

### Q: SSL certificate errors

**Error:**
```
Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE
```

**Solutions:**

1. **For self-signed certs (development only):**
   ```bash
   NODE_TLS_REJECT_UNAUTHORIZED=0 arka-cli start
   ```

2. **For production, add CA cert:**
   ```yaml
   database:
     ssl:
       ca: /path/to/ca-cert.pem
       cert: /path/to/client-cert.pem
       key: /path/to/client-key.pem
   ```

### Q: CORS errors in browser

**Error:**
```
Access to XMLHttpRequest blocked by CORS policy
```

**Solution:**
```yaml
# pact.config.yaml
api:
  cors:
    enabled: true
    origins:
      - https://your-app.example.com
      - http://localhost:3000
    credentials: true
```

---

## Performance Issues

### Q: Rules taking too long to evaluate

**Symptoms:**
- Timeout errors during rule execution
- High RTVM CPU usage

**Diagnosis:**
```bash
arka-cli diag rules --profile
```

**Solutions:**

1. **Optimize complex rules:**
   - Break into smaller rules
   - Use early exit conditions
   - Cache expensive lookups

2. **Increase timeout:**
   ```yaml
   rtvm:
     executionTimeout: 60000  # 60 seconds
   ```

3. **Add more RTVM workers:**
   ```yaml
   rtvm:
     workers: 8  # Increase from default
   ```

### Q: Database connection pool exhausted

**Error:**
```
Error: Unable to acquire a connection from the pool
```

**Solution:**
```yaml
# Increase pool size
database:
  pool:
    min: 10
    max: 100
    acquireTimeout: 30000
```

Also check for:
- Connection leaks (transactions not committed/rolled back)
- Long-running queries blocking connections

### Q: Redis cache not helping performance

**Checks:**
```bash
# Check cache hit rate
arka-cli metrics cache

# Verify Redis connectivity
redis-cli ping
```

**Solutions:**
1. Increase cache TTL for stable data
2. Review cache key patterns
3. Check Redis memory limits

---

## Rule Execution Issues

### Q: Rule not triggering when expected

**Debugging Steps:**

1. **Test rule in sandbox:**
   ```bash
   arka-cli rule test rule-123 --input transaction.json
   ```

2. **Check rule is active:**
   ```bash
   arka-cli rule show rule-123
   ```

3. **Verify conditions:**
   ```bash
   arka-cli rule explain rule-123 --input transaction.json
   ```

4. **Check rule priority:**
   Higher priority rules execute first and may short-circuit.

### Q: Rule triggering incorrectly (false positives)

**Solutions:**

1. **Add more specific conditions:**
   ```yaml
   conditions:
     - field: amount
       operator: greater_than
       value: 10000
     - field: transaction_type  # Add specificity
       operator: equals
       value: wire_transfer
   ```

2. **Add exception conditions:**
   ```yaml
   exceptions:
     - field: entity.type
       operator: equals
       value: verified_institution
   ```

3. **Use the sandbox to test edge cases**

### Q: "Rule evaluation timeout" errors

**Solutions:**

1. **Simplify the rule:**
   - Remove nested conditions
   - Avoid recursive lookups

2. **Increase timeout:**
   ```yaml
   rtvm:
     executionTimeout: 60000
   ```

3. **Pre-compute expensive values:**
   - Cache entity risk scores
   - Pre-calculate aggregations

---

## API Issues

### Q: 401 Unauthorized errors

**Causes & Solutions:**

1. **Expired token:**
   ```bash
   # Refresh token
   curl -X POST /auth/refresh -d '{"refreshToken":"..."}'
   ```

2. **Invalid API key:**
   ```bash
   # Verify key
   arka-cli apikey verify your_key

   # Generate new key
   arka-cli apikey create --name "new-key"
   ```

3. **Missing header:**
   ```bash
   # Correct format
   curl -H "Authorization: Bearer your_token" ...
   # or
   curl -H "X-API-Key: your_key" -H "X-API-Secret: your_secret" ...
   ```

### Q: 429 Too Many Requests

**Error:**
```json
{"error": "Rate limit exceeded", "retryAfter": 60}
```

**Solutions:**

1. **Implement exponential backoff:**
   ```javascript
   async function fetchWithRetry(url, retries = 3) {
     for (let i = 0; i < retries; i++) {
       const response = await fetch(url);
       if (response.status !== 429) return response;
       await sleep(Math.pow(2, i) * 1000);
     }
   }
   ```

2. **Upgrade rate limits:**
   Contact admin to increase limits for your API key.

3. **Batch requests:**
   Use batch endpoints instead of individual calls.

### Q: Request body validation errors

**Error:**
```json
{"error": "Validation failed", "details": [...]}
```

**Debugging:**
```bash
# Validate request against schema
arka-cli api validate --endpoint /events --body request.json
```

**Common issues:**
- Missing required fields
- Wrong data types (string vs number)
- Invalid enum values
- Date format (use ISO 8601)

---

## Plugin Issues

### Q: Plugin fails to load

**Error:**
```
Error: Failed to load plugin @arka/plugin-xyz
```

**Solutions:**

1. **Verify installation:**
   ```bash
   arka-cli plugin list --installed
   ```

2. **Check compatibility:**
   ```bash
   arka-cli plugin check @arka/plugin-xyz
   ```

3. **Reinstall plugin:**
   ```bash
   arka-cli plugin uninstall @arka/plugin-xyz
   arka-cli plugin install @arka/plugin-xyz
   ```

4. **Check plugin configuration:**
   ```bash
   arka-cli plugin config show @arka/plugin-xyz
   ```

### Q: Plugin conflicts with another plugin

**Symptoms:**
- One plugin works, another fails
- Unexpected behavior when both enabled

**Solutions:**

1. **Check for version conflicts:**
   ```bash
   arka-cli plugin deps
   ```

2. **Update both plugins:**
   ```bash
   arka-cli plugin update --all
   ```

3. **Check load order:**
   ```yaml
   plugins:
     loadOrder:
       - @arka/plugin-audit-log  # Load first
       - @arka/plugin-aml-monitor
   ```

---

## Database Issues

### Q: "Connection refused" to database

**Checks:**
```bash
# Test connectivity
pg_isready -h hostname -p 5432

# Check firewall
telnet hostname 5432

# Verify credentials
psql -h hostname -U user -d database
```

**Solutions:**
1. Verify `DATABASE_URL` is correct
2. Check firewall rules
3. Verify PostgreSQL is running
4. Check pg_hba.conf for allowed connections

### Q: Slow queries degrading performance

**Diagnosis:**
```bash
# Find slow queries
arka-cli diag queries --slow --threshold 1000

# Check missing indexes
arka-cli db analyze --suggest-indexes
```

**Solutions:**

1. **Add indexes:**
   ```sql
   CREATE INDEX idx_events_timestamp ON events(timestamp);
   CREATE INDEX idx_decisions_entity ON decisions(entity_id);
   ```

2. **Update statistics:**
   ```sql
   ANALYZE events;
   ANALYZE decisions;
   ```

3. **Vacuum tables:**
   ```sql
   VACUUM ANALYZE events;
   ```

### Q: Database disk space running low

**Solutions:**

1. **Archive old data:**
   ```bash
   arka-cli db archive --older-than 180d
   ```

2. **Clean up logs:**
   ```bash
   arka-cli maintenance cleanup --logs --older-than 30d
   ```

3. **Vacuum to reclaim space:**
   ```sql
   VACUUM FULL;
   ```

---

## Integration Issues

### Q: Webhook deliveries failing

**Diagnosis:**
```bash
arka-cli webhook status
arka-cli webhook logs --failed
```

**Common causes:**
1. Target endpoint down
2. SSL certificate issues
3. Timeout (default 30s)
4. Invalid response (non-2xx)

**Solutions:**

1. **Check endpoint:**
   ```bash
   curl -X POST https://your-webhook.com/endpoint
   ```

2. **Increase timeout:**
   ```yaml
   webhooks:
     timeout: 60000
     retries: 5
   ```

3. **Check logs for specific errors:**
   ```bash
   arka-cli webhook logs --id webhook-123
   ```

### Q: Kafka/Message queue integration issues

**Symptoms:**
- Messages not being consumed
- High lag

**Diagnosis:**
```bash
# Check Kafka connectivity
arka-cli diag kafka

# Check consumer lag
kafka-consumer-groups.sh --describe --group arka-consumers
```

**Solutions:**
1. Verify broker connectivity
2. Check topic permissions
3. Review consumer configuration
4. Scale consumers if lagging

---

## Frequently Asked Questions

### General

**Q: What is the maximum number of rules ARKA Engine can handle?**
A: ARKA Engine has been tested with 10,000+ active rules. Performance depends on rule complexity and hardware resources.

**Q: How long are audit logs retained?**
A: By default, 7 years for compliance. Configurable via `audit.retentionDays`.

**Q: Can I run ARKA Engine on-premises?**
A: Yes, ARKA Engine supports on-premises, cloud, and hybrid deployments.

**Q: What happens if the system goes down during a transaction?**
A: Transactions in progress are preserved in the queue. Upon restart, they are automatically reprocessed.

### Rules

**Q: How do I test a rule without affecting production?**
A: Use the Testing Sandbox (`/sandbox`) to simulate transactions against rules in an isolated environment.

**Q: Can rules be scheduled to activate in the future?**
A: Yes, use the `effectiveDate` and `expirationDate` fields when creating rules.

**Q: What's the difference between FLAG and DENY?**
A: FLAG allows the transaction but creates an alert for review. DENY blocks the transaction immediately.

### Security

**Q: Is data encrypted at rest?**
A: Yes, using AES-256 encryption. Encryption keys are managed via environment configuration or HSM.

**Q: How are API keys secured?**
A: API keys are hashed using bcrypt before storage. The secret is only shown once at creation.

**Q: Does ARKA Engine support SSO?**
A: Yes, SAML 2.0 and OAuth 2.0/OIDC are supported for enterprise SSO.

### Operations

**Q: How do I perform a zero-downtime upgrade?**
A: Use rolling deployment strategy. ARKA Engine supports blue-green and canary deployments.

**Q: What's the recommended backup strategy?**
A: Hourly database backups with 30-day retention, plus continuous WAL archiving for point-in-time recovery.

**Q: Can I export all data for regulatory audit?**
A: Yes, use `arka-cli export audit --format csv` to export complete audit trails.

---

## Error Code Reference

### HTTP Status Codes

| Code | Meaning | Common Cause |
|------|---------|--------------|
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Invalid/expired token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate or version conflict |
| 422 | Unprocessable | Validation failed |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Error | Server-side error |
| 502 | Bad Gateway | Upstream service down |
| 503 | Service Unavailable | Maintenance mode |

### ARKA Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| `ARKA_E001` | Database connection failed | Check DATABASE_URL |
| `ARKA_E002` | Redis connection failed | Check REDIS_URL |
| `ARKA_E003` | Rule execution timeout | Increase timeout or simplify rule |
| `ARKA_E004` | Plugin load failed | Reinstall plugin |
| `ARKA_E005` | Configuration invalid | Run config validate |
| `ARKA_E006` | License expired | Contact sales |
| `ARKA_E007` | Rate limit exceeded | Wait or upgrade |
| `ARKA_E008` | Insufficient permissions | Check user roles |
| `ARKA_E009` | Entity not found | Verify entity ID |
| `ARKA_E010` | Rule conflict detected | Review conflicting rules |

---

## Diagnostic Tools

### Built-in Diagnostics

```bash
# Full system diagnostic
arka-cli diag full

# Specific components
arka-cli diag database
arka-cli diag redis
arka-cli diag rtvm
arka-cli diag api

# Performance profiling
arka-cli diag profile --duration 60

# Memory analysis
arka-cli diag memory --heap-dump

# Network connectivity
arka-cli diag network
```

### Health Endpoints

```bash
# Basic health
curl http://localhost:3000/health

# Detailed health
curl http://localhost:3000/health/detailed

# Readiness
curl http://localhost:3000/health/ready

# Liveness
curl http://localhost:3000/health/live
```

### Log Analysis

```bash
# View logs
arka-cli logs --tail 1000

# Filter by level
arka-cli logs --level error

# Search logs
arka-cli logs --grep "connection failed"

# Export logs
arka-cli logs export --start "2024-01-01" --end "2024-01-31" --output logs.tar.gz
```

---

## Getting Support

### Self-Service Resources

- **Documentation**: https://docs.arka-engine.io
- **Knowledge Base**: https://help.arka-engine.io
- **Community Forum**: https://community.arka-engine.io
- **Status Page**: https://status.arka-engine.io

### Support Channels

| Priority | Channel | Response Time |
|----------|---------|---------------|
| P1 (Critical) | Emergency Hotline | < 15 min |
| P2 (High) | support@arka-engine.io | < 4 hours |
| P3 (Medium) | Support Portal | < 24 hours |
| P4 (Low) | Community Forum | Best effort |

### Before Contacting Support

Please gather:
1. ARKA Engine version (`arka-cli version`)
2. Error messages and codes
3. Steps to reproduce
4. Relevant logs (`arka-cli logs export`)
5. Diagnostic report (`arka-cli diag full`)

### Emergency Contacts

For production outages:
- **Phone**: +1-800-ARKA-911
- **Email**: emergency@arka-engine.io
- **PagerDuty**: Provide your organization's integration key
