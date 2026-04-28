# Contributing

This guide stays small on purpose. Treat it as a curated reading list
with opinions, not as a comprehensive textbook.

## What we accept

- **Factual corrections** with a primary-source citation (vendor docs,
  CNCF specs, RFCs, books, named industry voices).
- **New citations** for an existing claim, especially when they
  strengthen or qualify it.
- **Clarification PRs** that make an existing recommendation easier to
  apply.
- **New anti-patterns** with a concrete failure mode and a named
  replacement.
- **New `concrete check:` lines** that turn an opinion into something
  reviewable in a PR.

## What we usually decline

- "Everything about DevOps." We pick one recommended path per topic
  and defend it; we list runners-up briefly with the trade-off.
- Vendor-specific click-paths ("go to portal.azure.com…"). We cite
  IaC instead.
- Topic creep into adjacent crafts (data engineering, ML platform,
  embedded). Those are sibling guides, not this one.
- Trend-chasing. If a claim cites only a single team blog post from
  the last 12 months and contradicts the canonical sources, it's a
  blog post, not guidance.

## Style

Each chapter follows the same shape:

```
[CATEGORY] [SEVERITY: must|should|prefer|avoid] one-line opinion
  Source: <url or book + page>
  Concrete check: how a reviewer can verify this in a PR or repo
```

- `must` — load-bearing; production failure or compliance risk if missed.
- `should` — strongly recommended; deviate only with documented reason.
- `prefer` — taste call backed by evidence; reasonable people may differ.
- `avoid` — explicitly rejected; replacement is named.

## Sourcing rules

- Every `must` finding needs **≥2 independent sources**.
- Prefer primary sources (vendor docs, CNCF/CNCF-TAG papers, RFCs,
  books) over secondary (blog posts).
- Industry-voice citations (Charity Majors, Liz Fong-Jones, Kelsey
  Hightower, Cindy Sridharan, Tammy Butow, Brendan Gregg, Kief Morris,
  etc.) are valid when they're the original popularizer of the idea.
- Cite the *specific page or section*, not the homepage of a doc set.

## Page budget

150–400 lines per chapter. If a chapter is sprawling, split or trim
— don't blow past 400.
