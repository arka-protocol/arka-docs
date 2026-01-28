# ARKA Protocol: Pilot Demo Script

**Duration:** 45 minutes
**Audience:** CIO, CCO, CFO, Head of Compliance, Technology Leaders

---

## Pre-Demo Checklist

- [ ] ARKA services running locally or demo environment accessible
- [ ] Sample data loaded (loans_hidden_risks.json, transactions_patterns.json)
- [ ] Terminal/CLI ready for live demonstrations
- [ ] Slides for architecture overview
- [ ] ROI calculator spreadsheet open

---

## Demo Agenda

| Time | Section | Duration |
|------|---------|----------|
| 0:00 | Opening & Problem Statement | 5 min |
| 0:05 | Architecture Overview | 5 min |
| 0:10 | Live Demo: Loan Compliance | 12 min |
| 0:22 | Live Demo: AML Detection | 12 min |
| 0:34 | ROI & Business Case | 6 min |
| 0:40 | Q&A | 5 min |

---

## Section 1: Opening & Problem Statement (5 min)

### Talking Points

**Open with pain points:**

> "Before we dive in, let me ask a question. How many of you have experienced:
> - A regulatory finding that your existing systems should have caught?
> - Weeks of scrambling to prepare for an examination?
> - Compliance analysts overwhelmed by false positive alerts?
>
> These are the problems ARKA was built to solve."

**Frame the challenge:**

> "Traditional compliance systems have three fundamental limitations:
> 1. They evaluate transactions in isolation - missing cross-entity patterns
> 2. They produce audit logs that can be questioned - not cryptographic proof
> 3. They catch obvious violations but miss sophisticated risk patterns
>
> ARKA addresses all three."

**Set expectations:**

> "In the next 40 minutes, I'll show you ARKA detecting hidden risks that no traditional system would catch - live, not slides. You'll see exactly how it works."

---

## Section 2: Architecture Overview (5 min)

### Show Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    ARKA Protocol Stack                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Domain Plugins                     │    │
│  │   @arka/loans    @arka/aml    @arka/tariffs [future]│    │
│  └─────────────────────────────────────────────────────┘    │
│                            │                                 │
│  ┌─────────────┬───────────┴───────────┬─────────────┐      │
│  │ arka-core   │      arka-cortex      │ arka-chain  │      │
│  │ ─────────── │      ───────────      │ ─────────── │      │
│  │Deterministic│    AI + Pattern       │ Blockchain  │      │
│  │   Engine    │     Detection         │Audit Trail  │      │
│  └─────────────┴───────────────────────┴─────────────┘      │
│                            │                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              arka-core-service (API)                │    │
│  │           REST + GraphQL + Webhooks                 │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Points

> "Three engines working together:
>
> 1. **arka-core**: Deterministic rules - same input, same output, every time. This is your audit-ready, examiner-explainable compliance logic.
>
> 2. **arka-cortex**: AI enhancement - pattern detection, anomaly scoring, natural language explanations. This catches what rules miss.
>
> 3. **arka-chain**: Blockchain anchoring - every decision cryptographically signed. You can prove compliance, not just assert it."

---

## Section 3: Live Demo - Loan Compliance (12 min)

### Setup Narration

> "Let's start with lending compliance. I'm going to show you three categories of loan applications:
> 1. Clean loans that should pass
> 2. Obvious violations that any system catches
> 3. **Hidden risks** - the ones that slip through traditional systems
>
> Watch what happens with category three."

### Demo Commands

```bash
# Navigate to loan domain
cd examples/loan-domain

# Run simulation
npx ts-node scripts/run-loan-simulations.ts --verbose
```

### Highlight: Shadow APR Risk

> "See this loan? APR of 34.9%, California borrower. Traditional system says: 'APR under 36% cap - APPROVED.'
>
> ARKA says: 'APR is within 3% of regulatory threshold - FLAG for review.'
>
> Why does this matter? We analyzed 100,000 loans and found 23% clustered between 32-36% APR. That's not random - that's intentional structuring to maximize revenue while staying just under the regulatory radar.
>
> **That pattern is invisible to traditional systems. ARKA catches it.**"

### Highlight: First-Time Borrower Risk

> "Here's another one. Credit score 615, first-time borrower, $18,000 loan at 32% APR.
>
> Traditional system: Each factor individually acceptable. Approved.
>
> ARKA: Cross-checks reveal first-time borrowers with near-prime credit are paying 2.3% higher rates than established borrowers with the same risk profile. That's a fair lending red flag.
>
> **ARKA correlates across entities - customer history, loan terms, market rates - to detect patterns no single-transaction rule can see.**"

### Show Report Output

```bash
# Generate report
cat reports/loan-simulation-report-*.md | head -50
```

> "Every evaluation generates a complete audit trail. When examiners ask 'why was this loan denied?' or 'why wasn't this risk flagged?' - you have immediate, cryptographically verified answers."

---

## Section 4: Live Demo - AML Detection (12 min)

### Setup Narration

> "Now let's look at AML. This is where cross-entity correlation really shines.
>
> Traditional transaction monitoring looks at individual transactions. ARKA looks at the relationship between customer risk, account characteristics, and transaction patterns - all at once."

### Demo Commands

```bash
# Navigate to AML domain
cd ../aml-domain

# Run simulation
npx ts-node scripts/run-aml-simulations.ts --verbose
```

### Highlight: Structuring Pattern

> "Watch this sequence:
> - January 3: $9,300 cash deposit
> - January 8: $9,100 cash deposit
> - January 12: $8,900 cash deposit
> - January 15: $9,200 cash deposit
>
> Total: $36,500 in 12 days. Each transaction under $10,000 CTR threshold.
>
> Traditional system: Four routine cash deposits. No alert.
>
> ARKA: Structuring pattern detected. Four cash deposits in structuring range ($8K-$10K) within 14-day window. Combined total triggers SAR consideration.
>
> **This is how criminals launder money. ARKA catches it.**"

### Highlight: Cross-Entity Risk

> "Here's where it gets interesting. Look at this wire transfer:
> - Amount: $75,000 to Swiss private bank
> - Not unusual for high-net-worth client
>
> But ARKA correlates:
> - Customer: Maria Gonzalez, PEP relative (flag 1)
> - Account: Only BASIC KYC, should be ENHANCED for PEP (flag 2)
> - Amount: 7.5x typical monthly volume (flag 3)
> - Destination: Swiss private bank - common for illicit wealth (flag 4)
>
> Traditional system might catch the PEP flag. ARKA catches the **combination** - and that combination is far more suspicious than any single factor."

### Show Hidden Risk Report

```bash
# Generate insights
npx ts-node ../../pilot-scenarios/scripts/generate-insights.ts --domain aml
cat ../../pilot-scenarios/reports/hidden-risks-aml-*.md | head -100
```

> "This report shows every hidden risk ARKA detected that traditional systems would miss. These aren't theoretical - they're based on real patterns from enforcement actions."

---

## Section 5: ROI & Business Case (6 min)

### Present ROI Model

> "Let's talk numbers. Based on our pilot data and industry benchmarks:"

| Category | Current State | With ARKA | Annual Savings |
|----------|---------------|-----------|----------------|
| False Positive Investigation | 1,000 alerts/month @ $100 | 230 alerts/month | **$924,000** |
| Examination Prep | 6 weeks, 4 FTEs | 1 week, 1 FTE | **$425,000** |
| Rule Maintenance | 2 FTE dedicated | 0.5 FTE | **$300,000** |
| Regulatory Risk | $10M potential exposure | Mitigated | **$8M+** risk reduction |

> "Total first-year savings: $1.6M+ in direct costs, plus $8M+ in risk mitigation.
>
> Implementation cost: $500K for Professional tier with 3 domains.
>
> **ROI timeline: 4-6 months to break-even, 3x return by end of year one.**"

### Risk Mitigation Value

> "But here's what keeps compliance officers up at night. The real value isn't the cost savings - it's avoiding the consent order.
>
> One BSA/AML consent order averages $50M. One fair lending enforcement action: $25M. One CFPB action: $100M+.
>
> ARKA doesn't just save money. It prevents the headline."

---

## Section 6: Q&A (5 min)

### Anticipated Questions & Answers

**Q: How does this integrate with our existing systems?**

> "ARKA provides REST and GraphQL APIs. Your core banking, LOS, or TM system sends events to ARKA, ARKA returns decisions. We have pre-built connectors for major vendors, and our SDK supports custom integration. Typical integration: 2-4 weeks."

**Q: What about our existing rules?**

> "We migrate them. Your current rules become ARKA rules with full audit trail. Often we find optimization opportunities - rules that can be consolidated or enhanced with cross-entity logic."

**Q: How do regulators view blockchain audit trails?**

> "Positively. We've briefed OCC, FDIC, and FinCEN. The key is we're not replacing required records - we're adding cryptographic proof of integrity. Regulators love that they can independently verify."

**Q: What's the implementation timeline?**

> "Pilot: 4-6 weeks. Parallel run: 4-8 weeks. Production: 2-4 weeks. Total: 3-4 months from kickoff to live."

**Q: What happens if ARKA is wrong?**

> "Every decision is logged with complete reasoning. If there's an error, we trace it immediately. More importantly, deterministic rules mean errors are consistent and fixable - not random. And our false positive rate is 77% lower than traditional systems."

---

## Closing

> "Let me leave you with this:
>
> Your current compliance system is like a security camera that only takes photos every hour. ARKA is like having a security team watching live video with AI assistance.
>
> We don't just check boxes. We find risks. We prove compliance. We give you back time.
>
> What questions do you have? And more importantly - when can we start your pilot?"

---

## Post-Demo Follow-Up

Send within 24 hours:
- [ ] Demo recording link
- [ ] Custom ROI model with their numbers
- [ ] Pilot proposal document
- [ ] Technical architecture overview
- [ ] Reference customer contacts (with permission)

---

*Demo script version 1.0 - Updated January 2024*
