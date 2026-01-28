# ARKA Engine Service Level Agreement (SLA)

**Version:** 1.0.0
**Effective Date:** December 1, 2024
**Last Updated:** December 2024

---

## Overview

This Service Level Agreement (SLA) defines the performance, availability, and support commitments for ARKA Engine. This document applies to all customers with an active Enterprise subscription.

---

## Table of Contents

1. [Service Availability](#service-availability)
2. [Performance Targets](#performance-targets)
3. [Support Response Times](#support-response-times)
4. [Maintenance Windows](#maintenance-windows)
5. [Service Credits](#service-credits)
6. [Exclusions](#exclusions)
7. [Monitoring & Reporting](#monitoring--reporting)

---

## Service Availability

### Uptime Commitment

| Service Tier | Monthly Uptime | Annual Downtime (Max) |
|--------------|----------------|----------------------|
| **Enterprise** | 99.99% | 52.6 minutes |
| **Professional** | 99.9% | 8.76 hours |
| **Starter** | 99.5% | 3.65 hours |

### Availability Calculation

```
Monthly Uptime % = ((Total Minutes - Downtime Minutes) / Total Minutes) × 100

Where:
- Total Minutes = Total minutes in the calendar month
- Downtime Minutes = Minutes where service was unavailable
```

### Service Components

| Component | Availability Target | Measurement |
|-----------|--------------------| ------------|
| **API Gateway** | 99.99% | HTTP 5xx error rate < 0.01% |
| **Core Engine (RTVM)** | 99.99% | Rule execution success rate |
| **Decision Service** | 99.99% | Decision response availability |
| **Event Ingestion** | 99.99% | Event acceptance rate |
| **Admin Dashboard** | 99.9% | Dashboard accessibility |
| **Reporting Service** | 99.9% | Report generation success |
| **Webhook Delivery** | 99.9% | Successful delivery rate |

### High Availability Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           Global Load Balancer          │
                    │              (Active-Active)            │
                    └──────────────────┬──────────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
    ┌─────────▼─────────┐    ┌─────────▼─────────┐    ┌─────────▼─────────┐
    │    Region A       │    │    Region B       │    │    Region C       │
    │   (Primary)       │    │   (Secondary)     │    │   (DR)            │
    │                   │    │                   │    │                   │
    │  ┌─────────────┐  │    │  ┌─────────────┐  │    │  ┌─────────────┐  │
    │  │ ARKA Core   │  │    │  │ ARKA Core   │  │    │  │ ARKA Core   │  │
    │  │ (3 nodes)   │  │    │  │ (3 nodes)   │  │    │  │ (3 nodes)   │  │
    │  └─────────────┘  │    │  └─────────────┘  │    │  └─────────────┘  │
    │                   │    │                   │    │                   │
    │  ┌─────────────┐  │    │  ┌─────────────┐  │    │  ┌─────────────┐  │
    │  │ PostgreSQL  │──┼────┼──│ PostgreSQL  │──┼────┼──│ PostgreSQL  │  │
    │  │ (Primary)   │  │    │  │ (Replica)   │  │    │  │ (Replica)   │  │
    │  └─────────────┘  │    │  └─────────────┘  │    │  └─────────────┘  │
    └───────────────────┘    └───────────────────┘    └───────────────────┘
```

---

## Performance Targets

### API Response Times

| Endpoint Category | P50 | P95 | P99 | Max |
|-------------------|-----|-----|-----|-----|
| **Health Check** | 5ms | 10ms | 20ms | 50ms |
| **Event Submission** | 15ms | 50ms | 100ms | 200ms |
| **Decision Query** | 20ms | 75ms | 150ms | 300ms |
| **Rule Evaluation** | 25ms | 100ms | 200ms | 500ms |
| **Entity Lookup** | 10ms | 30ms | 75ms | 150ms |
| **Alert Management** | 30ms | 100ms | 200ms | 400ms |
| **Report Generation** | 500ms | 2s | 5s | 30s |
| **Batch Processing** | N/A | N/A | N/A | 5min/10K events |

### Throughput Targets

| Metric | Starter | Professional | Enterprise |
|--------|---------|--------------|------------|
| **Events/Second** | 100 | 1,000 | 10,000+ |
| **Concurrent Connections** | 100 | 1,000 | 100,000+ |
| **Active Rules** | 100 | 1,000 | 10,000+ |
| **Daily Transactions** | 100K | 1M | Unlimited |
| **API Requests/Day** | 100K | 1M | 10M+ |

### Rule Execution Performance

| Metric | Target |
|--------|--------|
| Rule evaluation latency (simple) | < 10ms |
| Rule evaluation latency (complex) | < 50ms |
| Rule graph traversal | < 5ms |
| Decision caching hit rate | > 90% |
| Concurrent rule evaluations | 10,000+ |

### Data Processing

| Operation | Target |
|-----------|--------|
| Event ingestion to decision | < 100ms |
| Real-time alert generation | < 500ms |
| Webhook delivery | < 2s (first attempt) |
| Report generation (standard) | < 30s |
| Batch import (1M records) | < 10 minutes |

---

## Support Response Times

### Priority Definitions

| Priority | Definition | Examples |
|----------|------------|----------|
| **P1 - Critical** | Complete service outage or data integrity issue affecting production | API unavailable, data loss, security breach |
| **P2 - High** | Major functionality impaired, no workaround | Key feature broken, significant performance degradation |
| **P3 - Medium** | Functionality impaired with workaround | Non-critical feature issue, intermittent errors |
| **P4 - Low** | Minor issue or question | Documentation, feature requests, general questions |

### Response Time Commitments

| Priority | Enterprise | Professional | Starter |
|----------|------------|--------------|---------|
| **P1 - Critical** | 15 minutes | 1 hour | 4 hours |
| **P2 - High** | 1 hour | 4 hours | 8 hours |
| **P3 - Medium** | 4 hours | 8 hours | 24 hours |
| **P4 - Low** | 8 hours | 24 hours | 48 hours |

### Resolution Time Targets

| Priority | Enterprise | Professional | Starter |
|----------|------------|--------------|---------|
| **P1 - Critical** | 4 hours | 8 hours | 24 hours |
| **P2 - High** | 8 hours | 24 hours | 72 hours |
| **P3 - Medium** | 24 hours | 72 hours | 1 week |
| **P4 - Low** | 72 hours | 1 week | 2 weeks |

### Support Hours

| Tier | Support Hours | Channels |
|------|---------------|----------|
| **Enterprise** | 24/7/365 | Phone, Email, Chat, Portal, Dedicated Slack |
| **Professional** | 24/7 (P1/P2), Business Hours (P3/P4) | Email, Chat, Portal |
| **Starter** | Business Hours (9am-6pm ET, M-F) | Email, Portal |

### Support Channels

| Channel | Availability | Response Time |
|---------|--------------|---------------|
| **Emergency Hotline** | 24/7 (Enterprise) | Immediate |
| **Support Portal** | 24/7 | Per SLA |
| **Email** | 24/7 | Per SLA |
| **Live Chat** | Business Hours | < 5 minutes |
| **Dedicated Slack** | Enterprise only | < 30 minutes |

---

## Maintenance Windows

### Scheduled Maintenance

| Type | Frequency | Duration | Notice |
|------|-----------|----------|--------|
| **Routine Updates** | Weekly | 15-30 minutes | 48 hours |
| **Minor Releases** | Monthly | 30-60 minutes | 7 days |
| **Major Releases** | Quarterly | 1-2 hours | 14 days |
| **Infrastructure** | As needed | 1-4 hours | 7 days |

### Maintenance Schedule

- **Preferred Window:** Sundays 02:00-06:00 UTC
- **Secondary Window:** Wednesdays 02:00-04:00 UTC
- **Region-Specific:** Aligned with lowest traffic periods

### Zero-Downtime Deployments

For Enterprise tier:
- Rolling deployments with no service interruption
- Blue-green deployment capability
- Canary releases for gradual rollout
- Instant rollback capability

### Maintenance Notifications

```
Notification Timeline:
├── 14 days: Major release announcement
├── 7 days: Detailed maintenance notice
├── 48 hours: Reminder with specifics
├── 24 hours: Final confirmation
├── 1 hour: Maintenance beginning soon
├── 0: Maintenance start
└── Complete: Maintenance complete notification
```

---

## Service Credits

### Credit Calculation

| Monthly Uptime | Service Credit |
|----------------|----------------|
| 99.9% - 99.99% | 10% |
| 99.0% - 99.9% | 25% |
| 95.0% - 99.0% | 50% |
| < 95.0% | 100% |

### Credit Request Process

1. **Submit Request:** Within 30 days of incident via support portal
2. **Provide Details:**
   - Incident date and time
   - Duration of impact
   - Affected services
   - Business impact description
3. **Review:** Credits processed within 15 business days
4. **Application:** Applied to next billing cycle

### Maximum Credits

- Maximum credit per month: 100% of monthly fee
- Credits are non-transferable and non-refundable
- Credits apply to subscription fees only

### Credit Calculation Example

```
Scenario: 99.85% uptime in a month (instead of 99.99%)

Monthly Fee: $10,000
Downtime: 64.8 minutes (vs. allowed 4.3 minutes)
Uptime Achieved: 99.85%
Credit Tier: 10%
Credit Amount: $1,000

Applied: Next month's invoice reduced by $1,000
```

---

## Exclusions

### SLA Exclusions

The following are excluded from availability calculations:

1. **Scheduled Maintenance**
   - Pre-announced maintenance windows
   - Emergency security patches (with notification)

2. **Customer-Caused Issues**
   - Misconfiguration by customer
   - Customer network issues
   - Exceeding rate limits
   - Invalid API usage

3. **External Factors**
   - Force majeure events
   - Internet connectivity issues beyond our control
   - Third-party service outages (when documented)
   - DNS issues outside our infrastructure

4. **Alpha/Beta Features**
   - Features explicitly marked as beta
   - Preview functionality
   - Experimental endpoints

5. **Customer Environment**
   - On-premises deployment issues
   - Customer-managed infrastructure
   - Customer's cloud provider outages

### Force Majeure

SLA commitments are suspended during:
- Natural disasters
- War or terrorism
- Government actions
- Widespread internet outages
- Pandemic-related disruptions

---

## Monitoring & Reporting

### Real-Time Monitoring

| Metric | Monitoring Frequency | Alert Threshold |
|--------|---------------------|-----------------|
| API availability | 10 seconds | < 99.9% (5 min) |
| Response latency | 10 seconds | P95 > 2x target |
| Error rate | 1 minute | > 1% |
| Throughput | 1 minute | < 50% normal |
| Database health | 30 seconds | Connection issues |
| Queue depth | 1 minute | > 1000 pending |

### Status Page

- **URL:** https://status.arka-engine.io
- **Updates:** Real-time
- **Historical Data:** 90 days
- **Subscription:** Email, SMS, Webhook, RSS

### Monthly Reports

Enterprise customers receive:

| Report | Contents | Delivery |
|--------|----------|----------|
| **Availability Report** | Uptime %, incidents, root causes | 5th of month |
| **Performance Report** | Latency trends, throughput stats | 5th of month |
| **Security Report** | Security events, compliance status | 5th of month |
| **Usage Report** | API calls, events processed, storage | 5th of month |

### Quarterly Business Reviews

Enterprise customers receive:
- Executive summary of service performance
- Trend analysis and capacity planning
- Roadmap preview
- Optimization recommendations
- Dedicated account manager review

---

## SLA Versions

| Version | Effective Date | Changes |
|---------|----------------|---------|
| 1.0 | December 1, 2024 | Initial release |

---

## Contact

**Support Portal:** https://support.arka-engine.io
**Emergency Hotline:** +1-800-ARKA-911
**Email:** support@arka-engine.io
**Status Page:** https://status.arka-engine.io

---

*This SLA is subject to the terms of your Master Service Agreement with ARKA Engine.*
