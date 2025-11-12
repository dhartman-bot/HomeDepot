# Home Depot Deployment Checklist

## Description
Guide deployment process following Home Depot's safety and reliability standards.

Use this skill when developer:
- Is ready to deploy to production
- Wants to deploy to staging/production
- Asks "how do I deploy?"
- Creates deployment PR

Activate for phrases like: "deploy to prod", "production deployment", "ready to ship", "deploy this"

## Instructions

Follow Home Depot's deployment checklist to ensure safe, reliable deployments.

### Pre-Deployment Validation

#### 1. Code Quality Checks

**All Tests Must Pass:**
```bash
# Run full test suite
mvn clean verify
npm run test:all

# Check coverage
mvn jacoco:report
# Verify coverage ≥ 80%

# Integration tests
mvn failsafe:integration-test
```

**Verify:**
- [ ] All unit tests passing
- [ ] All integration tests passing
- [ ] No flaky tests
- [ ] Test coverage ≥ 80%
- [ ] Load tests completed (for high-traffic features)

#### 2. Security Validation

**Security Scans:**
```bash
# Dependency vulnerabilities
mvn dependency-check:check
npm audit

# Static security analysis
mvn spotbugs:check

# Secrets scanning
git secrets --scan
```

**Verify:**
- [ ] No critical or high vulnerabilities
- [ ] No hardcoded secrets/credentials
- [ ] Security review completed and approved
- [ ] PCI compliance check passed (if payment-related)

#### 3. Code Review

**Required Approvals:**
- [ ] At least 2 peer reviews approved
- [ ] Security team approval (if touches sensitive data)
- [ ] Architecture team approval (if changes architecture)
- [ ] Product owner approval (if customer-facing)

**Review Quality:**
- [ ] All review comments addressed
- [ ] No unresolved discussions
- [ ] Changes tested by reviewer

#### 4. Documentation

**Required Documentation:**
- [ ] README updated (if public API changed)
- [ ] API documentation updated (Swagger/OpenAPI)
- [ ] CHANGELOG.md entry added
- [ ] Runbook updated (if operational changes)
- [ ] Release notes drafted

**Runbook Must Include:**
- What changed
- How to verify deployment success
- How to rollback
- Known issues or limitations
- Monitoring dashboards to watch

#### 5. Database Migrations

**If Database Changes:**
- [ ] Migration scripts tested locally
- [ ] Migration scripts tested on staging
- [ ] Migrations are backwards compatible
- [ ] Rollback script prepared
- [ ] Data backup completed
- [ ] Migration time estimated

**Migration Safety:**
```sql
-- ✅ SAFE: Add nullable column (no downtime)
ALTER TABLE orders ADD COLUMN gift_message VARCHAR(500);

-- ❌ UNSAFE: Drop column (breaks old code)
ALTER TABLE orders DROP COLUMN legacy_field;

-- Better: Two-phase deployment
-- Phase 1: Stop writing to column, deploy code
-- Phase 2: Drop column after confirmed safe
```

#### 6. Feature Flags

**For Risky Changes:**
- [ ] Feature flag created in LaunchDarkly
- [ ] Flag disabled by default
- [ ] Gradual rollout plan documented
- [ ] Rollback = flip flag off (no code deploy needed)

**Feature Flag Pattern:**
```java
if (launchDarkly.isEnabled("new-checkout-flow", user)) {
    return newCheckoutService.process(order);
} else {
    return legacyCheckoutService.process(order);
}
```

### Deployment Process

#### 1. Staging Deployment

**Deploy to Staging First:**
```bash
# Build and tag
mvn clean package
docker build -t homedepot/service-name:v1.2.3 .

# Push to staging registry
docker push registry.staging.homedepot.com/service-name:v1.2.3

# Deploy to staging
kubectl apply -f k8s/staging/ -n homedepot-staging
kubectl set image deployment/service-name service-name=registry.staging.homedepot.com/service-name:v1.2.3 -n homedepot-staging
```

**Staging Validation:**
- [ ] Deployment successful (pods running)
- [ ] Health checks passing
- [ ] Smoke tests passing
- [ ] Integration tests against staging pass
- [ ] Manual testing completed
- [ ] Performance acceptable (response times)
- [ ] No errors in logs

**Staging Soak Period:**
- [ ] Run in staging for minimum 2 hours
- [ ] Monitor error rates, latency, throughput
- [ ] No issues observed

#### 2. Production Deployment Window

**Deployment Timing:**
- ✅ **Preferred:** Tuesday-Thursday, 9am-3pm EST
- ⚠️ **Avoid:** Monday mornings, Friday afternoons
- ❌ **Never:** Weekends, holidays, Black Friday season

**On-Call Requirement:**
- [ ] Deploying engineer available for 2 hours post-deployment
- [ ] Team lead notified of deployment
- [ ] PagerDuty escalation path confirmed

#### 3. Production Deployment

**Pre-Deployment:**
```bash
# Verify no ongoing incidents
# Check PagerDuty for active alerts

# Notify team
# Post in #deployments Slack channel:
# "🚀 Deploying service-name v1.2.3 to production
#  Changes: [link to release notes]
#  Rollback plan: [describe]
#  Monitoring: [dashboard link]"
```

**Deployment Steps:**
```bash
# 1. Create backup/snapshot
kubectl get deployment service-name -n production -o yaml > backup-deployment.yaml

# 2. Build and tag production image
docker build -t homedepot/service-name:v1.2.3 .
docker push registry.prod.homedepot.com/service-name:v1.2.3

# 3. Update deployment (rolling update)
kubectl set image deployment/service-name \
  service-name=registry.prod.homedepot.com/service-name:v1.2.3 \
  -n production \
  --record

# 4. Watch rollout
kubectl rollout status deployment/service-name -n production

# 5. Verify pods are healthy
kubectl get pods -n production -l app=service-name
```

**Gradual Rollout (Recommended):**
```yaml
# Deploy to canary first (5% of traffic)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: service-name
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: service-name
  progressDeadlineSeconds: 600
  service:
    port: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 5
```

#### 4. Post-Deployment Validation

**Immediate Checks (First 5 Minutes):**

```bash
# Check deployment status
kubectl get deployments -n production
kubectl get pods -n production -l app=service-name

# Check health endpoints
curl https://api.homedepot.com/service-name/actuator/health

# Check logs for errors
kubectl logs -n production -l app=service-name --since=5m | grep -i error
```

**Verify:**
- [ ] All pods running and ready
- [ ] Health checks passing
- [ ] No error spike in logs
- [ ] Response times normal
- [ ] No 5xx errors

**Monitoring Dashboards:**

Check these dashboards for 30 minutes:

1. **Application Dashboard (New Relic):**
   - [ ] Response time < 500ms (p95)
   - [ ] Error rate < 0.1%
   - [ ] Throughput matches expected
   - [ ] Apdex score > 0.95

2. **Infrastructure Dashboard:**
   - [ ] CPU usage < 70%
   - [ ] Memory usage < 80%
   - [ ] No pod restarts
   - [ ] Network traffic normal

3. **Business Metrics:**
   - [ ] Order completion rate unchanged
   - [ ] Checkout success rate > 98%
   - [ ] Add-to-cart events normal
   - [ ] No spike in customer support tickets

**Smoke Tests:**
```bash
# Run automated smoke tests against production
npm run test:smoke -- --env=production

# Test critical paths:
# - User login
# - Product search
# - Add to cart
# - Checkout
```

#### 5. Monitoring Period

**First 30 Minutes:**
- Watch dashboards continuously
- Check logs every 5 minutes
- Run smoke tests

**First 2 Hours:**
- Monitor alerts and PagerDuty
- Check error tracking (Sentry/Rollbar)
- Review customer feedback channels
- Stay available for rollback

**First 24 Hours:**
- Daily metrics review
- Customer support ticket review
- Performance trend analysis

### Rollback Plan

**When to Rollback:**

Rollback immediately if:
- Error rate > 1%
- Response time degrades > 50%
- Critical functionality broken
- Database corruption detected
- Security vulnerability introduced

**Rollback Procedure:**

**Quick Rollback (Kubernetes):**
```bash
# Option 1: Rollback to previous version
kubectl rollout undo deployment/service-name -n production

# Option 2: Rollback to specific version
kubectl rollout undo deployment/service-name --to-revision=2 -n production

# Verify rollback
kubectl rollout status deployment/service-name -n production
```

**Feature Flag Rollback:**
```bash
# Just disable the feature flag in LaunchDarkly
# No code deployment needed!
```

**Database Rollback:**
```bash
# Run rollback migration
flyway migrate -target=<previous-version>

# Restore from backup if needed
pg_restore -d homedepot_prod backup.sql
```

**Post-Rollback:**
- [ ] Verify service restored to normal
- [ ] Check metrics returned to baseline
- [ ] Notify team in Slack
- [ ] Create incident ticket
- [ ] Schedule post-mortem

### Post-Deployment

#### 1. Communication

**Notify Stakeholders:**
- [ ] Post success message in #deployments
- [ ] Update JIRA ticket status
- [ ] Notify product owner
- [ ] Update release tracker

**Success Message Template:**
```
✅ Production Deployment Complete

Service: service-name v1.2.3
Deployed: 2024-01-15 10:30 AM EST
Status: Successful
Monitoring: All metrics normal

Release Notes: [link]
Monitoring Dashboard: [link]

No issues observed. Continuing to monitor.
```

#### 2. Post-Mortem (If Issues)

**If Rollback Required:**

Within 24 hours, document:
- What went wrong
- Why it wasn't caught in testing
- How we detected the issue
- Timeline of events
- Root cause analysis
- Action items to prevent recurrence

#### 3. Metrics Collection

**Track Deployment Metrics:**
- Deployment duration
- Time to detect issues
- Time to rollback
- Downtime (if any)
- Customer impact

## Deployment Checklist Summary

Use this checklist for every production deployment:

```markdown
## Pre-Deployment
- [ ] All tests passing (unit, integration, E2E)
- [ ] Test coverage ≥ 80%
- [ ] Security scans clean (no critical/high vulns)
- [ ] Code reviews approved (minimum 2)
- [ ] Documentation updated (README, API docs, CHANGELOG)
- [ ] Runbook updated
- [ ] Database migrations tested (if applicable)
- [ ] Feature flags configured (if applicable)
- [ ] Staging deployment successful
- [ ] Staging soak period completed (2+ hours)

## Deployment
- [ ] Deployment window appropriate (Tue-Thu, 9am-3pm)
- [ ] On-call engineer available
- [ ] Team notified in Slack
- [ ] Backup/snapshot created
- [ ] Deployed to production
- [ ] Rolling update completed successfully

## Post-Deployment (First 30 min)
- [ ] All pods running and healthy
- [ ] Health checks passing
- [ ] No error spike in logs
- [ ] Response times normal
- [ ] Smoke tests passing
- [ ] Monitoring dashboards green
- [ ] No customer-reported issues

## Extended Monitoring (2 hours)
- [ ] Error rate < 0.1%
- [ ] Response times normal
- [ ] Business metrics unchanged
- [ ] No alerts fired
- [ ] Customer support tickets normal

## Rollback Plan
- [ ] Rollback procedure documented
- [ ] Rollback tested in staging
- [ ] Database rollback plan ready
- [ ] Team knows how to rollback

## Communication
- [ ] Success posted in Slack
- [ ] JIRA ticket updated
- [ ] Stakeholders notified
```

## Automated Deployment Actions

When developer asks to deploy:

1. **Run all checks** (tests, security scans, coverage)
2. **Generate deployment checklist** with current status
3. **Highlight blockers** (failed tests, missing approvals, etc.)
4. **Prepare deployment commands** for staging and production
5. **Create monitoring checklist** with specific dashboard links
6. **Generate rollback commands** for quick access
7. **Draft Slack messages** for notifications

Do not actually deploy without explicit confirmation - just prepare everything.

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
