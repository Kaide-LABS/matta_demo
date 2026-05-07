# LATERAL PRD v3: Outlook Work Order Relay

## A. THE CONCEPT

**Outlook Work Order Relay** is a Microsoft 365-native lateral of the CMMS Bridge for automotive Tier-1 and enterprise manufacturing environments. Matta outbound webhooks create a structured work-order proposal, send it as an Outlook adaptive-card approval to the responsible maintenance manager, store supporting evidence metadata in SharePoint, and dispatch the approved result into SAP PM or IBM Maximo. The architectural delta is Microsoft 365 as the primary workflow surface: push ingress remains event-driven, but approval happens in Outlook rather than Slack, Teams chat, or a Matta-style dashboard.

## B. THE STRATEGIC HOOK

Matta's public market motion includes large industrial and automotive-adjacent environments. The intel records a manufacturing event with "BAE Systems, GKN Aerospace, McLaren Racing, Alpine Formula One Team, Cummins Inc., Bowers & Wilkins, Domino Printing Sciences" in the room [intel.md: Sentient Factories event post], and Doug's hiring post says Matta is deploying into "automotive and electronics to aerospace, nuclear submarines, waterproof coats and even gourmet cheese" [intel.md: Doug Brion LinkedIn, backend hiring post]. Microsoft 365 and Outlook are natural native surfaces in those enterprise environments.

This lateral also matches the "no-BS reality of building an AI company out of academia" that Doug described as research colliding with "real-world customers, urgency, and lots of failures" [intel.md: Doug Brion LinkedIn, Giant Ventures post]. Outlook Relay does not ask enterprise maintenance leaders to adopt a new dashboard; it drops the approval into the system where urgent plant emails already live.

The dossier's Pattinson analysis warns that "cyber-physical trust" is central and that "an error in an AI-controlled 5-axis CNC machine is a potentially fatal shrapnel event" [Dossier section 3]. Outlook Relay keeps the AI away from machine control and uses an approval-state machine plus SharePoint evidence ledger before any SAP PM/Maximo work order is created.

## C. THE AGENT ARCHITECTURE

All inference uses Vertex AI in `europe-west4` through `google-genai`. The sidecar consumes only Matta outbound webhooks. Microsoft Graph is used only for Outlook adaptive-card delivery and SharePoint evidence-link stubs; it does not access Matta internals.

Core schemas:

```python
class MattaDefectEvent(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)
    event_id: Annotated[str, Field(min_length=1, max_length=64)]
    agent: Literal["SENTRY", "GAUGE"]
    defect_class: str
    severity: Literal["low", "medium", "high", "critical"]
    timestamp: datetime
    line_id: str
    factory_id: str
    asset_id: str
    raw_context: Annotated[str, Field(max_length=2048)]

class RootCauseHypothesis(BaseModel):
    model_config = ConfigDict(extra="forbid")
    root_cause: Literal[
        "mechanical_wear", "tool_damage", "calibration_drift",
        "material_defect", "process_drift", "operator_error", "unknown"
    ]
    confidence_band: Literal["high", "medium", "low"]
    verifier_result: Literal["verified", "disagreed", "requires_human_review"]
    rationale: Annotated[str, Field(max_length=400)]

class CMMSWorkOrder(BaseModel):
    model_config = ConfigDict(extra="forbid")
    external_event_id: str
    outlook_approval_id: str
    sharepoint_evidence_url: str
    title: Annotated[str, Field(max_length=120)]
    description: Annotated[str, Field(max_length=1200)]
    urgency_tier: Literal["P1", "P2", "P3", "P4"]
    estimated_labor_minutes: Annotated[int, Field(ge=5, le=480)]
    required_parts: Annotated[list[str], Field(default_factory=list, max_length=10)]
    assigned_team: Literal["mechanical", "electrical", "quality", "maintenance_general"]
    requires_human_review: bool = False
```

Deterministic governance lives in `packages/governance/outlook_approval_state.py`. It validates Matta event shape, maps factory to SAP PM or Maximo, maps severity to urgency, validates Microsoft Graph signatures/callback IDs, enforces the approval state machine, and blocks dispatch unless the Outlook approver belongs to the configured maintenance-manager group. Gemini never decides routing, urgency, approval, or dispatch.

Pipeline pseudocode:

```python
def handle_webhook(raw_body):
    event = MattaDefectEvent.model_validate_json(raw_body)
    if idempotency.exists(event.event_id):
        return idempotency.cached_response(event.event_id)
    idempotency.reserve(event.event_id)
    queue.enqueue("build_outlook_proposal", event.event_id)
    return {"status": "accepted"}

def build_outlook_proposal(event_id):
    event = db.events.get(event_id)
    provider = rules.cmms_provider_for_factory(event.factory_id)  # sap_pm or maximo
    urgency = rules.urgency_from_severity(event.severity)

    classifier = gemini_flash_classify(event, schema=RootCauseHypothesis)
    verifier = gemini_flash_verify(event, classifier.root_cause, schema=VerifierDecision)
    if verifier.result != "verified" or classifier.root_cause == "unknown":
        classifier.verifier_result = "requires_human_review"
        requires_review = True
    else:
        classifier.verifier_result = "verified"
        requires_review = False

    proposal = gemini_pro_work_order(
        event=event,
        root_cause=classifier,
        fixed_provider=provider,
        fixed_urgency=urgency,
        allowed_parts=rules.parts_catalog(event.factory_id),
        schema=CMMSWorkOrder,
    )
    proposal = rules.enforce_admissible_work_order(proposal)
    evidence_url = sharepoint.create_evidence_stub(event, proposal, classifier, verifier)
    approval_id = outlook.send_adaptive_card(proposal, evidence_url, requires_review)
    db.proposals.insert(approval_id, proposal)

def approve_from_outlook(approval_id, user_id, action):
    rules.verify_graph_callback()
    rules.assert_approver(user_id)
    proposal = db.proposals.get(approval_id)
    rules.assert_transition(proposal.status, action)
    if action == "approve":
        cmms_id = adapter_for(proposal.provider).create_work_order(
            proposal.work_order,
            idempotency_key=proposal.external_event_id,
        )
        idempotency.commit(proposal.external_event_id, cmms_id)
```

Uncertainty uses a constrained enum classifier plus a two-stage verifier/arbiter. Stage 1 Gemini Flash classifies root cause into a finite enum with `unknown`. Stage 2 Gemini Flash receives the original event and proposed enum and must output `verified`, `disagreed`, or `requires_human_review`. Any disagreement, `unknown`, or low confidence forces manager review and disables one-click dispatch until the Outlook approver edits or confirms the proposal. This is a bounded disagreement-escalation pattern, not free-form LLM adjudication.

`CMMSAdapter` contract:

```python
class CMMSAdapter(Protocol):
    provider: Literal["sap_pm", "maximo"]
    def create_work_order(self, payload: CMMSWorkOrder, idempotency_key: str) -> str: ...
    def attach_url(self, work_order_id: str, label: str, url: str) -> None: ...
```

The idempotency key is the Matta `event_id`; Outlook callbacks include `approval_id` but cannot create duplicate CMMS work orders because adapters upsert by `external_event_id`.

## D. THE NATIVE ENVIRONMENT UI SPEC

The primary UI is Outlook, using Microsoft adaptive cards delivered through Microsoft Graph. The target customer is a Microsoft 365 automotive Tier-1 or large industrial account that already routes maintenance approvals through email and SAP PM/Maximo.

The card contains asset, line, severity, proposed work-order title, root-cause enum, verifier result, uncertainty state, required parts, estimated labor, and a SharePoint evidence link. Actions are Approve, Request Review, and Open CMMS. Approve creates the CMMS work order; Request Review marks `requires_human_review`; Open CMMS links to the created SAP PM/Maximo object when available. The UI must feel like native Outlook approval, not a Kaide or Matta dashboard.

## E. PHASE 1 EXECUTION SPEC

Repo deltas from Master PRD Section 6.1: keep `bridge_api`, `bridge_worker`, `schemas`, `adapters`, and `prompts`; add `packages/microsoft/graph_client.py`, `packages/governance/outlook_approval_state.py`, `apps/mocks/mock_graph_api`, and `apps/mocks/mock_sharepoint`.

FastAPI contracts:

```text
POST /webhooks/matta/defect -> accepts MattaDefectEvent, creates Outlook proposal
POST /outlook/actions/approve -> mock Microsoft Graph callback for Approve
POST /outlook/actions/review -> mock Microsoft Graph callback for Request Review
GET /sharepoint/evidence/{event_id} -> mock evidence metadata page
POST /cmms/{provider}/work-orders -> mock SAP PM/Maximo endpoint
```

Vertex prompt templates:

```text
Flash classifier prompt: classify into RootCauseHypothesis enum only, use unknown when insufficient evidence.
Flash verifier prompt: decide whether the proposed enum is verified, disagreed, or requires_human_review; output VerifierDecision JSON only.
Pro work-order prompt: generate CMMSWorkOrder JSON from fixed provider, fixed urgency, fixed SharePoint URL, fixed allowed parts catalog, and classifier/verifier result.
```

Mock data: create three automotive Tier-1 scenarios: casting porosity, bottling-line calibration drift, and high-severity press tooling wear. The headline demo uses tooling wear routed to SAP PM.

Demo flow: trigger the webhook, show an Outlook adaptive card arriving, open the SharePoint evidence stub, click Approve, and show SAP PM/Maximo receiving the work order with `external_event_id` and `sharepoint_evidence_url`.

Magic Moment: the manager approves an Outlook card and, without opening a Matta or Kaide surface, a structured SAP PM/Maximo work order appears with evidence attached.

## F. ANTI-REPLICATION VERIFICATION

Outlook Work Order Relay remains outside Matta's core IP and patent scope. Time scale is human-time approval in Outlook, normally minutes to hours, never millisecond-to-second control. The actuator is a maintenance manager approving an email card and SAP PM/Maximo receiving a work order, not a machine PLC, robot, firmware loop, extrusion parameter, or camera pipeline. The action space is discrete: approve, review, attach evidence, and create work order. The failure mode is a delayed or manually reviewed work order, not physical actuation that makes an out-of-spec part. The data flow is outbound Matta webhook to Microsoft 365 and CMMS; it never reaches into SENTRY, GAUGE, TALLY, TRACE, MFMs, Manufacturing OS UI, edge firmware, or real-time camera streams.
