# LATERAL PRD v2: Planner Triage Console

## A. THE CONCEPT

**Planner Triage Console** is a maintenance-manager-first lateral of the CMMS Bridge. Rather than processing each Matta defect as an immediate push notification, it pulls or receives outbound Matta events into 15-minute planning batches, clusters related defects into maintenance candidates, and presents them inside the customer's existing CMMS planning board or Microsoft Teams maintenance tab for approval. The architectural delta is batch pull plus manager triage: asynchronous, manager-governed dispatch into SAP PM or IBM Maximo instead of technician-first mobile acknowledgement.

## B. THE STRATEGIC HOOK

This lateral exists for factories where the person who owns maintenance sequencing is the planner, not the floor technician. Doug's public framing includes "we're deploying to around two factories a month and have a multi-year waitlist" [intel.md: Doug Brion LinkedIn, funding post], which implies many plants with incompatible planning cultures. A triage console avoids forcing one immediate-response workflow onto every facility.

It also fits Matta's customer set and industrial breadth. The intel records Matta running across "polymer plants, casting lines, bottling halls" [intel.md: Matta repost, founder activity] and mentions events including BAE Systems, GKN Aerospace, McLaren Racing, Cummins, Bowers & Wilkins, Domino Printing Sciences, ABB, and others [intel.md: Sentient Factories event post]. Those environments often have SAP PM or Maximo planners who batch maintenance windows rather than dispatch every alert instantly.

The dossier describes Damjan Denic as requiring "strict adherence to schema cleanliness and idempotency" [Dossier section 4] and states his hierarchy is "reliability first, latency second, feature completeness third" [Dossier section 5]. A 15-minute batch workflow honors that preference: it trades seconds of latency for durable state, deduplication, and planner control.

## C. THE AGENT ARCHITECTURE

All inference uses Gemini through Vertex AI `europe-west4` with `google-genai`. The system consumes only Matta's public outbound REST API or webhook export stream. It does not call SENTRY, GAUGE, TALLY, TRACE, MFMs, Manufacturing OS UI, edge firmware, or cameras.

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
    conformal_set: list[str]
    rationale: Annotated[str, Field(max_length=400)]

class CMMSWorkOrder(BaseModel):
    model_config = ConfigDict(extra="forbid")
    external_event_id: str
    batch_id: str
    title: Annotated[str, Field(max_length=120)]
    description: Annotated[str, Field(max_length=1200)]
    urgency_tier: Literal["P1", "P2", "P3", "P4"]
    estimated_labor_minutes: Annotated[int, Field(ge=5, le=960)]
    required_parts: Annotated[list[str], Field(default_factory=list, max_length=12)]
    assigned_team: Literal["mechanical", "electrical", "quality", "maintenance_general"]
    planned_window: datetime | None = None
    requires_human_review: bool = False
```

Deterministic governance is located in `packages/governance/planner_rules.py`. It owns REST polling windows, idempotency, deduplication, CMMS provider selection, severity-to-priority mapping, and manager approval state. The LLM does not decide whether a batch becomes a work order. It generates bounded candidate narratives and categorical hypotheses; the planner approval state machine decides final dispatch.

Pipeline pseudocode:

```python
def pull_matta_events(window_start, window_end):
    response = matta_public_api.get_defects(window_start, window_end)
    events = [MattaDefectEvent.model_validate(e) for e in response.items]
    for event in events:
        if not idempotency.exists(event.event_id):
            db.events.insert(event)
            idempotency.reserve(event.event_id)
    queue.enqueue("build_triage_batch", window_start, window_end)

def build_triage_batch(window_start, window_end):
    events = db.events.for_window(window_start, window_end)
    candidates = deterministic_cluster(events, keys=["factory_id", "line_id", "asset_id", "defect_class"])
    for candidate in candidates:
        scores = gemini_flash_triage_scores(candidate.events, schema=TriageScoreVector)
        # Conformal-style calibration uses a fixed held-out demo calibration table.
        prediction_set = conformal_prediction_set(scores, alpha=0.1, calibration_table=CALIBRATION_SCORES)
        requires_review = len(prediction_set) != 1 or "unknown" in prediction_set
        hypothesis = RootCauseHypothesis(
            root_cause=single_or_unknown(prediction_set),
            conformal_set=prediction_set,
            confidence_band=band_from_set_size(prediction_set),
            rationale="Conformal prediction set derived from calibrated batch triage scores.",
        )
        draft = gemini_pro_batch_work_order(candidate, hypothesis, schema=CMMSWorkOrder)
        draft.requires_human_review = requires_review
        draft = rules.enforce_manager_review_required(draft)
        db.triage_rows.insert(draft)

def approve_triage_row(row_id, manager_id):
    row = db.triage_rows.get(row_id)
    rules.assert_manager_can_approve(manager_id, row.factory_id)
    rules.assert_transition(row.status, "approved")
    payload = rules.enforce_admissible_work_order(row.cmms_work_order)
    cmms_id = adapter_for(row.factory_id).create_work_order(payload, idempotency_key=row.external_event_id)
    idempotency.commit(row.external_event_id, cmms_id)
```

Uncertainty uses a conformal-prediction-style confidence set over categorical root-cause outputs. Gemini Flash produces bounded category scores; a deterministic conformal wrapper uses a fixed calibration table from historical/mock labeled events to produce a prediction set at alpha `0.1`. If the set has more than one root cause, includes `unknown`, or crosses a severity override, the row is marked `requires_human_review`. This surfaces uncertainty as an explicit set-valued prediction, not a similarity score.

`CMMSAdapter` contract:

```python
class CMMSAdapter(Protocol):
    provider: Literal["sap_pm", "maximo"]
    def create_work_order(self, payload: CMMSWorkOrder, idempotency_key: str) -> str: ...
    def attach_batch_evidence(self, work_order_id: str, event_ids: list[str]) -> None: ...
```

The idempotency key is `event_id` for single-event candidates and `sha256(sorted(event_ids))` for clustered candidates. The CMMS external reference stores both batch ID and constituent Matta event IDs.

## D. THE NATIVE ENVIRONMENT UI SPEC

The primary UI lives where maintenance managers already plan work: SAP PM/Maximo planning board in production, or a Microsoft Teams tab embedding the same queue for demo. It must not look like Matta's dashboard. It is a triage table with rows grouped by asset, planned window, severity, conformal prediction set, required parts, and recommended maintenance team.

Manager actions are Approve, Merge, Split, Send to Review, and Open in CMMS. Approve is the only action that can create the CMMS work order, and it is backed by a deterministic approval-state machine. The UI must show uncertainty as a prediction set, for example `{mechanical_wear, calibration_drift}`, not as a pseudo-precise percentage. Rows with multi-label sets default to review.

## E. PHASE 1 EXECUTION SPEC

Repo deltas from Master PRD Section 6.1: keep `bridge_api`, `bridge_worker`, `schemas`, `adapters`, and `prompts`; replace `theater_ui` with `apps/planner_triage_ui`; add `packages/governance/planner_rules.py`; add `packages/uncertainty/conformal.py`; implement mock SAP PM and Maximo adapters.

FastAPI and worker contracts:

```text
POST /jobs/pull-matta-events -> starts a 15-minute Matta REST pull job
GET /triage/batches/{batch_id} -> returns grouped triage rows
POST /triage/rows/{row_id}/approve -> creates SAP PM/Maximo work order
POST /triage/rows/{row_id}/review -> marks requires_human_review
POST /triage/rows/{row_id}/merge -> deterministic merge of selected event IDs
POST /cmms/{provider}/work-orders -> mock SAP PM/Maximo create endpoint
```

Vertex prompt templates:

```text
Flash triage prompt: classify each grouped event cluster into allowed root_cause enum scores; output TriageScoreVector JSON only.
Pro work-order prompt: produce CMMSWorkOrder narrative from fixed urgency, fixed provider, fixed conformal prediction set, fixed parts catalog, and fixed planned window if provided.
```

Mock data: generate four 15-minute batches across two factories. Include repeated SENTRY defects on one asset and GAUGE calibration events on another so the clustering and conformal set behavior is visible.

Demo flow: run `/jobs/pull-matta-events`, open Teams tab or CMMS planning mock, select a row with a single-label conformal set, approve it, and show SAP PM/Maximo receiving the work order with batch evidence attached.

Magic Moment: a maintenance manager sees a noisy batch of Matta-detected defects collapse into a typed, reviewable planning row and one approval creates a structured SAP PM/Maximo work order.

## F. ANTI-REPLICATION VERIFICATION

Planner Triage Console is outside Matta's core IP and patent boundary. Time scale is explicitly minutes to hours because events are pulled and batched every 15 minutes and require manager approval. The actuator is the maintenance manager and SAP PM/Maximo work-order system, not a PLC, firmware loop, robot, extruder, or machine parameter. The action space is discrete: approve, merge, split, review, and create a work order. The failure mode is a delayed or escalated maintenance planning row, not automated production of an out-of-spec part. The data flow is outbound from Matta's public REST/webhook interface into customer enterprise planning systems; it never reaches into SENTRY, GAUGE, TALLY, TRACE, MFMs, Manufacturing OS UI, edge firmware, or live camera streams.
