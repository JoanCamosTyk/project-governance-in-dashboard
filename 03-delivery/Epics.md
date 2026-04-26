Delivery EPICS

1. Governance Ruleset Management (The Configuration)
Epic Name


Governance in Dashboard Ruleset Management (The Configuration)
Description

Goal
Establish the foundational backend & frontend capabilities to store, manage, and execute governance rulesets using the Vacuum engine within automated workflows & Tyk Dashboard.

Description
As a platform engineer, I need to create and customize governance rulesets to ensure our APIs meet organizational standards and industry best practices.

This epic will provide comprehensive capabilities for importing, creating and editing Spectral-compatible rulesets that define our API governance policies. This enables the enforcement of consistent API design, security, and compliance standards across our organization.

The system will support both pre-built industry-standard rulesets (Templates) and custom rulesets, allowing governance teams to tailor policies to their specific requirements.

This epic covers the "First Part" of the governance flow. It focuses on the Dashboard APIs and UI required to define what governance means for the organization (creating rulesets, using templates, and configuring severities).

 

Figma Designs: Governance Dashboard – Figma

Features
Ruleset Creation: Support creating from scratch, importing JSON/YAML files, and using templates.

Raw Definition Editor: A built-in code editor to view and modify the raw JSON/YAML definition of a ruleset.

Ruleset Configuration: Define Ruleset Name, Description, Status, Category, and Deployment Warning

Supported Specifications: OpenAPI Specification (OAS) and Model Context Protocols (MCPs)

Ruleset Templates: Provide real, out-of-the-box templates for OAS & MCP / Can also be customer configured templates

TBD:

Deployment Control: Configure a “Deployment Warning” toggle per ruleset to turn enforcement on/off

 

Stories
Backend Stories
“Vacum” fork + update

Integrate the Vacuum engine into the Tyk backend.

Implement the CRUD endpoints for rulesets

Support for Spectral-compatible JSON/YAML ruleset imports and creating a ruleset from scratch

Investigate and implement how the backend will parse MCP for ruleset configuration.

BE work for the Rulesets list to be able to filter by category

To do after the evaluation engine:

Provide out-of-the-box templates for OAS & MCP.

Frontend Stories
Ruleset Repository (Read): A centralized list view of all rulesets with their status, applied categories, and last updated dates. 

Create New Ruleset flow (Create): How to create ruleset flow.

Specific Ruleset Overview Page & Editor (Update, Delete)

Basic Information

Rules Overview

Configure Ruleset

Basic Config

Raw definition config

 

Lower Priority
Create a ruleset using “rule builder”

More supported API types

Questions
Is the Ruleset Repository (list) a Dashboard UI feature, or can you get this from CI/CD pipelines?. 

Stories from Epic 1:
TT- 17019 API for Create Rulesets

TT-17020 API for Read Rulesets

TT-17021 API for Update rulesets

TT-17022 API for Delete Rulesets

TT-17023 UI for API Ruleset listing

TT-17024 UI for API Ruleset Creation

TT-17025 UI for API Ruleset view, edit, delete

TT-17026 BE - Vacuum update

TT-17041 Ruleset Model & RBAC Setup

TT-17054 Investigate and implement how the backend will parse MCP for ruleset configuration


Epic 2: Ruleset Evaluation (The Execution)

Epic Name


Ruleset Evaluation (The Execution). Governance in Dashboard
Description

Goal
Provide the execution engine and CI/CD tooling to evaluate APIs against governance rulesets.

Description
This epic covers applying and evaluating rulesets. It focusses healivy on integrating the Vacuum engine and building the CI/CD bridge so developers can run their APIs against rulesets in UI and pipelines

 

Figma Designs: Governance Dashboard – Figma 

Features
Rule Execution Engine: Execute rulesets using the Vacuum engine

Ruleset Testing: A sandbox environment to test rulesets against API definitions and view execution results before saving. This needs to be configured in the Ruleset screen and in the API screen, both in the UI and CI/CD

Category-based Execution: Run rulesets against all APIs in a specific category in the UI and in CI/CD pipelines. Provide

Rule Execution & Feedback: Execute rulesets using the Vacuum engine. Provide violations, warnings.

Severity & Priority Mapping: Map rules to Severity (error/warn/info)

Remediation Guidance: Detailed issue views providing remediation guidance for specific rule violations

Define custom remediation guidance per rule within the ruleset definition

Display “Issue Details” with actionable guidance when a rule violation occurs

 

Backend Stories
Category-based Evaluation job: Async background job to evaluate a ruleset against all APIs within its assigned categories. Async job

Ability to test rulesets per API in the dashboard and in the CI/CD. An endpoint that can post an API spec and a rulesetId, to run Vacuum and return the JSON results.

Remediation Priority: Assign each rule a remediation priority. Configure this. 

Implement "Error/Warn/Info" severity. 

 

Front End Stories
Ruleset Test Sandbox: UI to select an API and run a ruleset against it, and view the results. Both in the Ruleset screen and Governance Tab per API section.

 

Open Questions and Doubts
We have rule severity (error/warn/info) when you configure a rule.

Performance Analysis when you run ruleset into APIs

The current idea is not to block customers on their deployments, just a warning. TBR after customer interviews. 
We could allow them to decide if they want to block or not.


Epic 3: Governance Ruleset Results & Visibility
Epic Name


Governance Ruleset Results & Visibility. Governace In Dashboard
Description

Goal
Surface governance evaluation results, track compliance, and enforce standards before deployment.

Description
This epic covers what you get from the evaluation. It handles receiving and storing linting feedback (whether generated externally by CI/CD pipelines or internally by Dashboard jobs), displaying them in the new Governance tab per API, and warning users if they try to deploy non-compliant APIs.

 

Figma Designs: Governance Dashboard – Figma

Features
Per-API Governance Status: View the overall compliance status (Compliant / Non-Compliant) for a specific API.

Applied Rulesets & Issues: List the rulesets evaluated against the API and detail the specific violations and warnings.

CI/CD Status Retrieval: API endpoints to retrieve the governance status and issues of an API

Issue Details: Configuration on how to fix (Remediation guidance) an API through the API goverannce tab and through the Test ruleset functionality in the Ruleset page.

 

Backend Stories
Endpoint to get the governance status and issues of a specific API. (Complaint/Non-Compliant)

Endpoint to automatically send the final linting feedback to the Dashboard after API evaluation

BE logic to map rule violations/warnings to their specific “How to fix”

Any other work needed for the new Governance tab per API section [UI Warnings].

 

Frontend Stories
Implement the new “Governance” tab in the API configuration view, displaying the compliance status, rulesets, and detailed issues

Display Issue Details, Affected area, and “How to fix” (Remediation Guidance) sorted by Remediation Priority

UI warnings when saving or changing an API to active if it's non-compliant

Show an “In progress” indicator when a ruleset is currently running on an API

Open Questions & Doubts
How do we calculate the compliance score? 

What makes an API Non-Compliant?. An error [A rule is not met] makes APIs non-compliant.

How does the “Warning” visual date happen in CI/CD to have that extra step before you move an API to active if non-compliant?

 

Further Ideas
Customers (NorthWestern Mutual & Barclays) would like RBAC to be expanded. A read-only shared link of the dashboard for Federated teams. Something like a snapshot you can share.