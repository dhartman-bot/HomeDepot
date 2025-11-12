# Home Depot Claude Code Skills - Sample Collection

This directory contains production-ready skills customized for Home Depot's engineering standards and workflows.

## 📚 Available Skills

### 🔒 Security & Compliance

#### `payment-api-security.md`
**Priority:** High | **Category:** PCI Compliance

Enforces PCI DSS requirements for all payment-related APIs. Automatically adds:
- AES-256 encryption for payment data
- Card number validation (Luhn algorithm)
- Audit logging to Splunk with PCI tags
- Input sanitization and rate limiting
- Security headers and error handling

**Use when:** Creating or modifying payment endpoints

---

#### `security-code-review.md`
**Priority:** High | **Category:** OWASP / Security

Comprehensive security review covering OWASP Top 10:
- SQL injection, XSS, XXE vulnerabilities
- Authentication and authorization issues
- Sensitive data exposure
- Hardcoded credentials scanning
- Dependency vulnerability checks

**Use when:** Before merging to main, deploying to production, or handling sensitive data

---

#### `pii-data-handling.md`
**Priority:** High | **Category:** Privacy Compliance

Ensures CCPA/GDPR compliance for customer PII:
- Identifies PII fields in data models
- Enforces encryption requirements
- Implements data retention policies
- Masks PII in logs and error messages
- Access logging for audit trails

**Use when:** Working with customer data (name, email, address, SSN)

---

### 🏗️ Architecture & Development

#### `new-microservice-scaffolder.md`
**Priority:** High | **Category:** Architecture

Creates production-ready microservices with Home Depot standards:
- Standard folder structure and configurations
- Spring Boot + PostgreSQL + Flyway setup
- Health checks for Kubernetes
- New Relic APM and Splunk logging
- Dockerfile, K8s manifests, CI/CD pipeline

**Use when:** Starting a new microservice or backend API

---

### ✅ Quality & Testing

#### `code-review-checklist.md`
**Priority:** High | **Category:** Code Quality

Automated code review following Home Depot standards:
- Linting and formatting checks
- Test coverage validation (≥80%)
- Security scanning
- Performance analysis (N+1 queries, etc.)
- Documentation completeness

**Use when:** Creating pull requests or before requesting code review

---

### 🚀 Operations & Deployment

#### `deployment-checklist.md`
**Priority:** Medium | **Category:** Operations

Guides safe production deployments:
- Pre-deployment validation (tests, security, approvals)
- Staging deployment and soak period
- Production deployment with monitoring
- Rollback procedures
- Post-deployment verification

**Use when:** Deploying to staging or production

---

## 🚀 Quick Start

### 1. Copy Skills to Your Project

```bash
# Navigate to your project
cd your-homedepot-project/

# Create skills directory
mkdir -p .claude/skills

# Copy all skills
cp /path/to/claude-skills-samples/*.md .claude/skills/

# Commit to Git
git add .claude/skills/
git commit -m "Add Home Depot Claude Code skills"
git push
```

### 2. Team Gets Skills Automatically

All developers with repo access get skills via normal `git pull`:

```bash
git pull origin main
# Skills now available in Claude Code!
```

### 3. Skills Activate Automatically

Claude detects when to use each skill based on context:

```
Developer: "Create a payment endpoint for Apple Pay"
Claude: [Automatically applies payment-api-security.md skill]
        [Generates code with encryption, validation, logging, etc.]
```

---

## 📖 How to Use Skills

### Automatic Activation (Recommended)

Skills activate automatically based on what you're doing:

- **Creating payment API?** → `payment-api-security.md` activates
- **Creating new service?** → `new-microservice-scaffolder.md` activates
- **Ready to deploy?** → `deployment-checklist.md` activates
- **Creating PR?** → `code-review-checklist.md` activates

Just work normally - Claude knows when to use each skill.

### Manual Activation

You can also explicitly request a skill:

```
"Run the security code review skill on this file"
"Use the deployment checklist skill to verify I'm ready"
"Apply the payment security skill to this endpoint"
```

---

## 🎯 Recommended Pilot Approach

### Phase 1: Core Skills (Week 1-2)
Start with these 5 highest-impact skills:

1. ✅ `payment-api-security.md` - Prevents PCI violations
2. ✅ `code-review-checklist.md` - Reduces review cycles
3. ✅ `new-microservice-scaffolder.md` - Ensures consistency
4. ✅ `security-code-review.md` - Catches vulnerabilities early
5. ✅ `pii-data-handling.md` - Maintains compliance

**Expected Impact:** 30-50% reduction in review cycles, zero PCI violations

### Phase 2: Expansion (Week 3-4)
Add operational skills:

6. ✅ `deployment-checklist.md` - Safer deployments

**Expected Impact:** Fewer production incidents, faster deployments

### Phase 3: Customization (Month 2+)
Create Home Depot-specific skills:

- Oracle Retail integration patterns
- SAP connector standards
- Store operations workflows
- Supply chain API conventions

---

## ✏️ Customizing Skills

Skills are just markdown files - easy to customize for your team!

### 1. Update Tool/Service Names

Example: Update Splunk configuration
```markdown
# Before (generic)
Log to your logging service

# After (Home Depot-specific)
Log to Splunk with index=pci_compliance and source=payment_audit
```

### 2. Add Team-Specific Patterns

Example: Add Home Depot authentication
```markdown
## Authentication
All endpoints must use Home Depot's OAuth2 Gateway:

\`\`\`java
@Configuration
@EnableOAuth2Client
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    // Home Depot-specific config
}
\`\`\`
```

### 3. Update Validation Rules

Example: Customize business rules
```markdown
## Validation
- Maximum order value: $50,000 (configurable via homedepot.order.max.amount)
- Valid store IDs: Must match pattern HD[0-9]{4}
- SKU format: [Department][Class][Item] (3-3-6 digits)
```

### 4. Test Changes

```bash
# Make changes to skill
vim .claude/skills/payment-api-security.md

# Test with Claude
# Ask Claude to create payment endpoint
# Verify it follows your updated rules

# Commit when ready
git add .claude/skills/payment-api-security.md
git commit -m "Update payment security skill with HD OAuth"
git push
```

---

## 📊 Measuring Impact

Track these metrics to demonstrate skill value:

### Before/After Metrics

| Metric | Before Skills | Target With Skills | Measurement |
|--------|--------------|-------------------|-------------|
| Time to First PR | 14 days | 7-9 days | Git/GitHub |
| Review Cycles | 2.5 rounds | 1.5 rounds | GitHub PR data |
| Standards Violations | 8 per PR | 2-4 per PR | Code review notes |
| Security Issues | 5/quarter | 1-2/quarter | Security scans |
| Test Coverage | 65% | 80%+ | SonarQube |
| PCI Violations | 2/quarter | 0/quarter | Audit reports |

### Qualitative Feedback

**Weekly Developer Survey (2 min):**
1. Did Claude Code skills help you this week? (Yes/No)
2. Which skill was most valuable?
3. What skill would you like to see added?
4. How much time did skills save you? (hours)

---

## 🐛 Troubleshooting

### "Claude isn't using my skill"

**Check:**
- Skill file in `.claude/skills/` directory?
- Description section specific about when to activate?
- Skill name descriptive (not `skill1.md`)?

**Fix:**
```markdown
# ❌ Vague description
"Security checks for APIs"

# ✅ Specific description
"Apply PCI DSS compliance requirements to all payment-related APIs.
Use when creating or modifying endpoints that handle credit cards,
customer payment data, or financial transactions."
```

### "Skill is giving outdated guidance"

**Fix:**
1. Update the skill file
2. Commit and push changes
3. Team members get update via `git pull`
4. Notify team in Slack

### "How do I know which skills are available?"

**Option 1:** Check `.claude/skills/` directory
```bash
ls -la .claude/skills/
```

**Option 2:** Ask Claude
```
"What skills are available?"
"List all Claude Code skills in this project"
```

**Option 3:** Read this README!

---

## 📞 Support

### Internal Home Depot Support
- **Slack:** #claude-code-pilot
- **Email:** platform-engineering@homedepot.com
- **Wiki:** [Internal Confluence page]

### Anthropic Support
- **Documentation:** https://docs.claude.com/en/docs/claude-code/skills
- **Solutions Architect:** [Your assigned contact]
- **Email:** support@anthropic.com

### Office Hours
- **When:** Every Thursday, 2-3pm EST
- **Where:** Zoom link in #claude-code-pilot
- **Topics:** Skill creation, troubleshooting, best practices

---

## 🎓 Training Materials

### 30-Minute Lunch & Learn
See main admin guide for complete training agenda

**Key Topics:**
1. What are skills? (5 min)
2. Live demo (10 min)
3. How to use skills (10 min)
4. Q&A and feedback (5 min)

### Documentation
- Main Admin Guide: `Claude_Code_Skills_Admin_Guide.html`
- Skill Creation Guide: https://docs.claude.com/en/docs/claude-code/skills
- Best Practices: [Internal wiki]

---

## 🚀 Creating New Skills

### When to Create a New Skill

Good candidates for skills:
- ✅ Repeated patterns across multiple developers
- ✅ Complex workflows with many steps
- ✅ Compliance requirements (security, privacy, etc.)
- ✅ Architecture patterns you want enforced
- ✅ Integration patterns with internal systems

Not good for skills:
- ❌ One-off tasks
- ❌ Simple patterns (better as code snippets)
- ❌ Rapidly changing requirements

### Skill Template

```markdown
# Skill Name

## Description
[When should this skill activate? Be specific!]

Use when: [list specific triggers]
Activate for phrases like: [examples]

## Instructions

[Step-by-step instructions for Claude]

1. **First Step**
   - Specific action
   - Code example if applicable

2. **Second Step**
   - Specific action
   - What to check for

[etc.]

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
[Or specify specific tools if restricted]
```

### Getting Help Creating Skills

1. Start with existing skills as templates
2. Use Claude's `skill-creator` skill (built-in helper)
3. Ask in #claude-code-pilot Slack channel
4. Schedule time with platform engineering team

---

## 📝 Skill Maintenance

### Version Control Best Practices

```bash
# Descriptive commit messages
git commit -m "Update payment-api-security: Add Apple Pay support"

# Tag major skill updates
git tag -a skills-v1.1 -m "Add deployment checklist skill"
```

### Review Schedule

- **Weekly:** Check pilot feedback for needed updates
- **Monthly:** Review skill usage metrics
- **Quarterly:** Comprehensive skill audit and updates

### Deprecating Skills

When a skill is no longer needed:

1. Add deprecation notice to skill description
2. Suggest alternative approach
3. Keep skill for 1 sprint (2 weeks)
4. Remove and document in CHANGELOG

---

## 💡 Success Stories

### Example: Payment API Security Skill

**Before Skill:**
- Junior dev creates payment endpoint
- Forgets encryption and audit logging
- Security review catches issues after 3 days
- 2 rounds of fixes, 5 days total delay
- Risk of PCI violation in codebase

**After Skill:**
- Dev asks Claude to create payment endpoint
- Skill automatically applies all PCI requirements
- First code review passes
- Ships same day
- Zero security issues

**Impact:** 4 days saved, 100% PCI compliance

---

## 🎯 Next Steps

1. **Review skills** - Read through each skill to understand coverage
2. **Copy to project** - Add skills to your repo's `.claude/skills/` directory
3. **Test with pilot team** - 10-20 developers for 2 weeks
4. **Collect feedback** - Weekly surveys and Slack discussions
5. **Iterate** - Refine based on usage patterns
6. **Scale** - Roll out to broader team

---

**Questions?** Ask in #claude-code-pilot or contact platform-engineering@homedepot.com

**Ready to get started?** Copy these skills to your project and watch productivity soar! 🚀
