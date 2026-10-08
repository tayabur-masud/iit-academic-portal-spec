<!--
Sync Impact Report
Version change: 1.0.0 -> 1.1.0
Modified principles: none
Modified sections: Technical and Data Constraints (approved versions and major-version approval rule)
Added sections: none
Removed sections: none
Follow-up TODOs: confirm the original ratification date.
-->
# IIT Academic Portal Constitution

## Core Principles

### I. IIT Visual Identity and Academic Usability
All user-facing modules MUST use the IIT visual identity and a consistent design system. Brand colors MUST derive from the IIT logo palette and be defined as centrally maintained design tokens; components MUST NOT introduce arbitrary brand colors or repeat hard-coded color values. UI elements and interaction patterns MUST reuse established components where practical, remain accessible and responsive for supported screen sizes, and present clear loading, empty, success, warning, validation, and error states. Destructive or irreversible actions MUST require appropriate confirmation. Academic workflows MUST use clear IIT terminology, minimize avoidable steps, expose meaningful workflow states, and provide feedback after important actions. Detailed colors, typography, spacing, and component specifications belong in a dedicated design-system specification, not duplicated in feature specifications. These rules keep the experience professional and understandable across roles.

### II. Maintainable Layered Architecture
The system MUST remain a layered monolith with clear separation among presentation/API, application/business logic, domain concerns, and infrastructure/data access. Controllers and endpoints MUST remain thin; business rules MUST NOT live in UI components, controllers, or database-access code. Dependencies MUST follow a clear, controlled direction, and infrastructure details MUST NOT unnecessarily leak into business logic or API contracts. Code MUST use meaningful names and focused responsibilities, follow existing conventions, avoid needless duplication, and apply SOLID principles or patterns only when they provide practical value. Prefer the simplest understandable solution that satisfies the approved requirements; avoid premature abstraction and over-engineering.

### III. RESTful API Contracts
Backend APIs MUST use resource-oriented URLs, semantically appropriate HTTP methods and status codes, and stateless interactions. Request and response conventions MUST remain consistent. APIs MUST validate incoming data at the boundary, use DTOs or explicit contracts where appropriate rather than exposing persistence entities, and return consistent, meaningful errors without internal implementation details. Collection resources MUST support pagination, filtering, sorting, or searching when appropriate. Use resource-oriented REST designs instead of RPC-style endpoints when they fit the operation. API changes MUST consider backward compatibility and document material contract changes.

### IV. Security and Data Integrity
Security MUST be designed into every feature. Authentication and authorization MUST be enforced at appropriate server boundaries, with least privilege; hiding a control in the UI MUST NOT substitute for server authorization. All user-controlled input and business rules MUST be validated on the server. Secrets and credentials MUST NOT be hard-coded or exposed in responses, logs, exceptions, or client code. Database access MUST preserve appropriate relationships, constraints, indexes, and normalization, retrieve only needed data, avoid N+1 and unnecessary queries, and use asynchronous operations where appropriate. Database migrations MUST be version controlled. Sensitive operations and security-relevant workflows MUST receive suitable automated tests.

### V. Reliable, Simple, and Observable Software
Business-critical rules MUST have appropriate automated tests, including unit tests where practical and integration/API tests for important workflows. Authentication and authorization behavior MUST be tested for security-sensitive functionality. Tests MUST be readable, deterministic, isolated, and focused on meaningful behavior; existing tests MUST continue to pass unless an approved behavior change requires updates. Expected business errors MUST be handled explicitly, and centralized error handling MUST prevent stack traces, SQL, and infrastructure details from reaching users. Important events and failures MUST use meaningful structured logs with useful diagnostic context and appropriate levels; sensitive data MUST never be logged. Implementations MUST avoid unnecessary calls, processing, rendering, and data loading, and MUST paginate potentially large collections. Add performance complexity only for a reasonable requirement. Documentation MUST explain important business rules, API contracts, architectural decisions, and non-obvious behavior, and remain accurate.

## Technical and Data Constraints

The approved technology stack is:

- Frontend: Angular 21, TypeScript, HTML, and CSS.
- Backend: ASP.NET Core 10 Web API with C# 14.
- ORM: Entity Framework Core 10.
- Authentication and authorization: ASP.NET Core Identity 10.
- Database: PostgreSQL 18 with pgvector for vector capability.
- Frontend runtime and tooling: Node.js 24 LTS.

The architecture is a layered monolith with RESTful APIs. Specifications, plans, and implementations MUST use this stack and existing project libraries. Major framework versions MUST NOT be changed without explicit project-level approval. New frameworks, packages, dependencies, or architectural patterns require a clear benefit; dependencies MUST remain minimal and justified. Database and infrastructure implementation details MUST stay behind appropriate application boundaries.

## Delivery, Quality, and Change Control

Specifications, plans, tasks, implementations, reviews, and AI-generated code MUST conform to this constitution. Implement only approved requirements; do not add speculative behavior or perform unrelated refactoring during feature work. When a requirement is ambiguous, record the ambiguity and resolve it before inventing behavior. Fix root causes, and document any knowingly accepted technical debt and its rationale. AI-generated output MUST be reviewed and understood before acceptance and MUST NOT justify unnecessary complexity, duplication, inconsistent UI, insecure design, non-RESTful APIs, or unjustified dependencies.

When principles conflict, decisions MUST prioritize, in order: correctness; security; user experience and usability; maintainability; consistency; testability; performance; simplicity and development efficiency. Document material trade-offs and explain why the higher-priority principle governs. Reviewers MUST evaluate compliance at specification, planning, task, and code-review stages; a feature is incomplete when it works but fails applicable constitution requirements.

## Governance

This constitution is the governing standard for project specifications, plans, tasks, implementation, and review. Lower-level artifacts MUST comply; a conflict MUST be resolved by amending the conflicting artifact or explicitly amending this constitution, not by silently disregarding a principle. Amendments require a documented rationale, impact assessment, and review of affected specifications, plans, tasks, tests, and design-system documentation before adoption. Reviewers MUST verify compliance and require remediation or a documented, approved amendment for violations.

Constitution versions use semantic versioning. Increment MAJOR for backward-incompatible principle removals or redefinitions, MINOR for new principles or materially expanded governance, and PATCH for clarifications or non-semantic edits. Every amendment MUST update the version and last-amended date; the ratification date remains the date of original adoption. If an original date cannot be verified, retain an explicit TODO until confirmed.

**Version**: 1.1.0 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date unknown | **Last Amended**: 2026-10-08
