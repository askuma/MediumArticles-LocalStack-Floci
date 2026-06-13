# LocalStack → Floci: The Complete 2026 Migration Guide

## Part 3: The Hybrid Posture — when Floci doesn't cover everything yet

---

_By Ashutosh Kumar | Project Manager & Engineering Tools Enthusiast | June 2026_

_Part 3 of 6 — LocalStack → Floci: The Complete 2026 Migration Guide_

---

_Part 1 covered what the LocalStack transition revealed about vendor risk. Part 2 compared LocalStack paid and Floci across the dimensions that matter for platform decisions. This article covers the practical model for most enterprise teams. Parts 4–6 are the hands-on implementation._

---

In practice, very few enterprise platforms can make a clean, all-or-nothing switch here. Floci covers 45 services but has no IAM enforcement and no CDK/Terraform integration. LocalStack paid covers 80+ services but comes with a vendor relationship, auth overhead, and a licence cost that adds up fast at team scale.

The answer I'd actually give most teams: stop treating it as a binary. Use both — deliberately, for different things — rather than forcing one tool to cover everything.

---

## Why a Hybrid Makes Sense

Think about what your engineers actually need from a local AWS emulator on any given day:

An application developer testing SQS message flow needs fast, reliable SQS emulation. A backend engineer writing Lambda functions needs Lambda to actually execute their code. A platform engineer validating a CDK stack needs `cdklocal deploy` to work end-to-end. A security engineer writing IAM policy tests needs permission evaluation to behave like real AWS.

These are four different requirements. Expecting one tool to serve all four at zero cost is what built the LocalStack dependency to begin with — and what made March 2026 so painful.

Splitting across tools also makes the setup more resilient. If any single tool's trajectory changes, the others still cover their ground. You're not one pricing announcement away from a broken development environment.

---

## Layer 1: Floci as the Primary Local Emulator

For the majority of everyday application development and CI/CD integration testing, Floci is the right tool.

It handles SQS, SNS, S3, DynamoDB, SSM, Secrets Manager, Cognito, KMS, API Gateway, CloudWatch, EventBridge, Step Functions, Route53, and more — all on port 4566, zero authentication, MIT licence. For Lambda, RDS, ElastiCache, ECS, and EKS, Floci runs real Docker containers underneath. Lambda actually executes your function code. RDS actually runs PostgreSQL or MySQL. These aren't mocks — test results are more trustworthy because of it.

This is the right choice for application integration testing, CI/CD pipeline test stages, developer local environments, and onboarding new engineers. Zero friction: no token, no account registration, `docker pull` and run.

```yaml
services:
  floci:
    image: hectorvent/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - FLOCI_DEFAULT_REGION=us-east-1
      - FLOCI_STORAGE_MODE=hybrid # memory | hybrid | persistent | wal
      - LOCALSTACK_PARITY=true # translates LOCALSTACK_* env vars automatically
```

Storage modes and what they mean in practice:

- `memory` — fastest, data lost on restart; use this in CI
- `hybrid` — default; in-memory with async disk flush every 5 seconds
- `persistent` — full disk persistence across restarts
- `wal` — write-ahead log for maximum durability; use for long-running local environments

---

## Layer 2: Specialist Tools for Where Fidelity Matters

Some services have purpose-built emulators that predate both LocalStack and Floci and have years of compatibility work behind them. For test scenarios where edge-case behaviour is what you're actually testing — DynamoDB transaction semantics, SQS FIFO ordering guarantees, S3 Object Lock compliance — these are worth running alongside Floci:

| Tool                              | Service           | Why Use It                                                          |
| --------------------------------- | ----------------- | ------------------------------------------------------------------- |
| **Amazon DynamoDB Local**         | DynamoDB          | Official AWS tool, highest fidelity for DynamoDB-specific behaviour |
| **ElasticMQ**                     | SQS               | Production-fidelity SQS including FIFO edge cases                   |
| **S3Mock** (Adobe)                | S3                | High-fidelity S3 for complex multipart and versioning scenarios     |
| **moto** (standalone server mode) | Python SDK stacks | In-process for Python, or as a server for multi-language projects   |

These tools run alongside Floci. Your test configuration routes specific services to the appropriate endpoint:

```yaml
services:
  floci:
    image: hectorvent/floci:latest
    ports: ["4566:4566"]
    volumes: ["/var/run/docker.sock:/var/run/docker.sock"]

  dynamodb-local:
    image: amazon/dynamodb-local:latest
    ports: ["8000:8000"]
    command: -jar DynamoDBLocal.jar -sharedDb -inMemory

  elasticmq:
    image: softwaremill/elasticmq-native:latest
    ports: ["9324:9324"]
```

---

## Layer 3: LocalStack Paid for IaC Workflow Validation

Floci doesn't yet have a CDK local wrapper or Terraform provider. Until it does, engineers actively developing CDK stacks or Terraform modules who run `cdklocal deploy` or `terraform apply` as part of their development loop need LocalStack for that specific workflow.

The key constraint to apply here: this doesn't mean the entire organisation needs a LocalStack paid licence. In most architectures, only engineers actively writing infrastructure code need this capability. Application engineers working against defined infrastructure do not.

A platform team of 5 engineers doing active IaC development: roughly $2,340/year for LocalStack paid. That's meaningfully different from licensing it organisation-wide for 50 engineers.

This layer is explicitly transitional. Floci's IaC integration is on the community roadmap. Review it annually — as it matures, the scope of this layer narrows and may eventually disappear. The hybrid posture is a 12–24 month transitional model, not a permanent architecture.

---

## Layer 4: Real AWS Sandbox Accounts for IAM Validation

Floci's IAM surface accepts all API calls. Permissions aren't evaluated. If your architecture mandates validation that least-privilege policies are correctly defined — that a service account genuinely cannot access a resource it shouldn't — that validation needs to happen against real AWS.

I'd treat this as a category decision rather than a tooling gap to engineer around. IAM policy evaluation in real AWS has subtleties — condition keys, service control policies, permission boundaries, resource-based policies — that no emulator fully replicates. Running those tests locally against something that doesn't enforce permissions isn't testing your IAM policies; it's testing whether your test code runs without errors.

Practical approach: provision dedicated AWS sandbox accounts per team or service domain with budget controls and automated cleanup. Run IAM boundary tests in CI using short-lived OIDC credentials (GitHub Actions OIDC, GitLab OIDC). The cost is AWS compute for test duration only — typically cents per pipeline run.

---

## The Architecture

```
Developer Laptop / CI Runner
│
├── Layer 1: Floci (port 4566)
│   ├── SQS, SNS, S3, DynamoDB, SSM, Secrets Manager
│   ├── Cognito, KMS, API Gateway, CloudWatch, EventBridge
│   ├── Lambda (real Docker container execution)
│   └── RDS, ElastiCache (real Docker containers)
│
├── Layer 2: Specialist Tools
│   ├── DynamoDB Local (port 8000) — fidelity-critical DynamoDB tests
│   ├── ElasticMQ (port 9324) — FIFO SQS edge cases
│   └── S3Mock (port 9090) — complex S3 scenarios
│
├── Layer 3: LocalStack Paid (IaC engineers only)
│   ├── cdklocal deploy workflows
│   ├── Terraform apply loops
│   └── Cloud Pods for infrastructure state snapshots
│
└── Layer 4: AWS Sandbox Accounts
    └── IAM boundary validation (OIDC credentials, automated cleanup)
```

---

## A Few Practical Notes on Running This

**Pin the Floci version.** Don't use `latest` in CI. Floci is moving fast — a patch release quietly changing a service's behaviour is a bad surprise on a Monday morning. Pin it, upgrade deliberately.

**Assign ownership per layer.** The platform team owns Layers 2, 3, and 4. Application teams own their Layer 1 Floci config. Without this being explicit, the hybrid setup slowly becomes ad-hoc as each team improvises their own variation.

**Revisit Layer 2 quarterly.** Floci is adding service coverage regularly. Some tools currently in Layer 2 will become redundant as Floci matures. It's worth checking every quarter and consolidating rather than accumulating tools nobody's actively maintaining.

**Write an ADR for this.** A hybrid multi-tool setup looks like unnecessary complexity without context. An Architecture Decision Record that captures the vendor risk reasoning, coverage gaps, and IaC constraints makes the decision legible to someone joining the team in six months — who would otherwise "simplify" it without knowing why it was built this way.

---

## The Deeper Point

The lesson of March 2026 isn't that LocalStack was the wrong choice — for a long time, it was exactly the right choice. The lesson is that embedding any single external tool this deeply in your development infrastructure, without a documented vendor risk assessment and a clear contingency, creates a category of platform risk that doesn't appear in service reliability metrics or security audits until it's already a problem.

The hybrid posture addresses that structurally. No single tool failure cascades across the whole environment.

---

## Series Summary So Far

**Part 1 — Why It Happened:** LocalStack's community edition sunset, the vendor risk pattern it represents, and why no fork emerged.

**Part 2 — The Decision Framework:** Technology stacks, service coverage, TCO at team scale, IaC toolchain fit, risk profiles, and the comparison matrix.

**Part 3 — The Hybrid Posture (this article):** The four-layer model for enterprise teams navigating Floci's coverage gaps while building toward a simpler, lower-cost posture over time.

Parts 4–6 shift to hands-on implementation: docker-compose configs, CI pipeline changes, service benchmarks, edge cases, and Testcontainers integration.

---

_How is your platform team approaching the hybrid posture? Are there coverage gaps or IaC workflow concerns not covered here? Worth discussing in the comments._

---

**Tags:** `#CloudArchitecture` `#PlatformEngineering` `#LocalStack` `#Floci` `#AWSStrategy` `#HybridCloud` `#OpenSource` `#TechLeadership` `#DevOps`

---

_About the author: Ashutosh Kumar is a Project Manager with 15 years of experience, currently exploring developer tooling, cloud workflows, and engineering process improvements. He writes at github.com/askuma._
