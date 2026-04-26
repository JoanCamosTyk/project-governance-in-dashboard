## What is this project trying to achieve? & What success looks like:

### Vision Statement
Tyk Governance transforms API governance from a fragmented, post-deployment concern into a proactive, continuous, and scalable process embedded directly within the development lifecycle. We enable organizations to establish universal governance standards that ensure API consistency, security, and compliance across their entire API ecosystem.

 

### Positioning
Unlike standalone governance tools that operate in isolation, Tyk Governance is an embedded governance solution that integrates directly into the Tyk Dashboard. We shift governance left in the development process while providing the flexibility teams need to innovate without bureaucratic friction.

### Market Position
Primary: The governance solution for organizations already using Tyk who need to scale API standards across teams

Secondary: Instead of managing separate tools like Spectral linters, custom CI/CD integrations, manual review processes, and disconnected reporting systems, Tyk Governance provides a unified experience where your existing governance rules work seamlessly within your API management workflow.

Differentiation: Native integration with Tyk Dashboard, design-first enforcement, and flexible rule application

### Strategic Direction Change
Tyk Governance use to be a PoC that was built as its own product outside of Tyk Dashboard. We decided to drop that project and itegrate Governance within the Tyk Dashboard product. 

### Value Proposition
Core Value Proposition: "Govern APIs at the speed of development, not the speed of compliance"

#### Key Value Drivers:

Shift-Left Governance: Catch violations during design, reducing rework by up to 60%

Embedded Experience: No context switching, governance lives where APIs are built

Flexible Enforcement: Guide without blocking, maintain development velocity

Universal Standards: Consistent policies across teams, environments, and API types

Measurable Improvement: Quantifiable API maturity scoring and compliance tracking

### Problem We Are Solving
Primary Problems:
Fragmented Governance Landscape

Organizations use disconnected tools (Spectral, custom scripts, manual reviews)

No single source of truth for API standards

Governance happens too late in the development cycle

Development Velocity vs. Compliance Tension

Teams avoid governance tools because they slow down delivery

Security and compliance violations discovered post-deployment

Expensive remediation cycles and technical debt accumulation

Inconsistent API Quality

Different teams follow different standards

Shadow APIs and duplicated functionality

Poor developer experience due to inconsistent API design

Lack of Visibility and Control

No centralized view of API compliance status

Difficulty tracking governance improvements over time

Manual processes that don't scale with API growth

### Pain Points by Persona:

#### Platform Engineers:

Struggle to enforce standards across multiple teams

Lack tooling to implement organization-wide governance policies

Manual governance processes that don't scale

#### API Developers:

Governance feedback comes too late in the process

Context switching between development and governance tools

Unclear or inconsistent governance requirements

#### Enterprise Architects:

Limited visibility into API compliance across the organization

Difficulty demonstrating governance ROI and improvements

Regulatory compliance requirements without proper tooling

 

### Target Audience
Existing Tyk Customers  looking to Govern their APIs(Enterprises)

Organizations with 10+ APIs in production

Multiple development teams

Existing governance pain points or compliance requirements

### Who's on the team?
PM: Joan
UI/UX and PM: Fiorella
Tech Lead: Tomas
Developers: 
BE: Sredny
FE: Jay


# Tyk API Governance — Product Vision
## Part 1: Executive Summary
Every enterprise has an API sprawl problem. What most don't have is the organizational infrastructure to govern it — and the tools they reach for make it worse.

Manual API review processes that take 2+ hours each, repeated 3–4 times a week, create exactly the bottleneck they're trying to prevent. Developers route around them. Undocumented APIs multiply. Compliance standards approved in architecture review boards never make it into runtime enforcement. The result: APIs that are theoretically compliant and actually exposed.

The problem isn't that organizations lack governance intent. It's that governance is too hard to do well. Review processes that require security, architecture, and design teams to manually inspect every API cannot scale to 500 APIs. They certainly can't scale to 5,000.

Tyk API Governance changes the equation. Instead of governance as a gatekeeping function that slows teams down, we're building governance as a continuous, automated process embedded in the workflows where APIs are actually created and deployed. Governance rules defined once by central teams. Enforcement happening automatically in CI/CD pipelines before a line reaches production. Compliance reported continuously — not discovered in audits.

For the teams who design and enforce governance standards: real-time visibility into their entire API portfolio's compliance posture, with the evidence needed to report to leadership and regulators. 
For the developers building APIs: fast, specific feedback that tells them not just what's wrong, but how to fix it — in their existing tools. 
For executives: APIs treated as strategic business assets, with the governance foundation that makes them reusable, secure, and AI-ready.

We're not building a compliance checkbox. We're building the operating system for organizations that take their API portfolios seriously.

## Part 2: Full Vision
Internal alignment document. Use to guide roadmap decisions, feature design, and technical architecture.

### The Problem We're Solving
#### Governance at Scale Is Broken
The most common governance process at enterprise API organizations today is a manual one: a developer submits an API for review, a committee of security, architecture, and design stakeholders examines it, and — if it passes — it's approved. This process is thorough. It is also fundamentally unscalable.

At RBFCU, each API review takes over 2 hours, with 3–4 reviews performed every week. That's 6–8 hours of governance overhead every week for a medium-sized organization. At Northwestern Mutual, a team of 3–4 people is responsible for governing over 500 APIs. At JP Morgan, the number exceeds 10,000. The math does not work.

The deeper problem is what happens when governance becomes a bottleneck. Developers route around it. APIs are deployed outside the gateway. Standards exist on paper but not in production. The manual process meant to prevent governance failures becomes the cause of them — creating a cycle we call the Shadow API feedback loop: gatekeeping governance → developer friction → undocumented APIs → governance sprawl → more gatekeeping required.

The principle that breaks this loop is simple: the compliant path must be easier than the workaround. Governance that adds friction will always be circumvented.

#### Visibility Without Insight Is Governance Theater
A second, equally critical problem is the illusion of governance in organizations that have adopted modern GitOps workflows. At Odido, a Dutch telecom managing 200–300+ APIs through Helm templates and GitOps, the team has strong process control — and still cannot answer basic governance questions:

"We just know because we control it via GitOps, but we don't really see or have the capability to analyze it." — Gregory Bohncke, Odido

This is common. Process control is not the same as compliance visibility. An organization can control every API deployment through version-controlled pipelines and still have no way to answer: How many APIs have weak authentication? Which ones handle PII without proper access controls? What percentage of our API portfolio is compliant with our security standards?

At FP&L (Florida Power & Light), APIs are distributed across Tyk, MuleSoft, Oracle, and AWS API Gateway with no unified inventory. At Move, conflicting API definitions across distributed teams caused production outages that should have been caught before deployment. At ACV Auctions: no API inventory exists.

Visibility requires more than deployment control. It requires a compliance lens on everything that's running.

#### The Design-to-Runtime Enforcement Gap
Even organizations with mature governance processes face a structural problem: policies approved at design time aren't enforced at runtime.

An API can pass an architecture review with compliant authentication requirements documented in its OpenAPI spec. That same API can be deployed with static API keys, wildcard permissions, and no HTTPS enforcement — because there is no automated step connecting the approved design to runtime configuration validation.

Northwestern Mutual named this gap explicitly in discovery. BYU found that APIs were accessible by bypassing the Tyk gateway entirely, rendering all authentication standards moot. FP&L identified static API keys, outdated authorization patterns, and missing HTTPS enforcement as live security risks in production APIs — all of which should have failed a governance check.

The gap between what is approved and what is running is where compliance fails.

#### Governance Is an Organizational Problem, Not Just a Tooling Problem
What makes API governance genuinely difficult is that it is not a single function. Large organizations have multiple teams with legitimate governance concerns — and those concerns don't overlap neatly.

A central compliance or security team cares about audit logs, access controls, regulatory reporting (PCI DSS, GDPR, SOC2, FERPA, SOX), and demonstrable security posture. They need evidence that every API in the organization meets baseline security standards.

An API design excellence team cares about something different: consistent naming conventions, documentation completeness, API versioning, breaking change management, and the developer experience offered to API consumers. Their job is to ensure the organization's API portfolio is coherent and professional, not just secure.

API developer teams need to understand the standards they're building to, get fast feedback when they violate them, and know specifically how to fix problems. They don't want to be responsible for governance decisions that should be made centrally — like which authentication method to use or what rate limiting defaults to apply.

Senior leadership needs a different view again: portfolio-level health, compliance trends over time, technical debt quantification, and evidence that API governance is generating business value rather than creating overhead.

Building for one of these audiences while ignoring the others produces partial solutions. The organizations that have tried this describe governance as a "bottleneck," a "queue that everyone is standing in." Getting it right means building for all of them simultaneously, with role-appropriate tools and views.

### Our Belief: Governance Must Be a Process, Not a Checkpoint
The dominant governance model in enterprises today treats governance as a checkpoint: APIs pass through a gate, get approved or rejected, and then governance steps back. This model fails at scale because it is reactive, manual, and concentrated in a bottleneck.

We believe governance should be continuous, embedded, and automated — a background operating system for the API lifecycle rather than a tollbooth on the deployment road.

This belief has practical implications for how we build:

Compliant = default. The governance tooling should make the right path the easy path. Pre-approved rule sets, API overlays that automatically apply required standards, templates that bake in security-by-default — governance compliance should be something that happens to APIs, not something developers have to manually achieve.

Feedback, not blocking. Governance systems that block deployments without explanation train developers to route around them. Effective governance gives specific, actionable feedback — not just "this failed" but "this rate limiting policy is missing; here's the recommended configuration." The SonarQube model, cited explicitly by Odido, is the right pattern: severity levels, configurable thresholds, technical debt scores, clear remediation paths.

Federated authority, centralized oversight. Large organizations cannot have one team governing all API decisions. Effective governance is federated: central teams define standards and rules, distributed teams own compliance within their domains, and a unified reporting layer gives the central governance function visibility across the entire organization. The governance platform must support this model, not assume a single governance authority.

Universal governance for a fragmented ecosystem. APIs are not homogeneous. REST APIs, GraphQL APIs, MCP APIs, and streaming APIs each have different governance requirements. They are deployed through different mechanisms — Tyk multi-gateway setups, Kubernetes operators, GitOps pipelines, direct API pushes. They may be internal-facing, external partner APIs, or public consumer APIs. A governance solution that works only for REST APIs on a single gateway is not enterprise-grade. Our platform must support governance that works universally across API types, deployment methods, and organizational structures — both design-first and code-first organizations.

### Who We're Building For
Three distinct governance personas, each with their own concerns, tools, and success criteria:

Iris — The Governance Owner
Head of Platform Governance, API Design Excellence Lead, or Compliance Engineering Lead.

Iris is responsible for defining governance standards and ensuring they are met across the organization. She may lead a compliance-focused team (concerned with security, access controls, regulatory evidence), a design-focused team (concerned with API quality and developer experience), or both. In large organizations, there may be multiple Irises with different domains.

Her core job: Create governance rules. Get visibility into compliance across the entire API portfolio. Report compliance posture to leadership. Demonstrate that standards are being met — or escalate where they aren't.

What she needs from us:

Tools to define and manage governance rule sets with configurable severity and thresholds

A real-time compliance portfolio showing the organization's API posture at a glance

Monthly compliance snapshots and trend views for leadership presentations

Audit trails that satisfy regulatory and internal audit requirements

Rule enforcement that happens without her team being in the loop for every API

Her current pain: Manual review processes that consume her team's time and create the bottleneck that undermines the entire governance program. No way to answer "are we compliant?" without running a manual audit.

Ada — The API Developer / Platform Engineer
Backend engineer, API team lead, or platform engineer deploying APIs through CI/CD pipelines.

Ada is building APIs. She cares about shipping fast and building well — in that order, if she's honest. Governance is a constraint on her velocity, and she has learned, through experience, where the compliance bottlenecks are so she can route around them when she needs to.

Her core job: Build and deploy APIs that work. Meet standards without being slowed down by governance processes she doesn't control.

What she needs from us:

Fast, specific feedback on governance violations in her existing workflow — CI/CD pipeline, PR review, IDE

Clear, actionable remediation: not just what failed, but how to fix it

Standards that are enforced, not just communicated — so she's not responsible for governance decisions that should be made centrally

API overlays and templates that apply required standards automatically, freeing her to focus on business logic

Her current pain: Governance as a black box that slows her down without clear feedback. Discovering compliance violations in production. Being responsible for security and design decisions that should be organizational standards.

Executive Sponsor
CTO, Head of Engineering, or CRO with API portfolio accountability.

The executive sponsor views APIs as business assets — or wants to. She understands that the organization's ability to build partner integrations, expose data to third parties, and power AI applications depends on the quality and governance of its API portfolio. She needs evidence that governance investment is generating returns.

Her core job: Make portfolio-level decisions about API strategy. Demonstrate to the board, regulators, and partners that API operations are professional and controlled.

What she needs from us:

Portfolio-level compliance dashboards that show health trends, not just point-in-time status

Quantified technical debt and risk posture

Evidence of governance ROI: reduced rework, faster time-to-market, audit-ready compliance records

The foundation for AI-ready APIs: governance as the operating system that enables AI agents to safely discover and consume organizational APIs

### The Critical Problems We Will Solve
In priority order, grounded in customer discovery:

#### — Must solve first:

Automate the governance bottleneck. Move governance from manual review into automated CI/CD validation. <30 second validation before any API reaches production. Governance should not require a human in the loop for every deployment.

Unify API visibility. Provide a single compliance portfolio across all API assets — regardless of how they're deployed, what type they are, or which team owns them. You cannot govern what you cannot see.

Close the design-to-runtime enforcement gap. Standards approved at design time must be enforceable at runtime. Gateway configuration governance (authentication enforcement, rate limiting, wildcard permissions, HTTPS requirements) must validate that deployed APIs match their approved design.

#### — High priority:

Enable federated governance. Central teams define rules. Distributed teams report compliance and remediate violations. Senior leadership gets a unified view. Support multiple governance stakeholders with role-appropriate tooling.

Provide compliance reporting. Executive dashboards showing portfolio health, compliance trends, and technical debt. Monthly snapshots suitable for leadership presentations and regulatory audit evidence.

Bring ungoverned organizations into governance. Many of our customers have no defined governance process. We must help them adopt one — not just give tools to those who already practice it. Pre-built rule sets, process templates, and a gradual adoption path (visibility-only → warn-only → enforcement) are essential.

#### — Medium priority:

Support universal API governance. REST, GraphQL, MCP, streaming APIs. Design-first and code-first organizations. Internal, external, and partner APIs. Governance that works across the full breadth of the API ecosystem.

### The Platform We're Building
Six capability areas, sequenced by the customer problems they solve:

#### Predefined Governance Rule Sets
Day-1 value delivery requires governance rules that are ready to use, not rules that must be authored from scratch. We provide:

Security baseline rules aligned to OWASP API Security Top 10: broken authentication, excessive data exposure, missing rate limiting, missing function-level authorization, security misconfiguration

API design standard rules: naming conventions, versioning, documentation completeness, breaking change detection

Tyk-native rules: x-tyk-* extension validation, gateway configuration validation, policy expiry, lifecycle gate enforcement, API key and subscription governance

Configurable quality gates: SonarQube-style thresholds with severity levels (critical / high / medium / warning), configurable pass/fail/staged review outcomes per environment, technical debt weighting with effort-to-fix estimates

The faster an organization gets its first governance report, the faster it sees the value of the platform. Pre-built rule sets reduce time-to-first-insight from weeks to hours.

#### Shift-Left CI/CD Enforcement
Governance must live where APIs are built, not where they're already deployed:

Jenkins plugin and GitHub Actions workflow: validate API specs and gateway configurations as a mandatory CI/CD pipeline step, before any deployment

Tyk Operator admission controller: Kubernetes-native validation webhook that intercepts API deployments and enforces governance rules before resources are admitted to the cluster

<30 seconds end-to-end: governance validation must not meaningfully slow the deployment pipeline

Inline feedback: violations surfaced directly in PR reviews and CI/CD logs with specific remediation guidance, not just pass/fail status

Configurable enforcement: block deployments on critical violations; warn on lower severity — with per-organization, per-environment configuration

#### Centralized Compliance Portfolio
The compliance portfolio is the single source of truth for organizational API governance posture:

Traffic light risk view (Red/Yellow/Green): immediate visual sense of portfolio health across all APIs

Compliance trends over time: is the organization getting better or worse? Where are violations concentrated?

Technical debt scores: quantify governance debt so it can be prioritized against product roadmap

Shadow API detection: identify zero-traffic managed APIs (Type 1 shadow APIs) that are consuming resources without delivering value; surface undocumented endpoints

#### Executive and Leadership Reporting
Different stakeholders need different lenses on the same compliance data:

Monthly compliance snapshots: management-ready slides showing portfolio trends, risk posture, and key wins. Designed for quarterly technical debt investment discussions and board-level risk reporting.

DORA-aligned benchmarking: connect governance outcomes to deployment frequency, change failure rate, and time-to-restore — the metrics engineering leadership already tracks

Regulatory audit trail: complete audit log of spec changes, governance state transitions, policy decisions, and violation history — structured for GDPR ROPA compliance, PCI DSS audit evidence, SOC2 reporting

#### Federated Governance Architecture
Enterprise governance is not a single team's responsibility. The platform must support:

Role-appropriate views: compliance dashboard for governance owners, developer-facing violation reports for API teams, executive portfolio view for leadership — same underlying data, different presentation and access controls

API overlay: centrally-defined governance standards auto-applied to new API specifications, freeing developers from making decisions that should be organizational standards (authentication method, rate limit defaults, required headers). Developers build; the platform ensures compliance.

Governance lifecycle: deployment → endorsement → reporting → retirement. Governance is not a one-time gate but a continuous feedback loop tied to the API's operational lifecycle. Telemetry-driven retirement: APIs with zero traffic for 90+ days flagged for deprecation review.

#### Ecosystem Integration
Governance must integrate into the tools and workflows that already exist:

Kubernetes: Tyk Operator admission webhooks intercept deployments in GitOps workflows. Governance enforcement happens at the cluster boundary, not as a separate process.

API design tools: Spectral rule set compatibility (existing rulesets can be imported and extended); integration with Stoplight and similar design-first tools so governance feedback is available at the spec authoring stage, not just at deployment

AI agent integration: governance rules exposed as machine-readable metadata (OpenAPI extensions, resource labels) so AI coding assistants — Claude, GitHub Copilot, OpenAI Codex — can reference governance standards when generating API specs and help developers get APIs right at authoring time, before any CI/CD step. This also enables AI agents consuming APIs to discover and verify compliance posture programmatically.

### Bringing Ungoverned Organizations Into Governance
A significant portion of our potential customers have no formal API governance process. They have a microservices guild, a Slack channel, and a wiki page with guidelines that nobody reads. Building tools that only serve organizations with mature governance programs leaves most of the market on the table.

Our onboarding approach for ungoverned organizations:

Start with visibility, not enforcement. The first deliverable is a compliance report on existing APIs — no blocking, no gates, just a clear picture of where the organization stands. This is the business case for investment in governance.

Provide process templates. Recommended governance models for different organizational structures: centralized (small governance team, all APIs), federated (domain teams own compliance, central team defines rules), and hybrid. Templates reduce the "we don't know where to start" barrier.

Activate rules progressively. Week 1: security baseline, warn-only. Week 4: security baseline, block on critical. Week 8: add design standards. Week 12: full enforcement with per-environment configuration. The gradual path lets teams develop governance muscle without organizational whiplash.

Measure adoption, not just compliance. "Time to first governance insight" and "time to first compliant deployment" are product metrics that matter as much as compliance rate. If organizations can't see value within hours of setup, the adoption funnel is broken.

### What Success Looks Like
12-month targets:

Metric

Baseline

Target

Manual API review time 

2+ hours per API 

<30 minutes (stretch: <5 min via automation) 

CI/CD governance coverage 

0% 

≥90% of deployments validated before production 

Production customers 

0 

≥3 enterprise customers with measurable compliance improvement 

Time-to-first-governance-insight 

Days/weeks 

<1 day for new organizations 

Governance team scalability 

~150 APIs/FTE 

500+ APIs/FTE 

Type 1 shadow API detection 

0% 

≥90% coverage 

The signal that we've succeeded: A governance owner at a large enterprise can answer "are we compliant?" in 30 seconds by looking at a dashboard — not by running a manual audit. An API developer gets feedback in their CI/CD pipeline, not from a review committee two weeks later. An executive can present API portfolio health to the board with the same confidence she presents financial metrics.

And the organizations that had no governance process at all? They have one now — and they got value from it in their first week.

### Connections to Broader Tyk Strategy
API governance is not a standalone product. It is the connective tissue between Tyk's gateway capabilities and the enterprise's need to manage APIs as strategic business assets.

Full-stack governance positioning: Tyk uniquely spans design-time governance (Spectral linting, spec validation), runtime policy enforcement (authentication, rate limiting, access control via the gateway), and AI-layer governance (LLM policy enforcement, MCP agent session management, tool-level RBAC in AI Studio). No competitor provides governance across all three planes. This is a defensible competitive position.

AI readiness as a governance outcome: Well-governed APIs — with complete documentation, consistent naming, enforced operationId fields, proper parameter metadata — are what AI agents need to be reliable consumers of organizational data. "Governance is the operating system for AI" is not a marketing claim; it is an engineering requirement. Organizations that invest in API governance now are building the foundation their AI strategy requires.

APIs as products: The shift from treating APIs as code to treating them as products — with ownership, lifecycle states, quality tiers (Bronze/Silver/Gold endorsement), and SLA governance — is where governance creates long-term business value. This framing resonates with executive sponsors and creates the narrative for governance investment.


## Governance Features & Concepts

Governance in Dashboard

Enables organizations to establish, enforce, and monitor governance rulesets across all their APIs within Tyk Dashboard.

### Characteristics

#### Universal Governance - Define and enforce consistent rulesets across OAS Apis & MCP.
Post 1st Release: Other Types

#### Shadow & Redudant APIs (Post 1st Release)
(Vacuum cannot determine if two APIs provide the same functionality or return the same data. This requires semantic analytics (comparing API schemas, endpoint structures, and data models)
Reduced Duplication - Identify redundant or shadow APIs across different departments, reducing maintenance costs and security risks. 

Redundant: APIs that provide the same functionality, returning basically the same data. 

Shadow APIs: APIs or endpoints that exist without visibility or governance. Outside of your governance/management control.

Type 1: “Rogue/Unmanaged APIs” (Completely Outside Control)
APIs that bypass all governance systems entirely
Characteristics:

 Not proxied through gateways

 Not documented anywhere

 No security policies applied

 Unknown to governance teams
Examples:
Developer deploys microservice directly to cloud
Internal service exposes HTTP endpoints without gateway
Third-party integrations bypassing API management
Kubernetes services with direct external access

Type 2: “Forgotten/Zombie APIs” (Managed but Neglected)
APIs that are properly registered/managed but abandoned
Characteristics:

 Proxied through gateways (Tyk, AWS..)

 Documented in API catalogs

 Zero or minimal traffic (unused)

 Not maintained or updated

 May have outdated security policies

Examples:

Old v1 API still running while everyone uses v2

Test APIs left in production

APIs for discontinued features

Legacy endpoints nobody remembers

Shadow APIs
Tyk Governance COULD Detect (Post 1st release)  Redundant & Shadow APIs Type 2)

✅ APIs registered in Tyk Dashboard with zero traffic
✅ APIs with outdated documentation
✅ APIs missing required governance policies
✅ APIs not following compliance rules
✅ Legacy APIs that should be retired

Tyk Governance CANNOT Detect (Harder) (Mainly Type 1):

❌ Services deployed outside gateway network
❌ Direct cloud service endpoints
❌ Microservices with external access
❌ APIs running on different infrastructure


 Documentation
 What is shadow API? - Product - Engineering - UX - Confluence
https://tyktech.atlassian.net/wiki/spaces/EN/pages/3879764021/Governance+-+Strategic+Direction+Change#Why-We-Changed.4 


#### Shift-Left Governance - Catch governance violations during design and development, not after deployment, reducing rework by up to 60%. Shift-left Approach
How do we encourage design-first without enforcing it? 
There are 3 ways to enforce this:
- Suggestion: Governance should not feel like a Gate; it should feel more like Helpful Feedback. We can’t block users.
We have that visual pressure of “Non-Compliant” while you are designing your API, and a Warning if you want to deploy, “This API does not fully meet the compliance requirements”. Governance Dashboard – Figma
- Block: This could perhaps be optional, but maybe some customers would like this functionality
- Overlays/Partials: ruleset tells you what is wrong with your API Def, you can apply an overlay instantly over it to fix the problems.

For Release 1: The idea is to have 2 flows, the customers can either have a message warning them if the APIs is non-compliant when hitting save, or not. 


- Old Flow (current flow) (Flow 1): Create API → Save → Live
- New Flow (Flow 2): Create API → Design State → Governance Check → Optional Approval → Deploy

Backward compatibility
Existing APIs in Prod will see changes if you apply a ruleset to an API Category, it might take some time, but it will not block them, they will just show as non compliant


Measurable API Maturity - Track and improve API quality with quantifiable metrics across technical excellence, business impact, and developer experience


Automated Compliance Checking: Continuous validation of APIs against organizational standards and regulatory requirements.


Maturity Assessment: Evaluation and scoring of APIs based on design, security, documentation, and performance criteria.


Centralized Reporting: Comprehensive visibility into API compliance and governance status.



### Governance Rulesets

A collection of Governance rules that can be applied to APIs to enforce best practices and compliance requirements. The requirements for API design, security, documentation, and operational characteristics.

Components of Rulesets:

Rules: Individual checks that validate specific aspects of an API definition.

Severity Levels: Categorization of rules by importance (violations, warnings).

Validation Functions: The specific logic used to evaluate API definitions against rules.

Remediation Guidance: Instructions on how to fix issues when rules are violated.



Understanding Ruleset Compliance in the Dashboard

When reviewing the compliance status of a ruleset applied to an API, two dimensions are important: “Violations vs Warnings” and Severity (Error, Warning, Info)



#### Violations vs Warnings

Violations

A violation occurs when a specific rule in a ruleset is not met.

Violations make the ruleset non-compliant. (You can deploy anyway; we never block deployments). A ruleset is non-compliant when it has an error. A violation is an error.

They represent critical issues that must be fixed to maintain security, operational integrity, or regulatory compliance.

Example: An API allows unauthenticated users to create orders → Violation, High priority.

Warnings

Warnings are advisory notifications that indicate potential risks or best practice improvements.

They do not affect the compliance status of the ruleset.

Warnings help teams plan improvements or anticipate problems before they become violations.

Example: API traffic is sent over HTTP → Warning, Low priority.

Summary: Violations are “must-fix” issues, while warnings are “good-to-fix” suggestions.



Integrating Governance validation into the API deployment pipeline, creating a validation checkpoint before APIs are actually deployed or updated by linting.

API Specification Linters (The engines that actually run the “Rulesets” to automatically check your OpenAPI specs for compliance, security, and design standards):

Tyk Governance uses Vacuum as the underlying engine to execute the rulesets (Optimized for OpenAPI and provides blazing-fast performance for evaluating rules). Solves Spectral’s performance issues on huge files. Vacuum is designed to be compatible with Spectral rulesets, meaning you can use the exact same industry-standard rules but run them much faster in your CI/CD pipelines. 

Spectral (Industry Standard): Generic JSON/YAML linter with broad capabilities. It can support:
- REST APIs: OAS written in JSON/YAML
- GraphQL APIs: These APIs are typically written in Schema Definition Language (SDL, .graphql files) rather than JSON/YAML. To lint GraphQL with Spectral-compatible rules, the GraphQL schema can be extracted and normalized into a structured JSON format (Abstract Syntax Tree, Intrsoepction Result). Once in JSON format, rules can be run against it using JSONPath.
- Event-Driven: Spectral has first-class, built-in support for AsyncAPI. Event-driven architectures can be linted similarly to REST APIs. (e.g webhooks)

Vacuum (The High-Performance Alternative): Current Gov PoC, the Ruleset Template we offer are:
- Vacuum-based OWASP (Focus on API Security Governance)
- Vacuum-recommended (Focus on API Quality & Design)
These are just templates we need to think on which good templates we want to give users. 
We are an API Management company we should have good knowledge into what good practices when building APIs are.
Vacuum is heavily optimized specifically for OpenAPI (REST, Webhooks, RPC)



#### Governance Engine & Ruleset Compatibility

Engine: Vacuum (Performance-Optimized)

✅ Vacuum executes the rules (faster than Spectral)
✅ Optimized specifically for OpenAPI validation
✅ Better performance for CI/CD pipelines

Ruleset Format: Spectral-Compatible

✅ Rules written in Spectral-compatible JSON/YAML format
✅ Uses JSONPath expressions (Spectral standard)
✅ Same rule structure as Spectral ecosystem

What This Means for Users

✅ Can Import Existing Spectral Rulesets

✅ Can Write Custom Spectral Rules



“Deployment Warning” configured per Ruleset to turn it on/off Governance Dashboard – Figma



### Governance Reporting 

Governance Dashboard – Figma

Refresh Report, Export Report

Compliance score %: Depends on the rulesets that pass your APIs

Total Number of APIs: Compliant & Non-compliant

Total compliance issues: High, Medium, Low risks

Meantime to resolve???

Rulesets compliance status: All rulesets with the number of APIs affected number of violations, warnings, and whether they are compliant.

Top offending APIs: APIs with the most Governance violations requiring immediate attention.



### API List

Governance Dashboard – Figma

Total no APIs

Total compliance issues

All APIs with their “Name, Type, Category, Compliance status, Owner, Segment tag, API Lifecycle, Date Actions.”





### API Category

This is what we use in the dashboard for filtering

Categorisation is very important as this is the way you will apply Rulesets and Overlays

(post 1st release): 

ODIDO mention the current API Category is very limiting, if this is the only way to categorise APIs. We should evolve the API Category. Create a more flexible way for users to categorize their APIs by for ex: Key Values.

Make it native to our dashboard. Categorisation relates a lot to API listing, Reporting, and maybe in the future to security policing (Applying current Security Policing in the dashboard to that Category). 





### Remediation Guidance

An API is non-compliant how does Tyk guide you to make the necessary changes? Do we give suggestions?

Yes, each rule within the ruleset will give you guidance in the Ruleset Definition on how to fix that specific API by giving you guidance per Ruleset.

From the individual API you click on view details? Governance Dashboard – Figma. 

From the API list, can you click on a specific API

From Rulesets Results “view issue info” Governance Dashboard – Figma

Dashboard Overview Page

Not 1st release.

We are not making changes here.

Governance Dashboard – Figma

Total Governance Score per Dashboard



### RBAC

Governance will be an Add-on Entitlement Feature. This means that Governance should be gated behind a license check. 

Add new “permissions”:
Governance [Read]: Allows viewing the Rulesets page, Report page, the new API Repository, and the Governance tab per API

Governance [Write]: Allows creating/editing Rulesets, running reports, viewing the new API Repository, and modifying governance configurations per API.

Governance [Deny]: Hides the Governance features entirely. Make it as if you don’t have the Add-on Entitlement Feature.



