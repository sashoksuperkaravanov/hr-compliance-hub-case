# HR Compliance Hub

*[Русская версия](README.md)*

Analytical and architectural documentation for an integration service that eliminates data desynchronization between an internal HRM system and the government social insurance registry (SFR).

> Business Analyst / System Analyst training case study. The case company ("MetalInvestHolding") is fictional, but the business problem, regulatory basis, and architectural decisions are realistic and grounded in actual legislation (Federal Laws No. 27-FZ, 125-FZ, 152-FZ) — see [`docs/business-analysis/requirements.md`](docs/business-analysis/requirements.md).

## Business Context

Companies with 10,000+ employees are required to submit employment record data to the SFR using form EFS-1 no later than the business day following the HR event (Federal Law No. 27-FZ, Art. 11, Para. 2). The manual process has four systemic failure points — from unvalidated typos in the national insurance number (SNILS) to a complete lack of feedback on rejections, sometimes discovered weeks later. The failure points are broken down in the BPMN AS-IS diagram and summarized in [`docs/business-analysis/traceability-matrix.md`](docs/business-analysis/traceability-matrix.md) (the "AS-IS Failure Point" column).

![BPMN TO-BE: dismissal process](docs/process-models/bpmn/dismissal-to-be.png)

*TO-BE process — sources: [AS-IS](docs/process-models/bpmn/dismissal-as-is.bpmn) / [TO-BE](docs/process-models/bpmn/dismissal-to-be.bpmn).*

## Solution

A dedicated integration service (anti-corruption layer) sitting between the HRM system and the external circuit (EDI Operator → SFR), with automatic upfront validation, classification of failures as technical vs. content-related, and differentiated handling of each type (retry with fixed intervals vs. immediate escalation), plus real-time feedback via webhook.

## Capabilities

- Automatic ingestion of HR events from the HRM system (event-driven, via message bus)
- Data validation before dispatch to external systems
- Package assembly and dispatch to the EDI operator, digitally signed
- Retry policy: 3 attempts, intervals of 5 min / 30 min / 2 hours — technical failures only
- Immediate escalation to HR staff on content-related rejections (no pointless retries)
- Status log with RBAC access for the Compliance Officer
- Configurable retry policy for the IT Administrator

## Architectural Approach

A dedicated integration service pattern was chosen over extending the status model directly inside the legacy HRM — this isolates the HRM from the instability and change cadence of the external circuit (EDI operator, SFR) and allows retry/escalation logic to evolve independently. The architecture is decomposed using the C4 model (Context → Component), where each component maps 1:1 to the boundary of a specific use case (see [`docs/architecture/`](docs/architecture/)).

![Context Diagram (C4 L1)](docs/architecture/context-diagram-c4-l1.png)

## Artifacts

| Layer | Artifact |
|---|---|
| Business Processes | [BPMN AS-IS](docs/process-models/bpmn/dismissal-as-is.bpmn) ([PNG](docs/process-models/bpmn/dismissal-as-is.png)) / [BPMN TO-BE](docs/process-models/bpmn/dismissal-to-be.bpmn) ([PNG](docs/process-models/bpmn/dismissal-to-be.png)) |
| Functional Requirements | [Use Case Diagram](docs/process-models/use-case-diagram.puml) ([PNG](docs/process-models/use-case-diagram.png)) · [Use Case Specification](docs/business-analysis/use-case-specification.md) · [User Stories (UC8/UC9)](docs/business-analysis/user-stories.md) |
| Architecture | [Context Diagram (C4 L1)](docs/architecture/context-diagram-c4-l1.puml) ([PNG](docs/architecture/context-diagram-c4-l1.png)) · [Component Diagram (C4 L2)](docs/architecture/component-diagram-c4-l2.puml) ([PNG](docs/architecture/component-diagram-c4-l2.png)) |
| Behavior | [State Diagram](docs/process-models/state-diagram.puml) ([PNG](docs/process-models/state-diagram.png)) · [Sequence Diagram](docs/process-models/sequence-diagram.puml) ([PNG](docs/process-models/sequence-diagram.png)) |
| Data | [ER Diagram](docs/architecture/er-diagram.puml) ([PNG](docs/architecture/er-diagram.png)) · [Data Dictionary](docs/data/data-dictionary.md) |
| Requirements | [FR/NFR](docs/business-analysis/requirements.md) · [Traceability Matrix](docs/business-analysis/traceability-matrix.md) |
| API | [OpenAPI 3.1](api/openapi.yaml) · [Contract Rationale](api/api-contracts-overview.md) |
| Testing | [Test Scenarios (Gherkin)](docs/testing/test-scenarios.md) |

## Technologies and Notations

BPMN 2.0 · UML (Use Case / State / Sequence) · C4 Model · ER Modeling · OpenAPI 3.1 · Gherkin

Underlying technology context (at the level of detail an analyst needs to understand): event-driven architecture, Kafka, PostgreSQL, mTLS, JWT/RBAC, qualified electronic signature.

## Project Structure

```
docs/business-analysis/   — requirements, use cases, user stories, traceability
docs/process-models/      — BPMN, Use Case / State / Sequence diagrams (+ PNG)
docs/architecture/        — C4 Context / Component, ER (+ PNG)
docs/data/                — data dictionary
docs/testing/             — test scenarios
api/                      — OpenAPI contract and rationale
```

## Viewing the Diagrams

A PNG preview of every diagram already sits next to its source file and opens directly on GitHub with no extra tooling. Sources are only needed if you want to edit a diagram:

- **`.puml`** (PlantUML, C4/UML) — [PlantUML](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml) extension in VS Code, [plantuml.com/plantuml](https://www.plantuml.com/plantuml), or locally via `plantuml *.puml`.
- **`.bpmn`** (BPMN 2.0 XML) — [bpmn.io](https://bpmn.io/) in the browser (drag and drop the file) or Camunda Modeler.
- **`.yaml`** (OpenAPI) — [Swagger Editor](https://editor.swagger.io/) or the Swagger Viewer extension in VS Code.

## Results

- 12 functional and 10 non-functional requirements, fully traceable from business goals to test scenarios
- Cross-artifact consistency was checked and corrected wherever it had drifted — for example, unified document-status terminology (Use Case Spec ↔ State Diagram) and a single, consistent legal basis for the log retention period (NFR-02) across all documents.

## Limitations

This is an analytical artifact, not an implementation. Test scenarios are specified (Gherkin) but not automated or executed — no runnable system exists. Acknowledged technical debt and open questions (UC1/UC2/UC7 without a dedicated specification, some NFRs without test coverage) are recorded in [`docs/business-analysis/traceability-matrix.md`](docs/business-analysis/traceability-matrix.md) (Section 4).
