# Home Depot Claude Code Skills - Quick Start Guide

**Time to First Value: 30 minutes**

This guide gets your pilot running TODAY.

---

## ⚡ 30-Minute Quick Start

### Minute 0-10: Setup

**1. Copy skills to your project:**
```bash
cd your-homedepot-project/
mkdir -p .claude/skills
cp claude-skills-samples/*.md .claude/skills/
```

**2. Commit to Git:**
```bash
git add .claude/skills/
git commit -m "Add Home Depot Claude Code skills for pilot"
git push origin main
```

**3. Team gets skills:**
```bash
# Other developers just need to:
git pull origin main
# Skills are now available!
```

**✅ Done!** Skills are now active for all developers with repo access.

---

### Minute 10-20: Test with Pilot Team

**1. Select 5-10 pilot developers**
- Mix of junior, mid, senior
- Active contributors (daily commits)
- Early adopters (like trying new tools)

**2. Send them this message:**

```
📢 Claude Code Skills Pilot Starting Today!

We've added AI-powered skills to enforce our standards automatically.

What's a skill?
A workflow that guides Claude to follow Home Depot's exact standards for security, architecture, and code quality.

What you need to do:
1. git pull (you now have the skills!)
2. Use Claude Code as normal
3. Skills activate automatically based on what you're doing

Examples:
- Creating payment API? → Security skill auto-applies PCI requirements
- Creating new service? → Architecture skill sets up standard structure
- Making a PR? → Code review skill checks everything

Try it now:
Ask Claude: "Create a payment endpoint for credit card processing"
Watch it automatically add encryption, validation, logging, etc.

Questions? #claude-code-pilot
```

---

### Minute 20-30: First Test

**Run a live test with the team:**

**Test 1: Payment Security**
```
Developer asks: "Create a payment endpoint that accepts credit card data"

Expected: Claude applies payment-api-security skill
- Adds AES-256 encryption
- Validates card with Luhn algorithm
- Adds Splunk audit logging
- Includes security headers
- Implements rate limiting
```

**Test 2: Code Review**
```
Developer asks: "Review my code for our standards"

Expected: Claude applies code-review-checklist skill
- Runs linters
- Checks test coverage
- Scans for security issues
- Validates documentation
- Checks for performance issues
```

**Test 3: New Service**
```
Developer asks: "Create a new microservice for inventory management"

Expected: Claude applies new-microservice-scaffolder skill
- Creates standard folder structure
- Sets up Spring Boot + PostgreSQL
- Adds health checks
- Configures New Relic monitoring
- Generates Dockerfile and K8s manifests
```

**✅ If all 3 tests work → Pilot is successful! Continue to next steps.**

---

## 📊 Week 1: Collect Feedback

### Daily Check-ins

**Quick Slack poll (2 min):**
```
🎯 Daily Claude Code Skills Check-in

1. Did Claude use skills today? 👍/👎
2. Most useful skill? Reply in thread
3. Any issues? Reply in thread

React with 📈 if skills saved you time today!
```

### End of Week Survey

**Send this survey (5 min to complete):**

1. How often did Claude use skills this week?
   - [ ] Multiple times per day
   - [ ] Daily
   - [ ] A few times
   - [ ] Rarely
   - [ ] Never (why not?)

2. Which skill was most valuable?
   - [ ] Payment API Security
   - [ ] Code Review Checklist
   - [ ] New Microservice Scaffolder
   - [ ] Security Code Review
   - [ ] PII Data Handling
   - [ ] Deployment Checklist

3. Did skills help you follow Home Depot standards?
   - [ ] Yes, significantly
   - [ ] Yes, somewhat
   - [ ] No difference
   - [ ] Made it harder (explain)

4. How much time did skills save you? __ hours

5. What skill should we add next?

6. Any other feedback?

---

## 📈 Week 2-4: Measure Impact

### Metrics to Track

**Automated (from existing tools):**

| Metric | Tool | How to Measure |
|--------|------|----------------|
| Time to first PR | GitHub | Days from hire date to first merged PR |
| Review cycles | GitHub | Count of "requested changes" before merge |
| Standards violations | Code reviews | Manual count of linting/style issues |
| Test coverage | SonarQube | % coverage in SonarQube dashboard |
| Security issues | Checkmarx/Snyk | Count of critical/high findings |

**Manual (weekly):**
- Developer satisfaction (1-10 scale)
- Time savings (hours per week)
- Success stories (specific examples)

### Sample Metrics Report

```markdown
# Week 2 Pilot Metrics

## Usage
- 8/10 developers used skills actively
- 45 skill activations this week
- Most used: code-review-checklist (20), payment-api-security (12)

## Impact
- Time to first PR: 11 days (was 14, target 7-9) ↓21%
- Review cycles: 2.0 rounds (was 2.5, target 1.5) ↓20%
- Standards violations: 5 per PR (was 8, target 2-4) ↓38%
- Developer satisfaction: 8.2/10 (was 7.5)

## Feedback
Positive:
- "Saved me 2 hours on PR prep this week"
- "Caught security issue before code review"
- "New service setup was so much faster"

Negative:
- "Skill sometimes too verbose"
- "Want more Java-specific skills"

## Next Steps
- Continue pilot with current team
- Refine verbose skills
- Create Java-specific skill (Spring Boot patterns)
```

---

## 🚀 Month 2: Scale

### When to Scale

Scale to more developers when:
- ✅ Positive feedback from pilot team (>8/10 satisfaction)
- ✅ Measurable impact (20%+ improvement in any metric)
- ✅ No major issues or blockers
- ✅ Skills are stable (not changing daily)

### How to Scale

**Phase 1: Department** (50-100 developers)
```bash
# Add skills to all team repos
for repo in team-repo-1 team-repo-2 team-repo-3; do
  cd $repo
  mkdir -p .claude/skills
  cp /path/to/skills/*.md .claude/skills/
  git add .claude/skills/
  git commit -m "Add Claude Code skills"
  git push
done
```

**Phase 2: Engineering Org** (200+ developers)
- Create central skills repository
- Teams import via Git submodule or copy
- Establish skills maintenance team
- Set up quarterly review process

---

## 🛠️ Customization Guide

### Quick Customizations (5 min each)

**1. Update Splunk Configuration**

Edit `payment-api-security.md`:
```markdown
# Find this section
**Log to Splunk:**
- Use `AuditLogger.logPaymentEvent()` method
- Tag with `source=payment_audit`
- Include custom index: `index=pci_compliance`

# Update with your actual config
**Log to Splunk:**
- Use `AuditLogger.logPaymentEvent()` method
- Tag with `source=payment_audit`
- Include custom index: `index=homedepot_pci`
- Add sourcetype: `sourcetype=payment_transaction`
```

**2. Update Service URLs**

Edit `new-microservice-scaffolder.md`:
```markdown
# Find observability section
**Observability:**
- New Relic APM for monitoring
- Splunk for centralized logging

# Add your specific endpoints
**Observability:**
- New Relic APM: https://newrelic.homedepot.com
- Splunk: https://splunk.homedepot.com
- Grafana: https://grafana.homedepot.com
```

**3. Add Team-Specific Patterns**

Create new skill `homedepot-api-standards.md`:
```markdown
# Home Depot API Standards

## Description
Apply Home Depot-specific API conventions.
Use when creating any REST API endpoint.

## Instructions

All Home Depot APIs must:

1. **URL Structure**
   - Format: `/api/v{version}/{resource}`
   - Example: `/api/v1/products`, `/api/v2/orders`

2. **Authentication**
   - Use Home Depot OAuth2 Gateway
   - Include `X-HD-Client-ID` header
   - Example: [code snippet]

3. **Error Codes**
   - Use Home Depot standard error codes
   - Format: `HD-{DOMAIN}-{CODE}`
   - Example: `HD-PAYMENT-001`

[etc.]
```

---

## 💡 Success Stories Template

Document wins to build momentum:

```markdown
# Success Story: Payment API Security

**Developer:** Jane Doe (Junior Developer, E-Commerce Team)

**Challenge:**
Creating first payment endpoint. Unfamiliar with PCI requirements.
Estimated: 2 days of work + multiple review cycles.

**With Skills:**
- Asked Claude: "Create payment endpoint for checkout"
- payment-api-security skill activated automatically
- Generated code included:
  - AES-256 encryption
  - Luhn card validation
  - Splunk audit logging
  - Rate limiting
  - Security headers
- First code review: PASSED
- Shipped: Same day

**Impact:**
- Time saved: 1.5 days
- Review cycles: 1 (vs typical 2-3)
- PCI violations: 0
- Developer confidence: High

**Quote:**
"I couldn't believe how much it handled automatically.
I learned our security standards while building, instead
of reading docs for hours. Game changer."
```

---

## 🎯 Common Questions

### Q: Do developers need to do anything different?

**A:** No! They use Claude Code exactly as before. Skills activate automatically based on context.

---

### Q: What if a skill gives wrong guidance?

**A:**
1. Fix the skill (it's just a markdown file)
2. Commit and push
3. Team gets update via `git pull`
4. Post in #claude-code-pilot

---

### Q: How do I know which skills are available?

**A:**
```bash
ls .claude/skills/
# Or ask Claude: "What skills are available?"
```

---

### Q: Can we customize skills for our team?

**A:** Yes! Skills are markdown files. Edit them like any code. See customization guide above.

---

### Q: Do skills work offline?

**A:** Skills are local files, but Claude Code requires internet connection for the AI.

---

### Q: What if Claude doesn't use a skill when expected?

**A:** Check the skill's "Description" section. Make it more specific about when to activate.

Example:
```markdown
# ❌ Too vague
"Security checks for APIs"

# ✅ Specific
"Apply PCI DSS compliance to payment APIs. Use when creating
endpoints that handle credit cards, transactions, or payment data."
```

---

## 📞 Get Help

### Slack Channel
**#claude-code-pilot**
- Quick questions
- Daily feedback
- Success stories
- Troubleshooting

### Office Hours
**Every Thursday, 2-3pm EST**
- Zoom link: [in channel topic]
- Topics: Skills, best practices, Q&A

### Documentation
- **Full Admin Guide:** `Claude_Code_Skills_Admin_Guide.html`
- **Skills README:** `claude-skills-samples/README.md`
- **Anthropic Docs:** https://docs.claude.com/en/docs/claude-code/skills

### Contacts
- **Platform Engineering:** platform-engineering@homedepot.com
- **Solutions Architect:** [Your Anthropic contact]

---

## ✅ Success Checklist

After 4 weeks, you should have:

- [ ] 5-10 developers actively using skills
- [ ] 3-5 skills in production use
- [ ] Measurable metrics showing improvement (20%+ in any area)
- [ ] Positive feedback (8+/10 satisfaction)
- [ ] 3-5 documented success stories
- [ ] Plan for scaling to broader team
- [ ] Skills maintenance process established

**If you have all these ✅ → Ready to scale!**

---

## 🚀 Ready to Start?

1. **Right now (5 min):** Copy skills to a repo
2. **Today (30 min):** Send pilot team announcement
3. **This week:** Collect daily feedback
4. **Next week:** Review metrics and iterate
5. **Month 2:** Scale to broader team

**Let's go! 🎯**

Questions? #claude-code-pilot
