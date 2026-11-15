# Lesson 5: Practice — Video Streaming Service

## Interview prompt

Design a data model for a subscription video streaming service. Users browse a catalog, subscribe to plans, play content across devices, pause/resume, download titles, and rate or watchlist content. The company needs engagement, content performance, subscription, and revenue analytics.

## 1. Clarify the business problem

Ask:
- Is viewing measured by playback start, heartbeat, completion, or watch seconds?
- Can an account have multiple profiles?
- Can a profile stream on multiple devices concurrently?
- How are episodes grouped into seasons and series?
- How are ads represented?
- What is the subscription and household/account relationship?
- How are trials, pauses, cancellations, and grace periods handled?
- What freshness is needed for operations versus finance?

## 2. Candidate conceptual diagram

```text
ACCOUNT 1 ─────< PROFILE
ACCOUNT 1 ─────< SUBSCRIPTION
PROFILE 1 ─────< PLAYBACK_SESSION >──── 1 CONTENT
PLAYBACK_SESSION 1 ─────< PLAYBACK_EVENT
PROFILE ─────< WATCHLIST_ITEM >──── CONTENT
CONTENT ─────< CONTENT_AVAILABILITY >──── REGION
SUBSCRIPTION 1 ─────< BILLING_TRANSACTION
```

Content may be organized hierarchically: series, season, episode, film, or other content types. Model the hierarchy explicitly rather than assuming every content item is an episode.

## 3. Candidate grains

| Fact | Grain |
|---|---|
| Playback session | One row per playback session |
| Playback event | One row per playback lifecycle/heartbeat event |
| Viewing interval | One row per normalized viewing interval, if derived |
| Subscription event | One row per subscription lifecycle event |
| Billing transaction | One row per charge, refund, or adjustment |
| Content availability | One row per content-region-rights-period combination |
| Rating | One row per rating submission or rating change |

A playback session and its heartbeat events are different grains. Avoid treating every heartbeat as a separate viewing session.

## 4. Modeling viewing metrics

Possible measures:
- Playback starts.
- Qualified starts.
- Watch seconds.
- Completion percentage.
- Unique viewers.
- Concurrent streams.
- Rebuffer events.
- Playback failures.

Define whether watch time excludes paused periods, background playback, previews, trailers, and repeated viewing. Completion percentage should specify whether the denominator is runtime or an eligible content duration.

## 5. Subscription history

Subscription lifecycle includes trial start, activation, renewal, pause, resume, cancellation, expiration, and plan change. A subscription event fact preserves transitions; a periodic snapshot can show daily subscription state and support active-subscriber trends.

Revenue analytics must distinguish billing transactions from recognized revenue. Finance defines the applicable accounting rules.

## 6. Interviewer–candidate conversation

**Interviewer:** How would you calculate total watch time?

**Candidate:** I'd first define eligible viewing. I'd use playback events or normalized viewing intervals to calculate watch seconds, excluding paused or invalid intervals as agreed. I'd deduplicate retries and handle overlapping sessions carefully.

**Interviewer:** Why not use playback session duration?

**Candidate:** Session wall-clock duration may include pauses, buffering, idle time, or abandoned playback. It may overstate actual viewing. We need event semantics and quality rules.

**Interviewer:** How do you model account sharing across profiles?

**Candidate:** I'd keep account and profile as separate entities. Playback is associated with the profile where that information is reliable, while billing and subscription are usually account-level. I'd avoid assuming the profile is the payer.

## 7. Follow-up questions
1. How do you model concurrent streams?
2. How do you calculate content completion?
3. How do you represent regional licensing windows?
4. How do you track plan changes historically?
5. How do you distinguish billed revenue from recognized revenue?
6. How do you handle offline downloads and later playback synchronization?

## 8. Principal-level discussion
- How would you support near-real-time concurrency monitoring and daily finance reporting?
- How would you govern account/profile identity?
- How would you manage late mobile events and offline playback?
- How would you scale heartbeat processing without making every report scan raw heartbeats?

## Evaluation rubric
- Requirement clarification: 20%
- Entity and hierarchy design: 20%
- Grain and event semantics: 25%
- Subscription and financial history: 15%
- Scale and latency: 10%
- Communication: 10%

## Key takeaway
**Streaming analytics requires precise playback semantics and a clear separation between account, profile, content, subscription, session, and event grains.**
