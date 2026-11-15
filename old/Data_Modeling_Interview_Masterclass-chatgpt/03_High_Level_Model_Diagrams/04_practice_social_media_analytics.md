# Lesson 4: Practice — Social Media Analytics

## Interview prompt

Design a high-level analytical model for a social platform. Users publish posts, follow other users, react, comment, share content, view feeds, and report abuse. Product teams want engagement, growth, creator, and content-quality analytics.

## 1. Clarify the business questions

- Daily and monthly active users.
- New user activation and retention.
- Engagement by content type.
- Creator performance.
- Follower growth and network analysis.
- Feed impression-to-engagement conversion.
- Abuse reports and moderation outcomes.

Clarify definitions: Does an impression mean rendered, visible for a threshold, or merely returned by the feed service? What counts as an active user? Are reactions and shares reversible?

## 2. Candidate entities and relationships

```text
USER 1 ─────< POST
USER 1 ─────< COMMENT >──── 1 POST
USER 1 ─────< REACTION >─── 1 POST
USER 1 ─────< SHARE >────── 1 POST

USER ─────< FOLLOW_RELATIONSHIP >──── USER

USER 1 ─────< FEED_IMPRESSION >──── 1 POST
USER 1 ─────< ABUSE_REPORT >─────── 1 POST
ABUSE_REPORT ─────< MODERATION_ACTION
```

A follow relationship is directional: follower and followed user are different roles. It is naturally many-to-many between users and should have a clear grain, such as one row per follow relationship over its lifecycle or one row per follow/unfollow event.

## 3. Candidate fact grains

| Fact | Grain |
|---|---|
| Post event | One row per post creation/update/deletion event, if event history is required |
| Reaction event | One row per reaction add/remove event |
| Comment | One row per comment |
| Share event | One row per share action |
| Feed impression | One row per eligible impression event |
| Follow event | One row per follow/unfollow action |
| Moderation action | One row per moderation decision/action |

Do not collapse impressions, views, likes, and comments into a single generic engagement row without preserving event type and semantics.

## 4. Analytical modeling considerations

Dimensions may include date/time, user, post/content, content type, device, geography, and moderation reason. User attributes and creator status may need historical tracking.

High-volume impression events may require separate storage and aggregation strategies from lower-volume post metadata. Privacy and access restrictions are especially important for user-level behavioral data.

## 5. Metric definitions

- DAU: distinct eligible users with a qualifying activity during a defined day.
- Engagement rate: must specify numerator, denominator, and whether it is impression-, reach-, or user-based.
- Retention: define cohort entry event and return activity window.
- Creator reach: define unique viewers and attribution window.
- Follow conversion: define eligible profile visits and resulting follow actions.

Distinct counts are generally non-additive across dates and segments. Summing daily active users does not produce monthly active users.

## 6. Interviewer–candidate conversation

**Interviewer:** How would you model the follower graph?

**Candidate:** I'd represent the directed relationship with follower_user_id and followed_user_id, plus created and ended timestamps or separate follow/unfollow events. I'd clarify whether re-following creates a new relationship episode and whether the current state is also needed.

**Interviewer:** Can you sum DAU to get MAU?

**Candidate:** No. A user active on multiple days would be counted multiple times. MAU is a distinct-user count over the month. We may maintain user activity or carefully designed aggregate structures, but the metric semantics must remain distinct.

**Interviewer:** We receive billions of impressions. Would you join them directly to every post and user attribute?

**Candidate:** I'd preserve the impression grain and use governed keys. For common analytics, I'd aggregate to appropriate grains and use conformed dimensions or snapshots. I'd evaluate retention, clustering, and query patterns rather than repeatedly scanning raw event data for every dashboard.

## 7. Follow-up questions
1. How do you model edits and deletions of posts?
2. How do you count unique viewers?
3. How do you handle bots and invalid traffic?
4. How do you model a reaction that is changed from like to love?
5. How do you avoid double counting when one post has multiple comments and reactions?
6. How do you support cohort retention?

## 8. Principal-level discussion
- How do you balance user-level analytics with privacy and purpose limitation?
- How do you manage deletion propagation through aggregates?
- How do you standardize activity definitions across product teams?
- How do you support high-cardinality, graph-oriented questions in a warehouse?

## Evaluation rubric
- Business definition clarity: 20%
- Entity/relationship correctness: 20%
- Event grain and metric semantics: 25%
- Scale and privacy: 20%
- Communication: 15%

## Key takeaway
**Social analytics is event-heavy and identity-sensitive. Preserve event semantics, directed relationships, distinct-count behavior, and privacy boundaries.**
