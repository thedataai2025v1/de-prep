# Module 2: Gathering Business Requirements

## Lesson 2.2 — Recognizing the Core Business Problem

### Why This Step Is the Most Important

In a data modeling interview, **everything flows from correctly identifying the business problem**. If you solve the wrong problem, no amount of elegant schema design will save you. Interviewers often embed the *real* need inside a vague or incomplete prompt — your job is to surface it.

> **Rule of thumb:** Spend the first 5–8 minutes clarifying the problem. A candidate who jumps straight into drawing tables signals "junior." A candidate who asks sharp questions signals "senior."

---

### 1. How to Identify the Core Business Problem

When an interviewer says, *"Design a data warehouse for a ride-sharing platform,"* that is **not** a complete problem statement. It's a starting point. Follow this mental checklist:

| Question | What You're Looking For |
|---|---|
| **Who is the end user?** | Executives? Product managers? ML engineers? Support teams? |
| **What decisions are they making?** | Pricing adjustments? Driver incentives? Route optimization? |
| **What actions follow the analysis?** | Does this feed a dashboard, an automated system, or a report? |
| **What would happen if this data were wrong?** | Revenue loss? Regulatory fines? Churn? |

#### The "Three Whys" Technique

Peel back layers by asking **"why"** three times. Each answer reveals a deeper business need:

```
Interviewer: "We need a data model for our ride-sharing app."

You: "What decisions will this support?"
Interviewer: "We want to improve driver availability."

You: "Why is driver availability a problem?"
Interviewer: "Riders are canceling because they can't get a ride in 10 minutes."

You: "Why can't they get a ride in 10 minutes?"
Interviewer: "Because drivers aren't positioned in high-demand zones during peak hours."

Core problem identified: We need to model supply-demand gaps by zone and time
to inform dynamic incentives and driver dispatch strategies.
```

> **Key takeaway:** The core problem is almost never "I need a data model." It's always "I need to make a specific business decision better."

---

### 2. Mapping Business Problems to Data Requirements

Once you've identified the core problem, translate it into concrete data requirements:

| Business Problem | Implied Data Requirements | Fact Table Grain | Key Dimensions |
|---|---|---|---|
| Optimize pricing by demand | Ride transactions, surge multipliers, time, location | Per ride | Time, location, driver, rider, market |
| Reduce driver cancellations | Driver sessions, acceptance rates, wait times | Per ride request | Driver, time, zone, device type |
| Improve content recommendations | User interactions, content attributes, sessions | Per event | User, content, session, device |

#### The Translation Framework

```
Business Problem  →  Key Metrics  →  Events/Transactions  →  Fact Table Grain
     │                    │                   │                     │
     ▼                    ▼                   ▼                     ▼
"Increase rider        "Completion       "Ride start,             One row per
 retention"             rate, churn,"      ride end, cancel"       completed ride
```

**Always state the grain explicitly.** This is the single most important design decision:

- **Transaction fact:** One row per individual event (e.g., per ride, per click)
- **Periodic snapshot:** One row per entity per time period (e.g., daily inventory levels)
- **Accumulating snapshot:** One row per process with status milestones (e.g., order fulfillment stages)

---

### 3. Common Interview Scenarios — Deep Dives

#### A. Ride-Sharing (e.g., Uber, Lyft)

**Core problems you might encounter:**
- **Supply-demand matching:** Are drivers where riders need them?
- **Pricing optimization:** When and how much to surge?
- **Driver retention:** Why do drivers leave? What keeps them?
- **Rider experience:** Completion rate, wait time, cancellation rate

**Interviewer Questions You'll Face:**
1. "How would you model ride-matching efficiency?"
2. "What's the grain of your main fact table for rider analytics?"
3. "How would you track a driver's journey through onboarding → active → churned?"
4. "How do you handle ride data that arrives incrementally (start event, then end event hours later)?"

**Questions YOU Should Ask the Interviewer:**

| Category | Questions to Ask |
|---|---|
| **Business** | "Is the focus on rider metrics, driver metrics, or both?" "Are we optimizing for revenue, rider retention, or driver supply?" |
| **Metrics** | "What are the top 3 KPIs?" "Is completion rate more important than wait time?" |
| **Volume** | "How many rides per day? How many driver sessions?" |
| **Latency** | "Do pricing decisions need real-time data, or is nightly batch acceptable?" |

**Top-Up Questions (Senior-Level Probes):**
- "How would you design an accumulating snapshot for the ride lifecycle from request → matched → in-progress → completed/canceled?"
- "How do you handle the fact that a ride spans multiple zones over time?"
- "What's your strategy for attributing a rider's churn to a specific experience?"

---

#### B. E-Commerce (e.g., Amazon, Shopify)

**Core problems you might encounter:**
- **Conversion funnel analysis:** Where do shoppers drop off?
- **Inventory optimization:** Avoid stockouts and overstock
- **Personalization:** Product recommendation, search ranking
- **Fraud detection:** Payment anomalies, return abuse

**Interviewer Questions You'll Face:**
1. "Walk me through how you'd model the purchase funnel."
2. "How would you handle returns in your fact table?"
3. "What grain would you use for inventory tracking?"
4. "How do you model a product that moves between categories over time?"

**Questions YOU Should Ask the Interviewer:**

| Category | Questions to Ask |
|---|---|
| **Business** | "Is this for the merchandising team, the product team, or finance?" |
| **Metrics** | "Are we tracking conversion rate, AOV, GMV, or something else?" |
| **Volume** | "How many products? How many transactions per day?" |
| **Complexity** | "Do we need to handle multi-item carts, subscriptions, or returns?" |

**Top-Up Questions (Senior-Level Probes):**
- "How would you model product hierarchy changes (SCD) when categories reorganize?"
- "How do you handle split shipments in your fulfillment fact table?"
- "What's your approach to modeling promotional attribution when a customer sees multiple coupons?"

---

#### C. Social Media (e.g., Instagram, Twitter)

**Core problems you might encounter:**
- **Engagement measurement:** What drives likes, shares, comments?
- **Content recommendation:** What should appear in each feed?
- **User growth & retention:** Acquisition, activation, churn
- **Abuse detection:** Spam, fake accounts, harmful content

**Interviewer Questions You'll Face:**
1. "How would you model user engagement across different content types?"
2. "What's the grain for tracking who viewed what post?"
3. "How would you measure the effectiveness of a new feed algorithm?"
4. "How do you handle billions of daily events?"

**Questions YOU Should Ask the Interviewer:**

| Category | Questions to Ask |
|---|---|
| **Business** | "Is the priority growth, engagement, or content moderation?" |
| **Metrics** | "What defines 'engagement' — views, likes, time spent, shares?" |
| **Volume** | "How many daily active users? How many events per user per day?" |
| **Retention** | "Do we need to track user behavior changes over weeks or months?" |

**Top-Up Questions (Senior-Level Probes):**
- "How would you model A/B test results for feed algorithm comparisons?"
- "What's your strategy for building a user activity summary table from raw events?"
- "How do you model content reach when a post is reshared through multiple layers?"

---

#### D. Healthcare (e.g., Hospital System, Telehealth)

**Core problems you might encounter:**
- **Patient outcomes:** Readmission rates, treatment effectiveness
- **Compliance & auditing:** HIPAA, billing accuracy, documentation
- **Resource allocation:** Staff scheduling, bed availability
- **Population health:** Risk stratification, preventive care gaps

**Interviewer Questions You'll Face:**
1. "How would you model patient episodes of care?"
2. "How do you handle PII and PHI in your data warehouse?"
3. "What fact tables would support measuring readmission rates?"
4. "How would you model the relationship between a patient, encounters, and providers?"

**Questions YOU Should Ask the Interviewer:**

| Category | Questions to Ask |
|---|---|
| **Business** | "Are we focused on clinical outcomes, operational efficiency, or billing?" |
| **Metrics** | "What are the key patient outcome measures?" |
| **Compliance** | "What are the HIPAA and data governance requirements?" |
| **Volume** | "How many patients? How many encounters per day?" |

**Top-Up Questions (Senior-Level Probes):**
- "How would you model the 30-day readmission window in an accumulating snapshot?"
- "How do you handle data from multiple EHR systems with different schemas?"
- "What's your approach to anonymizing PHI while preserving analytical value?"

---

### 4. Red Flags: Signs You've Misunderstood the Problem

Watch for these signals — they mean you should pause and re-clarify:

| Red Flag | What It Means | What to Do |
|---|---|---|
| You started designing tables before asking "what decisions will this support?" | You solved a technical problem, not a business one | Stop and restate: "Let me confirm the business goal before I proceed." |
| Your fact table grain doesn't map to any business metric | You're modeling the wrong events | Ask: "Which business metric would this row count represent?" |
| You can't explain why any column exists | You're adding unnecessary complexity | Remove columns you can't justify with a business use case |
| The interviewer interrupts with "That's not what we use for" | You missed a key requirement | Pivot: "Help me understand — what's the actual use case for this data?" |
| You designed for a different time grain than the business needs | Your queries will be too slow or too granular | Ask: "What's the typical query window — daily, weekly, monthly?" |
| You forgot about data consumers | Your model may not support their tools | Ask: "Will this be queried via SQL, a BI tool, or an API?" |

---

### 5. Your Opening Script — A Template

Use this as your go-to response when the interviewer presents a vague prompt:

> "Before I jump into design, let me make sure I understand the problem:
>
> 1. **What decisions** will this data warehouse support? Who's making them?
> 2. **What are the top KPIs** — the 3–5 metrics that matter most?
> 3. **Who are the primary consumers** — analysts, executives, ML models?
> 4. **What's the data volume** — events per day, distinct users, growth rate?
> 5. **What latency is needed** — real-time, hourly, daily batch?
> 6. **How far back** do we need to retain and query data?
>
> Once I have clarity on those, I'll sketch the high-level model and walk through my design choices."

---

### Practice Exercise

> **Scenario:** You're interviewing at a food delivery platform (like DoorDash or Uber Eats). The interviewer says:
>
> *"We want to build a data warehouse to help us understand why customers are ordering less frequently. Design the data model."*
>
> **Your tasks:**
>
> 1. **Apply the "Three Whys":** Write out three follow-up questions that peel back layers of this problem. What might the *real* core problem be?
>
> 2. **State the core problem** in one sentence. (Hint: Is it really about "ordering less" — or is it about churn, delivery experience, pricing, or restaurant availability?)
>
> 3. **Identify 3–5 key metrics** you'd need to measure.
>
> 4. **Define the grain** of your primary fact table. Justify your choice.
>
> 5. **List 4 clarifying questions** you'd ask the interviewer using the template above.
>
> 6. **Spot a red flag:** If your first instinct was to model every ingredient in every order, what red flag does that reveal?

---

### Related Lessons

- ⬅️ [Lesson 5: Introduction to Gathering Business Requirements](./05-introduction-to-gathering-business-requirements.md) — the questioning approach
- ➡️ [Lesson 7: Analyzing Metrics](./07-analyzing-metrics.md) — decompose those KPIs
- ➡️ [Lesson 12: Example Business Requirements Gathering](./12-example-business-requirements-gathering.md) — walk-through of a real BA session
