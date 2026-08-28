00 — Master Architecture & Document Map

Restaurant AI Platform
Architecture & Governance Map

---

Document Control

Field| Definition
File| "00 Master Architecture & Document Map.md"
Role| Master architectural map and navigation document
Scope| Entire Restaurant AI Platform
Status| Foundational Architecture / Meta-Governance
Audience| Principal architects, AI engineers, prompt engineers, developers, QA, product, deployment, and auditors
Platform| Restaurant AI Platform
Author| Ramy Bella
Last Updated| 2026-08-16

---

1. Purpose

This document defines the overall architecture, organization, dependency structure, document hierarchy, and governance model of the Restaurant AI Platform.

It exists to ensure that the platform is developed as one coherent system rather than as a collection of disconnected documents, prompts, workflows, integrations, and implementation decisions.

This document answers:

- What are the major architectural phases?
- What belongs in each phase?
- Which documents are authoritative?
- Which documents are operational?
- Which documents are implementation-specific?
- How do the phases depend on one another?
- Where should new specifications belong?
- Which source takes precedence when documents appear to conflict?
- What belongs in Project Knowledge versus the actual implementation repository?
- How should the system evolve without creating architectural drift?

This document is an architectural map.

It is not itself a runtime system prompt and must not be interpreted by the deployed Assistant as a behavioral rule unless a downstream implementation explicitly incorporates an approved rule from an authoritative specification.

---

2. System Definition

The Restaurant AI Platform is a production-grade AI operating layer for restaurants.

Its purpose is to provide restaurants with an intelligent, reliable, safe, and scalable digital hospitality system capable of interacting with:

- guests,
- restaurant knowledge,
- restaurant operational systems,
- external service providers,
- communication channels,
- booking and ordering systems,
- internal staff workflows,
- testing and monitoring systems,
- deployment infrastructure,
- and continuous-improvement processes.

The platform is designed as a reusable product rather than a one-off chatbot.

Restaurant-specific information is therefore separated from the platform's foundational intelligence, architecture, governance, and operational logic.

---

3. Architectural Principle

The platform follows a layered architecture.

Each layer has a defined responsibility.

A lower-level or later implementation layer must not silently redefine a higher-level architectural contract.

The general dependency direction is:

Architecture & Governance
        ↓
Core Intelligence
        ↓
Knowledge
        ↓
Conversation
        ↓
Prompt / Orchestration
        ↓
Integrations
        ↓
Testing
        ↓
Deployment
        ↓
Continuous Improvement

Information and feedback may flow upward for analysis and improvement, but authority does not automatically flow upward.

---

4. Eight Major Platform Phases

The platform is organized into eight principal phases.

Phase 1 — Core Intelligence

Defines the fundamental intelligence, identity, constitutional behavior, reasoning boundaries, safety principles, authority model, and foundational AI behavior.

This is where the deepest behavioral contracts live.

Typical contents include:

- AI identity
- mission
- core values
- priorities
- limitations
- decision hierarchy
- constitutional governance
- foundational reasoning principles
- non-negotiable safety requirements
- behavioral boundaries

Phase 1 establishes the behavioral foundation that downstream phases must respect.

---

Phase 2 — Knowledge Base

Defines how the AI acquires, stores, structures, validates, retrieves, versions, and uses restaurant knowledge.

Typical contents include:

- restaurant configuration
- menus
- pricing
- opening hours
- policies
- dietary information
- allergen information
- locations
- services
- FAQs
- knowledge validation
- knowledge freshness
- source authority
- retrieval rules
- tenant isolation

Knowledge must not silently alter foundational AI behavior.

It supplies facts and configuration to the intelligence and conversation layers.

---

Phase 3 — Conversation Engine

Defines how the AI manages live conversations with guests.

Typical contents include:

- conversation lifecycle
- intent handling
- context management
- memory boundaries
- clarification
- escalation
- handoff
- multilingual behavior
- conversation state
- error recovery
- channel behavior
- guest experience logic

The Conversation Engine operationalizes the foundational behavior defined in Phase 1 using knowledge supplied by Phase 2.

---

Phase 4 — Prompt Engineering

Defines how the platform translates architecture, configuration, knowledge, and runtime state into model instructions.

Typical contents include:

- system prompts
- developer prompts
- contextual prompts
- prompt composition
- variable injection
- instruction hierarchy
- prompt templates
- model-specific adaptations
- prompt versioning
- prompt security
- prompt evaluation

Prompt Engineering must not become an unofficial source of constitutional authority.

A prompt implements approved system behavior; it does not redefine the system's governing principles.

---

Phase 5 — Integrations

Defines connections between the AI platform and external systems.

Typical contents include:

- reservation systems
- POS systems
- ordering systems
- CRM systems
- messaging channels
- SMS
- WhatsApp
- website interfaces
- APIs
- authentication
- webhooks
- external providers
- failure handling
- synchronization
- data mapping

External systems provide capabilities and data.

They do not automatically become independent sources of behavioral authority.

---

Phase 6 — Testing

Defines how the platform is validated before and after deployment.

Typical contents include:

- unit testing
- integration testing
- behavioral testing
- adversarial testing
- safety testing
- prompt testing
- regression testing
- hallucination testing
- knowledge accuracy testing
- escalation testing
- multilingual testing
- security testing
- performance testing
- acceptance criteria

Testing must validate compliance with the higher-level architecture rather than redefine it.

---

Phase 7 — Deployment

Defines how validated platform components become production systems.

Typical contents include:

- environments
- configuration
- tenant provisioning
- deployment procedures
- secrets
- monitoring
- logging
- observability
- rollback
- release management
- production safeguards
- incident handling
- operational ownership

Deployment changes how the approved system is operated, not what the foundational system is permitted to be.

---

Phase 8 — Continuous Improvement

Defines how the platform learns operationally from real-world usage without uncontrolled behavioral drift.

Typical contents include:

- analytics
- feedback
- failure analysis
- incident analysis
- conversation review
- performance monitoring
- knowledge-gap detection
- prompt optimization
- model evaluation
- regression prevention
- change management
- versioning
- improvement cycles

Continuous improvement must remain bounded by the foundational architecture and governance model.

Improvement must never mean silently weakening safety, honesty, authority boundaries, or other foundational requirements.

---

5. Master AI System Definition

The platform contains a dedicated foundational AI specification layer.

Its primary sequence is:

01 AI Identity
        ↓
02 AI Constitution
        ↓
03–10 Subsequent AI System Specifications

Within this Master AI System Definition:

01 AI Identity

"01 AI Identity.md" is the foundational behavioral source of truth.

It establishes the Assistant's:

- identity
- mission
- responsibilities
- core values
- priorities
- success criteria
- scope
- limitations
- customer-experience goals
- restaurant value
- foundational authority principle
- decision hierarchy

No subsequent Master AI System Definition document may weaken, contradict, or redefine these foundational requirements without a formally governed amendment to File 01.

02 AI Constitution

"02 AI Constitution.md" is the constitutional enforcement layer.

It operationalizes and formalizes the principles established by File 01.

It may introduce:

- enforcement mechanisms
- validation procedures
- severity classifications
- decision pipelines
- implementation constraints
- safety controls
- constitutional enforcement logic

It must not supersede the foundational authority of File 01.

Files 03–10

Files 03–10 operationalize the system within the boundaries established by Files 01 and 02.

They may define increasingly specific implementation behavior, but they must remain consistent with the higher-level contracts.

---

6. Authority Model

There are multiple kinds of authority in this project and they must never be conflated.

6.1 Architectural Authority

Defines which architectural layer governs another.

This document maps that relationship.

6.2 Behavioral Authority

Defines what the AI is fundamentally allowed and required to do.

This is established by the authoritative AI specification, beginning with "01 AI Identity.md".

6.3 Constitutional Authority

Defines how foundational behavioral requirements are formally enforced.

This is the role of "02 AI Constitution.md".

6.4 Operational Authority

Defines what a deployment, integration, restaurant configuration, or workflow is permitted to do.

Operational permissions must remain within the higher-level boundaries.

6.5 Implementation Authority

Defines how an approved requirement is technically implemented.

Implementation choices must not silently become new behavioral policy.

---

7. Source-of-Truth Rules

When determining where a rule belongs, use the following principle:

«The document closest to the architectural root owns the principle; downstream documents implement it.»

A downstream document must not duplicate a foundational rule merely to create a second source of truth unless the duplication is explicitly identified as an implementation reference.

Where duplication is necessary, the downstream document should reference the authoritative source.

The goal is:

One authoritative principle
        ↓
Many controlled implementations

Not:

One principle
        ↓
Eight independently maintained copies
        ↓
Eventual contradictions

---

8. Restaurant-Specific Configuration

Restaurant-specific information must remain separate from platform-level foundational architecture.

Examples include:

- restaurant name
- menu
- prices
- opening hours
- restaurant policies
- discount permissions
- location-specific information
- brand voice
- supported languages
- integrations
- operational rules

The platform architecture defines how such information is used.

Restaurant configuration defines what is true for a particular deployment.

These two concepts must not be confused.

---

9. Document Classification

Every new document should be classified before creation.

Possible classifications include:

A. Foundational

Defines principles that downstream systems inherit.

B. Constitutional

Defines enforcement of foundational principles.

C. Architectural

Defines system structure and dependencies.

D. Operational

Defines how a capability is operated.

E. Implementation

Defines how a requirement is technically implemented.

F. Configuration

Defines deployment-specific values.

G. Testing

Defines how behavior is validated.

H. Reference

Provides supporting information without becoming an authority source.

I. Template

Provides reusable structures for future documents.

A document should never acquire authority merely because it exists.

Authority must be explicitly defined.

---

10. New Document Placement Rule

Before creating a new document, answer:

1. What problem does this document solve?
2. Which phase owns that problem?
3. Is the content foundational, constitutional, architectural, operational, implementation-specific, configuration-specific, testing-related, or reference material?
4. Does another document already own this concept?
5. Is this creating a second source of truth?
6. Which documents depend on it?
7. Which documents is it allowed to influence?
8. Does it introduce a new authority level?
9. Does it introduce terminology that conflicts with an existing terminology system?
10. Does it need version control?

If these questions cannot be answered, the document should not yet be considered architecturally complete.

---

11. Cross-Phase Dependency Rule

Dependencies should generally move from foundational layers toward implementation layers.

A downstream phase may consume an upstream contract.

A downstream phase must not silently redefine an upstream contract.

For example:

Core Intelligence
      ↓
Knowledge
      ↓
Conversation
      ↓
Prompt Engineering
      ↓
Integrations
      ↓
Testing
      ↓
Deployment
      ↓
Continuous Improvement

Feedback may travel in the opposite direction:

Production
    ↓
Monitoring
    ↓
Testing / Analysis
    ↓
Improvement
    ↓
Proposed architectural change
    ↓
Governed approval
    ↓
Updated authoritative source

Production observations do not automatically become new rules.

---

12. Change Governance

Changes to the platform must be evaluated for architectural impact.

A change is considered foundational if it affects:

- identity
- mission
- core values
- safety boundaries
- authority
- decision hierarchy
- constitutional principles
- fundamental scope

Foundational changes require review of downstream dependencies before implementation.

A change is considered operational when it changes how an already-approved capability is executed without changing its underlying authority or principles.

A change is considered configuration when it changes deployment-specific values without changing platform behavior.

The distinction prevents routine restaurant configuration changes from becoming accidental platform architecture changes.

---

13. Versioning Principle

All authoritative documents must have explicit version information.

At minimum:

- document name
- version
- status
- author
- last updated date
- change history

A revision must identify whether it changes:

- architecture
- authority
- behavior
- implementation
- configuration
- testing requirements
- documentation only

Version numbers must not be treated as interchangeable across unrelated documents.

---

14. Consistency & Conflict Resolution

When reviewing the platform, consistency must be evaluated at multiple levels.

Terminological consistency

The same concept should not receive conflicting names without an explicit distinction.

Authority consistency

A downstream document must not claim authority that belongs to an upstream document.

Behavioral consistency

Operational behavior must remain compatible with foundational requirements.

Data consistency

Facts should have a defined source of truth.

Configuration consistency

Restaurant-specific configuration must not be hardcoded into foundational architecture.

Numerical consistency

Tier, priority, severity, risk, and other numbered systems must be explicitly scoped and must never be assumed equivalent merely because their numbers match.

Version consistency

Referenced documents must identify compatible versions where required.

---

15. Adversarial Architecture Review

The platform should be reviewed not only for whether documents make sense individually, but whether they remain coherent when combined.

Reviewers should actively search for:

- contradictory rules
- duplicated authority
- undefined terminology
- conflicting numbering systems
- circular dependencies
- hidden assumptions
- undocumented overrides
- impossible requirements
- unsafe fallbacks
- configuration overriding foundational behavior
- prompts overriding constitutional rules
- integrations becoming unintended authority sources
- testing requirements contradicting production requirements
- continuous-improvement processes causing behavioral drift
- stale references
- missing dependencies
- version mismatches

A document is not considered production-ready merely because it reads well in isolation.

---

16. Project Knowledge vs. System Source Files

Project Knowledge is a context layer used by AI assistants and development tools to understand the project.

It should contain the information required to reason about the project accurately.

The actual project repository remains the authoritative implementation environment.

Project Knowledge should therefore prioritize:

- architectural maps
- authoritative specifications
- current relevant documentation
- terminology
- interfaces
- important constraints
- project-wide conventions
- current implementation context

It should not be treated as an uncontrolled duplicate of every historical or obsolete file.

When multiple versions of a document exist, the current authoritative version must be clearly identifiable.

---

17. Obsidian as the Architectural Workspace

The platform architecture should preferably be maintained in Markdown/Obsidian because the system is fundamentally document- and relationship-driven.

Obsidian provides useful capabilities for:

- linking specifications
- navigating dependencies
- maintaining architecture maps
- versioning through Git
- connecting phases
- documenting relationships
- maintaining a single navigable knowledge graph

The architecture map should therefore preferably exist as:

"00 Master Architecture & Document Map.md"

A Word/PDF export may be produced when needed for external sharing, review, or archival purposes.

The exported document is not automatically the authoritative source.

---

18. Relationship Between 00 and the Eight Phases

"00 Master Architecture & Document Map.md" provides the map.

The eight phases provide the system structure.

The individual documents provide the detailed specifications.

The hierarchy is therefore:

00 — Master Architecture & Document Map
│
├── Phase 1 — Core Intelligence
│   └── Detailed specifications
│
├── Phase 2 — Knowledge Base
│   └── Detailed specifications
│
├── Phase 3 — Conversation Engine
│   └── Detailed specifications
│
├── Phase 4 — Prompt Engineering
│   └── Detailed specifications
│
├── Phase 5 — Integrations
│   └── Detailed specifications
│
├── Phase 6 — Testing
│   └── Detailed specifications
│
├── Phase 7 — Deployment
│   └── Detailed specifications
│
└── Phase 8 — Continuous Improvement
    └── Detailed specifications

"00" explains the structure.

It does not replace the detailed specifications.

---

19. What 00 Must Not Become

This document must not become:

- a second AI Constitution
- a second AI Identity
- a giant system prompt
- a duplicate of every phase
- a replacement for technical specifications
- a repository for restaurant-specific configuration
- an uncontrolled collection of implementation rules
- a hidden source of runtime authority

Its purpose is architectural clarity.

When a detailed rule belongs somewhere else, it should be placed there and referenced from this map.

---

20. Definition of Architectural Success

The architecture is successful when a qualified engineer can enter the project and determine, without relying on undocumented personal knowledge:

1. What the platform is.
2. What its major phases are.
3. Where a given requirement belongs.
4. Which document owns a given principle.
5. Which document is authoritative when conflicts occur.
6. What may be configured per restaurant.
7. What may never be configured away.
8. How phases depend on one another.
9. How changes are governed.
10. How the entire system can be audited.

The architecture should remain understandable even if the original architect is no longer available to explain it.

---

21. Architectural North Star

The Restaurant AI Platform should be built as:

«One coherent AI system with one clear foundational behavioral authority, explicit architectural layers, controlled configuration boundaries, auditable dependencies, and governed evolution.»

Every document, prompt, integration, test, deployment procedure, and improvement process should strengthen that coherence rather than fragment it.

---

22. Status

This document is a master architectural map.

It should be updated whenever the platform's architecture, phase structure, document hierarchy, or governance model materially changes.

It should not be modified merely to accommodate a local implementation shortcut.

Architecture should constrain implementation — implementation should not silently redefine architecture.