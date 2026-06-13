# LocalStack → Floci: The Complete 2026 Migration Guide

## Part 1: Why It Happened — the vendor risk pattern every platform owner needs to understand

---

_By Ashutosh Kumar | Project Manager & Engineering Tools Enthusiast | June 2026_

_Part 1 of 6 — LocalStack → Floci: The Complete 2026 Migration Guide_

---

March 23, 2026 was one of those days where you get the same ticket filed by three different teams before lunch.

The symptoms were all the same: CI pipelines failing with authentication errors, engineers who'd never thought about their local AWS tooling suddenly asking why their builds were broken. If you run AWS workloads and your teams do any amount of integration testing or local development, your platform probably had a dependency on LocalStack — even if nobody had ever formally documented it.

That dependency became very visible, very fast.

This first article in the series isn't about the technical fix — that's Part 2. What I want to cover here is what this event reveals about how we evaluate infrastructure tooling. Specifically, the vendor risk pattern that LocalStack followed, and why recognising it should change how you assess similar decisions going forward.

---

## What LocalStack Was — and Why It Was Invisible

LocalStack, originally created by Atlassian engineers and later spun off as its own company, gave developers a local emulation layer for AWS services. Point your AWS SDK at `localhost:4566` and you'd get dozens of AWS services responding faithfully — S3, SQS, DynamoDB, Lambda, RDS, and more. No internet connection required. No AWS costs. No waiting.

Teams embedded it everywhere: in `docker-compose.yml` files committed to source control, in CI pipeline definitions, in onboarding scripts, in shared developer environment configs. It became the kind of infrastructure nobody audits because it always works.

That's exactly what made March 2026 so disruptive. When a tool is embedded at the foundation of your development workflow, "swap it out on Tuesday morning" isn't really an option.

---

## The Timeline

Here's how it unfolded:

**2016–2024:** LocalStack operates as open-source under Apache 2.0. Community edition freely available, no authentication, no usage limits in CI. The tool gains broad adoption across thousands of teams globally.

**2024–2025:** LocalStack raises venture capital and grows its paid Pro/Enterprise tier. Free community edition continues unchanged. No alarms going off yet.

**December 2025:** LocalStack co-CEOs announce the community edition will be consolidated into a single paid-tier image. The stated reason: maintaining high-fidelity AWS emulation had become too expensive to sustain in a free, unauthenticated tier.

**February 2026:** Pricing restructure announced. A Hobby tier (free, non-commercial) is introduced. Paid plans start at $39/seat/month. The Hobby tier doesn't include CI credits — which is exactly the usage pattern most teams relied upon.

**March 23, 2026:** Repository archived. Docker image requires `LOCALSTACK_AUTH_TOKEN`. Pipelines using `docker pull localstack/localstack` without a token break.

**April 6, 2026:** Grace period bypass flag expires. No remaining workaround for unauthenticated use.

That last step is when teams relying on the bypass flag discovered they'd been on borrowed time.

---

## This Pattern Has Happened Before

What makes the LocalStack situation worth studying isn't that it happened — it's that this is the third major example of the same pattern in three years.

**Elasticsearch → OpenSearch (2021):** Elastic relicensed Elasticsearch to SSPL, restricting cloud provider use. AWS forked it as OpenSearch, now hosted independently.

**Terraform → OpenTofu (2023):** HashiCorp moved Terraform from MPL 2.0 to the Business Source License. The community forked it as OpenTofu under the Linux Foundation.

**Redis → Valkey (2024):** Redis Ltd relicensed Redis to RSALv2 and SSPLv1. The community forked it as Valkey, also under the Linux Foundation.

**LocalStack → Floci (2026):** LocalStack moved the community edition behind authentication and a paywall. But this time, the community didn't fork LocalStack. Instead, someone built a replacement from scratch.

That last detail matters, and I'll come back to it.

Each of these followed the same arc: tool gets widely adopted while free, raises money, then the commercial model collides with the free tier. The community reacts — but whether that reaction produces anything useful depends on one thing nobody asks until they're already stuck: is the original codebase actually forkable?

---

## Why No LocalStack Fork Emerged

OpenSearch, OpenTofu, and Valkey all succeeded as forks because the original codebases were clean enough, modular enough, and maintainable enough for a community to pick up and run with.

LocalStack's codebase is a different story. It spans a massive custom implementation of 80+ AWS service APIs in Python, accumulated over nine years. As of June 2026, no credible fork has materialised. The community has instead rallied around Floci — a completely new implementation written in Java, using Quarkus and GraalVM, started from scratch in early 2026.

The lesson here for anyone assessing long-term platform risk: the forkability of a tool's codebase is a legitimate evaluation criterion, not an afterthought. A permissive licence on an unmaintainable codebase offers much less protection than it appears to. If the project's direction changes and nobody can realistically fork it, the community is left with no good options — only expensive ones.

---

## Floci: What Emerged Instead

Floci (`floci-io/floci`) is the replacement that gained traction. Created by Hector Ventura, it's built on an entirely different technical foundation than LocalStack:

- **Language:** Java (not Python)
- **Framework:** Quarkus — a Kubernetes-native Java framework backed by Red Hat
- **Compilation:** GraalVM Mandrel — ahead-of-time native compilation, no JVM at runtime
- **Licence:** MIT — no restrictions on commercial or CI use, ever
- **Authentication:** None required
- **Telemetry:** None collected

The name comes from _cirrocumulus floccus_ — a small, lightweight cloud formation. The design intent matches the name: minimal footprint, fast startup, zero overhead.

It runs on port 4566, the same port LocalStack uses, so switching requires no changes to existing application code or tooling. It crossed 10,000 GitHub stars in May 2026 — less than four months after initial release.

---

## What This Changes About How You Evaluate Tools

The honest version of the lesson here isn't "avoid LocalStack" or "only use projects under the Linux Foundation." It's more uncomfortable: developer communities form deep dependencies on free tooling and treat promises of openness as permanent. When those promises change, the trust damage is severe.

If I were running a vendor assessment today before embedding any open-core tool this deeply, I'd want clear answers to at least these:

**What's the actual business model?** Venture-backed open-core tools have structural pressure to monetise the free tier. That pressure doesn't ease as the company matures — it increases.

**What's the licence on the binary, not just the source?** LocalStack's source remained Apache 2.0. The binary became proprietary. Those aren't the same thing.

**Could someone realistically fork this if needed?** A permissive licence on a complex, decade-old, undocumented monolith is less protection than it appears.

**What breaks if this goes behind a paywall tomorrow?** If it's wired into CI, onboarding scripts, and integration test suites, the blast radius is high enough to warrant a real assessment — not "we'll deal with it if it happens."

None of these are new questions. March 2026 just made them harder to ignore.

---

## What's Next

Part 2 gets into the actual comparison: Floci vs. LocalStack paid across service coverage, cost at team scale, IaC toolchain fit, and risk — with a decision matrix for the most common team profiles.

Part 3 covers the hybrid model for teams where a clean Floci-only migration isn't yet possible.

Parts 4–6 are the hands-on migration guide: docker-compose before/after, CI pipeline changes, benchmark numbers, known edge cases, and Testcontainers setup in Java, Python, and Go.

---

_Has your organisation formally assessed its open-source infrastructure dependencies for vendor risk? What evaluation criteria do you apply? The conversation in the comments is worth having before the next March 23._

---

**Tags:** `#CloudArchitecture` `#VendorRisk` `#OpenSource` `#LocalStack` `#Floci` `#AWSStrategy` `#TechLeadership` `#PlatformEngineering` `#EnterpriseCloud`

---

_About the author: Ashutosh Kumar is a Project Manager with 15 years of experience, currently exploring developer tooling, cloud workflows, and engineering process improvements. He writes at github.com/askuma._
