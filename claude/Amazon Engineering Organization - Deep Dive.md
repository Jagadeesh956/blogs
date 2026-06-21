# 🏢 How Amazon Works — Engineering Organization Deep Dive

> A ground-up exploration of Amazon's internal structure, how it operates, how engineering teams are organized, the full career ladder, and how AWS is built internally. Based on public SEC filings, Amazon's own blog, and verified research (2025–2026).

---

## 🌐 Amazon at a Glance (2026)

| Fact | Detail |
|------|--------|
| **CEO** | Andy Jassy (since 2021) |
| **Employees** | 1.5M+ worldwide (one of the largest private employers) |
| **Annual Revenue** | ~$620B (2024) |
| **AWS Revenue Run Rate** | $100B+ (crossed milestone in 2024) |
| **Structure Type** | Functional-Divisional Hybrid with AI Matrix Overlay |
| **HQ** | Seattle, WA (+ Arlington, VA as second HQ) |
| **Stock** | NASDAQ: AMZN |

Amazon operates as **multiple semi-autonomous businesses under one holding company** — not a single monolithic company. Each major division (AWS, Retail, Advertising, Devices, Entertainment) has its own CEO or SVP and runs largely independently.

---

## 🏗️ The Top-Level Structure

### The S-Team (Senior Leadership Team)

At the very top sits the S-Team — a small group of ~28 senior executives reporting directly to CEO Andy Jassy. The S-Team is responsible for disseminating ideas, solving problems, setting high-level goals, and shaping company culture.

**Current S-Team members (engineering-relevant):**

| Name | Role | Engineering Relevance |
|------|------|-----------------------|
| **Matt Garman** | CEO, Amazon Web Services | Entire AWS engineering org |
| **Peter DeSantis** | SVP, Foundational AI, Custom Silicon & Quantum | AI models, Graviton, Trainium, Quantum |
| **Panos Panay** | SVP, Devices & Services | Alexa, Echo, Kindle, Ring, Fire TV |
| **Paul Kotas** | SVP, Advertising, IMDb, Grand Challenge | Ad tech engineering |
| **Dave Treadwell** | SVP, eCommerce Foundation | Retail engineering platform |
| **David Brown** | SVP, AWS Compute & ML Services | EC2, SageMaker, Bedrock |
| **Colleen Aubrey** | SVP, AWS Applied AI Solutions | Amazon Connect, AWS Supply Chain |
| **James Hamilton** | SVP and Distinguished Engineer | Technical architecture for all of Amazon |
| **Neil Lindsay** | SVP, Amazon Health Services | One Medical, pharmacy tech |
| **Christine Beauchamp** | SVP, North America Stores | Consumer retail tech |

---

## 🗺️ Amazon's Major Divisions — Engineering Focus

```
Andy Jassy (CEO, Amazon)
│
├── Matt Garman → AWS (Cloud + AI Services)
│     ├── AWS Compute & ML (David Brown)
│     │     ├── EC2, ECS, EKS, Lambda, Fargate
│     │     ├── SageMaker (ML platform)
│     │     └── Amazon Bedrock (Foundation Models)
│     ├── AWS Applied AI Solutions (Colleen Aubrey)
│     │     ├── Amazon Connect (contact center AI)
│     │     ├── AWS Supply Chain
│     │     └── Autonomous retail tech
│     └── AWS Sales, Marketing & Global Services
│
├── Peter DeSantis → AI, Silicon & Quantum (NEW 2026)
│     ├── Amazon Nova (frontier AI models)
│     ├── Annapurna Labs (Graviton, Trainium, Nitro chips)
│     ├── Quantum Computing research
│     └── AGI research (Pieter Abbeel leading frontier models)
│
├── Doug Herrington → Worldwide Amazon Stores (Retail)
│     ├── eCommerce Foundation (Dave Treadwell)
│     ├── Amazon Fresh / Grocery tech
│     ├── Supply chain & logistics engineering
│     └── Amazon.com platform engineering
│
├── Panos Panay → Devices & Services
│     ├── Lab126 (hardware R&D — Kindle, Echo, Fire TV)
│     ├── Ring (security cameras)
│     ├── Blink (smart home)
│     ├── Alexa AI (voice/LLM)
│     └── Device Software & Wireless Technologies
│
├── Paul Kotas → Advertising & IMDb
│     ├── Sponsored Products / Display Ads engineering
│     ├── Amazon DSP (programmatic ad platform)
│     └── IMDb engineering
│
├── Mike Hopkins → Prime Video & Studios
│     ├── Video streaming engineering
│     ├── Content technology
│     └── MGM integration
│
└── Neil Lindsay → Amazon Health
      ├── One Medical tech platform
      ├── Amazon Pharmacy
      └── Health AI/data
```

---

## 🔵 AWS Deep Dive — The Engineering Engine

AWS is Amazon's most profitable division and the world's largest cloud provider with ~31% market share. It's where the majority of Amazon's software engineering talent works.

### AWS 2026 Restructure

In late 2025, Andy Jassy announced a significant restructure: Peter DeSantis was placed in charge of a new organization covering Amazon's most expansive AI models (Nova), silicon development (Graviton, Trainium, Nitro), and quantum computing. This split AWS into two major engineering pillars:

**Pillar 1: AWS Utility Computing** (remained under AWS)
- EC2, ECS, EKS, Lambda
- S3, EBS, EFS (storage)
- RDS, DynamoDB, Aurora (databases)
- VPC, CloudFront, Route 53 (networking)
- CloudWatch, CloudTrail (observability)

**Pillar 2: AI, Silicon & Quantum** (new org under Peter DeSantis)
- Amazon Nova foundation models
- Amazon Bedrock (model serving platform)
- Graviton (ARM-based CPU), Trainium (AI training chip), Inferentia (inference chip), Nitro (hypervisor)
- Quantum computing research
- Near-term focus: EC2 UltraClusters 3.0, designed to connect up to 1 million Trainium chips in a single massive cluster for training frontier models.

### AWS Engineering Teams by Service Category

| Category | Services | Engineering Focus |
|----------|----------|-------------------|
| **Compute** | EC2, Lambda, ECS, EKS, Fargate | Hypervisor, containers, serverless runtime |
| **Storage** | S3, EBS, EFS, Glacier | Distributed storage, durability, performance |
| **Databases** | DynamoDB, RDS, Aurora, Redshift, ElastiCache | Distributed DB engines, query optimization |
| **Networking** | VPC, CloudFront, Route 53, Direct Connect | SDN, CDN, DNS at global scale |
| **AI/ML** | SageMaker, Bedrock, Rekognition, Comprehend | ML platforms, model training/serving |
| **Security** | IAM, KMS, GuardDuty, Security Hub | Identity, encryption, threat detection |
| **DevTools** | CodeBuild, CodePipeline, Cloud9, CodeWhisperer | CI/CD, IDE, AI coding assistant |
| **Observability** | CloudWatch, X-Ray, CloudTrail | Monitoring, tracing, audit logging |
| **Custom Silicon** | Graviton, Trainium, Inferentia, Nitro | CPU/GPU/NPU chip design (Annapurna Labs) |

---

## 🧱 How Amazon Engineers Teams — The Operating Model

This is what makes Amazon genuinely different from other big tech companies internally.

### The Two-Pizza Team

Jeff Bezos famously said "If a team can't be fed with two pizzas, it's too large." Teams are typically 5–8 people, each with a clear mission, full accountability, and the freedom to move fast without bureaucratic layers.

But the pizza is the metaphor — the real innovation is what it enables:

Each team has a Single-Threaded Owner — someone whose only job is to make that team successful. Teams are structured around services and APIs, not departments, enabling them to work independently yet integrate easily.

### Single-Threaded Ownership (STO)

This is Amazon's most important organizational concept. Every service, product, or initiative has **exactly one person who owns it fully** — with no other job, no split responsibilities.

> *"The best way to fail at inventing something is by making it somebody's part-time job."* — Amazon internal principle

**How it works in practice:**
- The S3 team owns S3 end-to-end: API design, reliability, security, billing, customer support
- They don't ask another team for permission to change their API
- They are accountable for their own operational metrics (uptime, latency, cost)
- They communicate with other teams only via APIs (Bezos's famous API mandate)

### The API Mandate (Bezos's 2002 Memo)

Bezos issued a famous internal mandate: (1) All teams will expose their data and functionality through service interfaces. (2) Teams must communicate with each other through these interfaces. (3) There will be no other form of interprocess communication allowed.

This is why AWS exists. Amazon built all its internal systems as services, then realized they could sell those services externally. The internal API-first discipline became the product.

### Working Backwards — PR/FAQ

Amazon's product development starts with the PR/FAQ document (Press Release/Frequently Asked Questions), the product of the "Working Backwards" approach. There are also operational business reviews on weekly, monthly, and quarterly cadences, and narratives for operational readiness reviews before any new service launches.

**How a new feature/product gets built at Amazon:**
1. Engineer or PM writes a **Press Release** as if the product already launched (customer-facing language)
2. Follows with **FAQs** — both external (customer questions) and internal (technical/operational questions)
3. Document is reviewed and debated before a single line of code is written
4. If approved, the team works backward from the PR to build what would make it true

### Six-Pager Narratives (No PowerPoint)

Amazon famously banned PowerPoint for decision meetings. Instead:
- Teams write **6-page narrative memos** before every decision meeting
- Meeting starts with **silent reading** of the memo (20–30 min)
- Discussion only begins after everyone has read it
- Forces deep thinking and clear writing

---

## 👤 Amazon Engineering Career Ladder — Full Breakdown

Amazon uses a numbered level system (L4–L10+) for all employees. Here's the complete engineering track:

### Individual Contributor (IC) Track — Software Development Engineers

| Level | Title | Years Exp | What You Do | Total Comp (USD) |
|-------|-------|-----------|-------------|-----------------|
| **L4** | SDE I | 0–2 yrs | Entry-level; new grad. Learn Amazon systems, work on well-defined tasks, contribute to team deliverables. Mostly implementation, limited design scope. | $150K–$220K |
| **L5** | SDE II | 2–5 yrs | Own features end-to-end. Independently design components, write technical specs, mentor L4s. Most common level — where most engineers stay 2–4 years. | $250K–$350K |
| **L6** | Senior SDE | 5–8+ yrs | Own entire systems or sub-systems. Set technical direction for your team. Drive cross-team decisions. Significant jump from L5 in scope and influence. Most engineers plateau here. | $300K–$500K+ |
| **L7** | Principal SDE | 10+ yrs | Multi-team or org-wide technical influence. Define architecture for entire services. Drive Amazon-wide technical standards. Very few engineers reach this level. | $550K–$750K |
| **L8** | Senior Principal SDE | 15+ yrs | Division-wide technical vision. Work directly with VPs and SVPs on strategic direction. Some L8s are essentially VP-equivalent in technical influence. | $750K–$1M+ |
| **L10** | Distinguished Engineer / VP | Exceptional | Highest engineering title. Distinguished Software Development Engineers also serve as Vice Presidents at Amazon. Company-wide technical leadership. Industry-level impact. Examples: James Hamilton (VP & Distinguished Engineer). | $1M+ |

> **Note:** There is no L9 at Amazon. The gap between some levels is much larger than between others — the jump from L6 to L7 requires managerial help and is considered difficult; L10 is mostly external hires or 20+ year internal legends.

---

### Management Track — Software Development Managers (SDM)

| Level | Title | Scope |
|-------|-------|-------|
| **L5** | SDM I | Manage 4–8 engineers (one two-pizza team). First people-manager role, often an internal SDE II→SDM transition. |
| **L6** | SDM II | Manage multiple teams or a larger team. Responsible for delivery, hiring, and technical direction. |
| **L7** | Senior Manager / Principal SDM | Manage managers. Drive org-level planning and direction. |
| **L8** | Director | Lead an entire engineering organization (50–200+ engineers). P&L accountability. |
| **L10** | VP | Lead a major division's engineering function. S-Team eligible at SVP. |

---

### Other Engineering Roles at Amazon

| Role                                | Level Range | What They Do                                                                                  |
| ----------------------------------- | ----------- | --------------------------------------------------------------------------------------------- |
| **TPM** (Technical Program Manager) | L4–L8       | Drive cross-team engineering programs. No direct reports but coordinate delivery across orgs. |
| **System Engineer**                 | L4–L6       | Infrastructure and systems reliability. Common in AWS infrastructure teams.                   |
| **Support Engineer**                | L4–L6       | Enterprise customer technical support, often writes code to solve customer issues.            |
| **Solutions Architect**             | L5–L7       | Customer-facing technical advisory (AWS SA). Bridge between sales and engineering.            |
| **Data Engineer / Scientist**       | L4–L7       | Build data pipelines, ML models, analytics for Amazon's internal data (retail, ads, etc.)     |
| **Security Engineer**               | L4–L8       | AppSec, infrastructure security, pen testing, compliance. Amazon Security is a massive org.   |
| **Hardware Engineer**               | L4–L8       | Annapurna Labs — chip design, RTL, verification for Graviton, Trainium, Nitro.                |

---

### Promotion Process — How Amazon Engineers Get Promoted

Performance reviews happen twice yearly (aligned with Forte review cycles in Q1 and Q3). Promotions are manager-driven with a promotion panel review. The manager writes a formal promotion document (promo doc) using input from the engineer. For SDE II→SDE III (L5→L6), this doc is 15+ pages, structured around Leadership Principles, and reviewed adversarially by a promotion panel.

**The Promotion Gauntlet:**
1. **Forte Reviews** (twice yearly) — performance signals documented
2. **Manager writes promo doc** — structured around specific Leadership Principle examples
3. **Stakeholder endorsements** — 4–6+ endorsements required from L6+ engineers for L5→L6
4. **Promotion panel** — reviews adversarially, challenges every claim
5. **OLR** (Organization and Leadership Review) — final approval by org leadership

**Common sticking points:**
- L5→L6 is the hardest common jump — requires sustained "already operating at L6" behavior
- L7 often requires an advocate at the VP level who actively sponsors you
- L8+ is almost always driven by an SVP believing in you specifically

---

## 🔬 Inside an Amazon/AWS Engineering Team — Day-to-Day Reality

### A Typical Two-Pizza Team Structure

```
Single-Threaded Owner (L6–L7 SDE or SDM)
│
├── SDE I (L4) × 1–2       ← Implements well-defined features
├── SDE II (L5) × 2–3      ← Core feature development, design
├── Senior SDE (L6) × 1–2  ← Architecture, tech lead for the team
├── TPM (L5–L6) × 1        ← Drives roadmap, cross-team coordination
└── PM (L5–L6) × 1         ← Product decisions, PR/FAQ owner
```

### How a Feature Ships at AWS

1. **PR/FAQ** written by PM + tech lead → reviewed by leadership
2. **Design Doc** (OP1/OP2 — internal planning cycle) written by Senior SDE
3. **API design review** — reviewed by Principal or Distinguished engineer if cross-service impact
4. **Implementation** — SDEs build, code reviewed by L6+
5. **Operational Readiness Review (ORR)** — before launch: who's on-call, runbooks, rollback plan
6. **Launch** → team owns operations post-launch (you build it, you run it)
7. **COE** (Correction of Errors) — if something breaks, team writes a detailed post-mortem

### The "You Build It, You Run It" Culture

Amazon pioneered the DevOps culture of **full ownership**:
- Teams deploy their own code (no separate ops team)
- Teams own their own on-call rotation
- Teams own their SLAs and are measured on them
- If your service pages at 3am, your team wakes up — not a separate "operations" team

---

## 📊 Amazon's 14 Leadership Principles — The Engineering Filter

These are not corporate posters. At Amazon, **every engineering decision, promotion, and hire is evaluated against these principles**. In interviews, you will be asked to give STAR stories for each one.

| # | Principle | Engineering Implication |
|---|-----------|------------------------|
| 1 | **Customer Obsession** | Start from customer need, not tech coolness |
| 2 | **Ownership** | You own your code in prod at 3am |
| 3 | **Invent and Simplify** | Build elegant solutions, challenge complexity |
| 4 | **Are Right, A Lot** | Data-driven decisions, challenge bad assumptions |
| 5 | **Learn and Be Curious** | Self-directed learning, explore new tech |
| 6 | **Hire and Develop the Best** | Bar Raiser interviews, raise the bar with each hire |
| 7 | **Insist on the Highest Standards** | Don't ship mediocre; technical debt is tracked |
| 8 | **Think Big** | Don't over-scope — think at Amazon scale from day 1 |
| 9 | **Bias for Action** | Ship fast with calculated risk, don't wait for perfect |
| 10 | **Frugality** | Build efficiently; cost-per-unit matters in AWS |
| 11 | **Earn Trust** | Be transparent, admit mistakes, follow through |
| 12 | **Dive Deep** | Know your system's internals; don't delegate understanding |
| 13 | **Have Backbone; Disagree and Commit** | Push back on bad decisions, then fully commit once decided |
| 14 | **Deliver Results** | Metrics, metrics, metrics — outcomes over effort |

---

## 🌏 Amazon India — Engineering Presence

Amazon has a massive engineering presence in India, particularly in Bangalore, Hyderabad, and Chennai.

### Key India Engineering Offices

| Location | What's Built There |
|----------|--------------------|
| **Bangalore** | AWS India engineering, Alexa, retail platform, payments, Prime Video tech |
| **Hyderabad** | Largest Amazon campus outside US — AWS, retail, operations tech, data engineering |
| **Chennai** | Supply chain tech, Alexa, device software |
| **Pune** | AWS engineering, security teams |

### India-Specific Engineering Teams

- **Amazon Pay India** — UPI, payments platform (separate regulated entity)
- **Amazon India Retail** — localization, regional features, vernacular language support
- **AWS India** — datacenters in Mumbai and Hyderabad (ap-south-1 and ap-south-2 regions)
- **Alexa India** — Hindi, Tamil, Bengali NLP engineering
- **Amazon Logistics India** — last-mile delivery tech, route optimization

### India Engineering Levels → India Compensation (Approx.)

| Level | Title | India CTC Range |
|-------|-------|----------------|
| L4 | SDE I | ₹20L–₹40L |
| L5 | SDE II | ₹35L–₹65L |
| L6 | Senior SDE | ₹65L–₹1.2Cr |
| L7 | Principal SDE | ₹1.2Cr–₹2.5Cr |
| L8 | Senior Principal | ₹2.5Cr–₹4Cr+ |

*(Includes base + RSUs + joining bonus. RSUs vest over 4 years with back-weighted schedule: 5/15/40/40)*

---

## 🎯 How to Think About Amazon's Engineering Structure (Summary Model)

```
AMAZON (Andy Jassy, CEO)
│
├── WHAT AMAZON SELLS TO EXTERNAL CUSTOMERS
│   ├── Cloud infrastructure & AI → AWS (Matt Garman)
│   ├── E-commerce → Worldwide Stores (Doug Herrington)
│   ├── Digital ads → Advertising (Paul Kotas)
│   ├── Devices → Alexa/Echo/Kindle (Panos Panay)
│   └── Entertainment → Prime Video (Mike Hopkins)
│
├── HOW AMAZON ORGANIZES ENGINEERING
│   ├── Division → Org → Team (two-pizza)
│   ├── Each team = 5–8 people, 1 Single-Threaded Owner
│   ├── Teams talk via APIs only (Bezos mandate)
│   └── Teams own their product from design → build → operate
│
├── HOW AMAZON DECIDES WHAT TO BUILD
│   ├── Start with PR/FAQ (Working Backwards)
│   ├── Write 6-pager narrative (no PPT)
│   └── OP1/OP2 annual planning cycles drive investment
│
├── HOW ENGINEERS GROW
│   L4 → L5 → L6 → L7 → L8 → L10
│   SDE I → SDE II → Senior → Principal → Sr Principal → Distinguished
│   Promotions: Promo doc + Panel review + OLR
│
└── HOW AMAZON MEASURES ENGINEERS
    ├── Forte reviews (Q1 + Q3)
    ├── Leadership Principles alignment
    ├── Impact on customer-facing metrics
    └── Operational excellence (on-call, COE quality)
```

---

## 🔗 Free Sources to Explore Amazon's Structure Yourself

| Source | What You'll Find | URL |
|--------|-----------------|-----|
| **Amazon About page** | Official S-Team, press releases on org changes | aboutamazon.com |
| **SEC EDGAR DEF 14A** | Proxy statement — full executive list, compensation, reporting lines | edgar.sec.gov → search AMZN → DEF 14A |
| **Amazon Leadership Blog** | Internal memos made public (e.g., Peter DeSantis reorg announcement) | aboutamazon.com/news |
| **LinkedIn → Amazon People tab** | Filter by department/seniority to map org layers | linkedin.com/company/amazon |
| **The Org** | Community-maintained org chart | theorg.com/org/amazon |
| **Levels.fyi** | Verified compensation by level, role, and location | levels.fyi/company/amazon |
| **Glassdoor** | Internal team structure described by employees, review culture | glassdoor.com |
| **Amazon Jobs** | Job descriptions reveal team names, reporting lines, required skills | amazon.jobs |
| **AWS re:Invent talks** | Engineering leadership presents architecture and team philosophy | youtube.com (search AWS re:Invent) |

---

*Sources: Amazon official press releases, SEC DEF 14A filings (2025–2026), aboutamazon.com S-Team page, AWS Executive Insights blog, fourweekmba.com, levels.fyi, careerclimb.app, interviewkickstart.com, fearlesssalarynegotiation.com — verified June 2026.*
