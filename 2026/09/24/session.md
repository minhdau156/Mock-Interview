# Mock Interview — Junior Backend Developer
**Date:** 2026-09-24

**Categories covered (date-based seed since no timestamp was available: seed = 24×137 + 9×31 = 3567, start=8, step=1; sequence: Web Security & Frontend Basics → System Design & Architecture Basics → Tooling: Git, Build & Environment; opening category (8) differs from 09-17's opener (9) per the no-repeat-opener rule; each sub-topic was checked against every prior session's knowledge summary — Web Security's formula pick (fetch/CORS) had already been covered twice (09-07 Q2, 09-13 Q2), so it shifted to the untested angle, cookies vs. sessions vs. tokens; System Design's formula pick (URL shortener / DRY vs. premature abstraction) and Tooling's formula pick (commit hygiene) were both untested, so both kept their formula picks):** Web Security & Frontend Basics, System Design & Architecture Basics, Tooling: Git, Build & Environment

---

## Q1 — Web Security & Frontend Basics

**Question:** You're reviewing a teammate's PR for a new login endpoint. After a successful login, they store a JWT in the browser's `localStorage` and attach it as a `Bearer` token on every request. A second teammate comments: "why aren't we just using a cookie instead?"

Walk me through how the server keeps track of who's logged in across requests in general — since HTTP itself doesn't remember anything between calls — and then weigh in on the localStorage-token vs. cookie question: what's the actual tradeoff, and is there a concrete risk you'd flag in that PR comment?

**Candidate's answer:**
> i think when the request go to the server we need to get the token in localStorage or session so that use this token for authentication
> the server will validate the token in each reuquest it will use the Sign key and decode the token so that it can know does the token valid or not
> i think keep in the cookie or localStorage is same

**Score: 4/10**

**Strong:**
- The JWT mechanism itself is right: client attaches the token (pulled from `localStorage`) on each request, and the server verifies it statelessly by decoding and checking the signature against the signing key — no need to look anything up server-side.

**Missing:**
- The actual question was the cookie-vs-localStorage tradeoff, and "they're the same" is the wrong conclusion. JavaScript running on your page (from an XSS bug, a compromised npm package, a malicious ad) can freely read `localStorage`, so it can steal the token and ship it to an attacker. A cookie marked `httpOnly` is invisible to JavaScript entirely — that same script literally cannot read it. That's the risk worth flagging in the PR comment.
- Missing the other side: cookies aren't free either — the browser auto-attaches them to any request to that domain, including ones triggered by a malicious third-party page, which is what CSRF exploits. That's why `SameSite` cookie attributes exist. A complete answer names both risks.
- Jumped straight to token-based auth without naming the other standard mechanism: server-side sessions, where the server stores actual session state (memory/Redis) and the cookie just carries an opaque session ID, versus a JWT which carries everything in the token itself with no server-side storage needed.

**Tip:**
- When comparing storage options for anything sensitive, ask "what can JavaScript touch here?" — fastest way to find the real security gap.

---

## Q2 — System Design & Architecture Basics

**Question:** Your team is asked to build a URL shortener: paste a long URL, get back a short code, and hitting the short URL redirects to the original. At a junior scale — what's the API, what does the database table look like, and what's the one part of this that's actually going to become a bottleneck once it sees real traffic?

Second part: a teammate submits the PR for this and has already built it as an interface `ShortCodeGenerator`, three separate implementations of it, a factory to pick between them, and a strategy pattern wiring it together — for a service that just needs to generate a random 7-character code. Is that the right call at this stage, and what would you say in the review?

**Candidate's answer:**
> i think the API should be the short url and when we go to the server we will look up the DB have 2 field short url and real url so that we get the real url and redirect the user to it
>
> but it will have the problem when having a lot of traffic that go into the server in the short time it will hit the DB so the DB will down and the server can be down
>
> i think if the service want to have many method to generate random code we should keep that but in this case it just need a random code so that we just need have 1 class that generate code and it can make the code less redundant

**Score: 6/10**

**Strong:**
- Correctly designs the redirect flow: look up the short code in a table with a short-code/original-URL pair, then redirect.
- Good instinct on the bottleneck: every click triggers a DB lookup, so under real traffic that redirect path becomes the hot spot.
- Correctly pushes back on the strategy-pattern PR: since there's only one way to generate the code right now, a single class is the right call — and correctly noted the pattern would make sense if there were genuinely multiple interchangeable generation methods to swap between.

**Missing:**
- "What's the API" only covered the redirect side. Missing the write path — the endpoint that creates the short URL, e.g. `POST /urls { "url": "..." }` returning the generated short code, `201 Created`. A URL shortener needs both a create endpoint and a redirect endpoint.
- "the DB will down and the server can be down" is imprecise — it's not that the DB crashes outright, it's that every redirect is a synchronous DB read, so as traffic grows that read path becomes the bottleneck (rising latency, connection pool exhaustion) before anything "goes down."
- The word "redundant" mischaracterizes the strategy-pattern problem — it's not code duplication, it's unnecessary indirection: four extra files (interface, 3 impls, factory) a future reader has to jump through to understand "how does a code get generated," for zero present benefit. That's the YAGNI/premature-abstraction argument.

**Tip:**
- When judging "is this abstraction earned," ask "do I have two real cases today that need to differ," not "might I need this someday."

---

## Q3 — Tooling: Git, Build & Environment

**Question:** You're pairing with a junior who's been working for 3 hours: they fixed a typo in a log message, refactored a helper method, and added a new email-validation feature. They're about to run `git add .` and commit all of it as one commit titled "updates."

What would you tell them — and why does this matter beyond just tidiness? What actually breaks later if commits look like this?

**Candidate's answer:**
> i think they should seperate the commit at each thing they do . First they can know which commit do thing and when have the problem in this we need to rollback at this code but it will contain the bug or the method that we don't fix or refactor at this time

**Score: 6/10**

**Strong:**
- Identified a real, concrete consequence rather than just "it's messy": if the email-validation feature turns out buggy and needs reverting, you're forced to also revert the harmless typo fix and refactor bundled into the same commit — there's no way to undo just the bad part without manual surgery.

**Missing:**
- `git bisect` — the sharpest concrete answer here. Bisect binary-searches commits to find which one introduced a regression, but if three unrelated changes are squashed into "updates," bisect can point at the right commit without telling you which of the three changes inside it is the actual culprit.
- Code review: a reviewer looking at "updates" can't tell what to focus on — a typo fix needs a glance, a new feature needs real scrutiny, a refactor needs a check that behavior didn't change.
- `git blame`/history usefulness: months later, someone runs `blame` on the buggy line and gets "updates" instead of a message that actually explains what and why.

**Tip:**
- Commit each time you'd naturally say "okay, that part's done" out loud — that's usually exactly the logical boundary.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Web Security & Frontend Basics | 4/10 |
| 2 | System Design & Architecture Basics | 6/10 |
| 3 | Tooling: Git, Build & Environment | 6/10 |

**Overall: 5.3/10**

**Strengths:**
- Solid grasp of the JWT verification mechanism itself (client sends token, server validates signature statelessly).
- Good instinct for spotting bottlenecks under load and pushing back on unnecessary abstraction (the strategy-pattern PR) with real reasoning, not just a style preference.
- Gives concrete "what actually breaks" reasoning rather than vague answers — e.g. tying commit hygiene to rollback safety.

**Areas to review before the real interview:**
- Security tradeoffs of `localStorage` vs. cookies for tokens — XSS vs. CSRF, `httpOnly`, `SameSite`.
- Full API design habit: remember to cover both the write path and the read path, not just one.
- Precision in describing system failure modes ("hot path getting hammered" vs. "the DB goes down").
- Broader consequences of poor commit hygiene beyond rollback: `git bisect`, code review clarity, `git blame` history.
