# Task

Run today's mock interview (day 10) using the `junior-backend-developer.md` persona, but without repeating ground already documented in `KNOWLEDGE_SUMMARY.md` from the prior 9 sessions — the user wanted the category picker to stay random (per the persona's seed-based rule), while every question's actual scenario/content had to be genuinely new, not a reworded version of something already logged. After the interview, fold what was learned back into `KNOWLEDGE_SUMMARY.md` so it stays a usable pre-interview study reference.

# What I changed

- `2026/08/02/session.md` (new) — full transcript of the 5-question session (Databases, Spring Boot/JPA, REST API, Concurrency & Locking, Testing & Debugging), each with the candidate's answer, a score, and Strong/Missing/Tip feedback, plus a session summary table.
- `KNOWLEDGE_SUMMARY.md` — added the 08-02 row to the Score Trend table and trend narrative; added 3 new Critical Misconceptions (FK/GDPR retention, `@Transactional` self-invocation, deadlock vs. lock contention); added new worked sections under Databases, Spring Boot/JPA, Concurrency & Locking, and Testing & Debugging; cross-referenced the pagination note under REST & API Design to the fuller DB-mechanism writeup; renumbered the Priority List to insert the new gaps; added two Consistent Strengths observations.
- `answer.md` — the user's own scratch file where they typed each answer during the interview; it gets overwritten per question as part of their normal workflow, so its diff (old answer → newest answer) is a side effect of the session rather than something authored by this change.

# The real problem

After 9 prior sessions, `KNOWLEDGE_SUMMARY.md` had already become a fairly complete study guide, and the persona's literal category-selection rule (seed-based random pick from 10 categories) doesn't know anything about what's already been asked — left alone, it could easily re-serve a question whose content (not just topic) was already tested and documented, burning a practice slot without surfacing new information before the real interview. The tension to resolve wasn't "how do we test this candidate" (the persona already answers that) but "how do we keep every question net-new when the picker itself has no memory of prior content."

# Why this solution

Keeping the seed-random category picker (per the user's explicit answer) preserves the broad, non-cherry-picked coverage the persona was designed to guarantee, and pushes the "must be new" constraint down to the question-authoring step instead: for whichever category the seed lands on, read `KNOWLEDGE_SUMMARY.md`'s section for that category first, then construct a concrete scenario whose specific mechanism/example doesn't overlap what's already written up there. This is the smallest change that satisfies both stated constraints at once, rather than trading one off against the other.

# Production shape

This only keeps working if `KNOWLEDGE_SUMMARY.md` stays internally organized as the sessions accumulate: category sections have to stay the authoritative "has this been asked" check before each new question is authored, the Critical Misconceptions section has to stay reserved for confidently-wrong statements (not just any gap), and the Priority List has to get re-ranked (not just appended to) each time, or it stops being a fast pre-interview read. The failure mode to watch for as more sessions land: the file growing purely additively, categories bloating with near-duplicate examples, and the Priority List silently becoming stale (old items staying ranked above newly-confirmed, more urgent misconceptions).

# Other possible approaches

1. **Override the seed-random picker and hand-pick categories straight from the Priority List** (deliberate gap-targeting) — the right call when the goal is maximizing coverage of the *worst-known* gaps as fast as possible, at the cost of the persona's designed broad/unbiased category spread. This is literally what the two immediately preceding sessions (07-29, 08-01) already did, per the summary's own notes.
2. **Append each session's findings as a dated, standalone section at the end of `KNOWLEDGE_SUMMARY.md`**, instead of weaving new material into the existing per-category prose — the right call if the file's purpose were a chronological session log rather than a topic-organized study guide, e.g. if the user cared more about auditing trend-over-time than reviewing a topic in one place before the interview.

# Why I did not choose those alternatives

1. The user was asked directly whether to keep deliberate gap-targeting (matching 07-29/08-01) or go back to the seed-random picker, and explicitly chose "random the topic" — overriding that with hand-picked categories would have ignored a direct answer given moments earlier in the same conversation.
2. `KNOWLEDGE_SUMMARY.md`'s structure (per-category sections, a dedicated Critical Misconceptions section, a ranked Priority List) was already established across all 9 prior sessions specifically so it reads as a single study reference, not a log — the Score Trend table already covers the chronological/session-by-session view, so a second dated-append section would duplicate that purpose while fragmenting topics the candidate needs to review together.

# Key concepts to learn

- **Spring AOP proxy & self-invocation** — `@Transactional` (and other annotation-driven Spring behavior) is applied by a proxy wrapping the bean; a call on `this` from inside the same class bypasses that proxy entirely.
- **Deadlock vs. lock contention** — a deadlock is a circular wait (each transaction holds what the other needs), detected near-instantly by the DB; contention is ordinary serialized waiting in one direction and is load/timeout-dependent.
- **Referential integrity vs. data retention** — a foreign key protects against orphaned rows, but satisfying it by deleting the child rows can conflict with legal/financial retention requirements; anonymizing the parent is often the correct real-world fix instead.
- **B-tree OFFSET cost** — an index can still be "used" (an Index Scan) while OFFSET pagination remains slow, because a B-tree can jump to a key value but not to an arbitrary row position.
- **Resource/threshold-based intermittent bugs** — some "sometimes fails" bugs aren't timing races at all; they're a fixed input sitting right at a resource limit (e.g. JVM stack depth), tipped over by incidental variation in already-consumed resources.

# Common mistakes

- Reusing a previously-correct rule (checked/unchecked exception rollback) without checking whether its precondition actually held for the new bug in front of you (Q2).
- Equating "many concurrent users queueing for the same lock" with "deadlock" — they are different failure modes with different fixes (Q4).
- Treating a foreign-key-constraint error as a pure data-integrity problem to resolve mechanically, without asking whether the domain (GDPR + financial records) changes what "correct" means (Q1).
- Reaching for a debugging tool ("I'll add print statements") before stating an explicit hypothesis for *why* a bug is intermittent — timing races and resource/threshold effects need different diagnostic plans (Q5).

# Small example

```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        this.saveOrder(order);        // plain call on `this` — bypasses the Spring proxy
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
        inventoryRepository.decrementStock(order); // throws -> should roll back save(), but doesn't
    }
}
```
No transaction is ever opened for this call path, because `@Transactional` only takes effect when the call arrives through the Spring proxy — from *outside* the bean, not via `this.method()` inside it.

# How to think about this next time

When a rule you've memorized ("`@Transactional` needs `rollbackFor`", "add a lock") seems to explain a new bug, trace the actual call path and mechanism first — self-invocation, lock ordering, and retention requirements each look superficially like a bug you've already solved, but resolve completely differently. And when asked to avoid re-testing old ground while keeping a random selection mechanism intact, push the novelty requirement into the content you author for that selection, not into the selection process itself.
