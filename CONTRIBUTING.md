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

Each chapter follows one of two equivalent shapes; declare which at the top
of the chapter (most chapters already do via a `> Conventions:` block).

**Form A — bullet form (default for ch01, ch02, ch03, ch07, ch08, ch10, ch11):**

```
[CATEGORY] [SEVERITY: must|should|prefer|avoid] one-line opinion
  Source: <url or book + page>
  Concrete check: how a reviewer can verify this in a PR or repo
```

**Form B — `Do/Don't/Prefer` prose form (used in ch04, ch05, ch06, ch09):**

```
**Do** … one-line opinion. Source: <url or book + page>. Concrete check: …
**Don't** … explicitly rejected pattern with a named replacement.
**Prefer** … taste call backed by evidence.
```

Severity mapping between the two forms:

- `must`  ≡ `Do` (load-bearing; production failure or compliance risk if missed).
- `should` ≡ `Prefer` (strongly recommended; deviate only with documented reason).
- `prefer` ≡ `Prefer` (taste call backed by evidence; reasonable people may differ).
- `avoid` ≡ `Don't` (explicitly rejected; replacement is named).

A chapter picks one form and uses it consistently. Mixing forms inside one
chapter is rejected in review.

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
