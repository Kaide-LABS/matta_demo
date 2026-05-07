# LATERAL PRD v4: Airgap Courier

## A. THE CONCEPT

**Airgap Courier** is an offline-tolerant lateral of the CMMS Bridge for industrial networks that cannot rely on continuous cloud or CMMS connectivity. Instead of assuming always-on webhook delivery, a customer-side courier service ingests Matta outbound REST exports or webhook spool files, validates and queues them locally, runs Gemini inference only when Vertex AI `europe-west4` is reachable, and dispatches the resulting Maximo or SAP PM work orders when the CMMS connection returns. The architectural delta is durable asynchronous pull/spool processing with an offline-first queue and a native CMMS admin status view.

## B. THE STRATEGIC HOOK

This lateral matches the physical-network reality in the dossier. Denic's operational reality includes local factory instances that can operate "completely disconnected from the cloud (air-gapped or intermittent connection) while still synchronizing state asynchronously when a connection is restored" [Dossier section 3]. Airgap Courier is designed for that exact deployment context.

The same dossier says Denic judges systems by "strict adherence to schema cleanliness and idempotency" [Dossier section 4], and that factory networks drop packets because of heavy machinery interference [Dossier section 4]. Airgap Courier makes durable queue state the product center rather than a footnote.

Doug's hiring post says Matta is installing its "factory nervous system across UK, EU, and US factories in EVERY sector" [intel.md: Doug Brion LinkedIn, FDE hiring post]. Those factories include old plants, noisy networks, and strict OT/IT boundaries. The Courier keeps the CMMS loop working without asking Matta FDEs to hand-code one-off integration retry scripts for each site.

## C. THE AGENT ARCHITECTURE

All inference uses Gemini through Vertex AI in `europe-west4` via `google-genai`, but inference is deferred when the Vertex endpoint is unreachable. The system consumes only Matta's public REST export endpoint or customer-configured outbound webhook spool directory. It never reads Matta internals, SENTRY/GAUGE camera streams, edge firmware, MFMs, or the Manufacturing OS UI.

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
    cascade_stage: Literal["rules", "flash_classifier", "pro_review", "human_review"]
    rationale: Annotated[str, Field(max_length=400)]

class CMMSWorkOrder(BaseModel):
    model_config = ConfigDict(extra="forbid")
    external_event_id: str
    courier_sequence_id: int
    title: Annotated[str, Field(max_length=120)]
    description: Annotated[str, Field(max_length=1200)]
    urgency_tier: Literal["P1", "P2", "P3", "P4"]
    estimated_labor_minutes: Annotated[int, Field(ge=5, le=480)]
    required_parts: Annotated[list[str], Field(default_factory=list, max_length=10)]
    assigned_team: Literal["mechanical", "electrical", "quality", "maintenance_general"]
    offline_state: Literal["queued", "inferred", "pending_cmms", "dispatched", "requires_human_review"]
    requires_human_review: bool = False
```

Deterministic governance lives in `packages/governance/airgap_rules.py` and `packages/queue/durable_outbox.py`. It validates file signatures/checksums, event schemas, sequence ordering, idempotency, urgency mapping, allowed CMMS providers, retry limits, and offline state transitions. The LLM is never allowed to decide whether to dispatch during an outage, change urgency, or bypass human review.

Pipeline pseudocode:

```python
def ingest_spool_file(path):
    raw = read_once(path)
    rules.verify_spool_checksum(raw)
    event = MattaDefectEvent.model_validate_json(raw)
    if outbox.exists(event.event_id):
        return "duplicate_ignored"
    outbox.insert(
        event_id=event.event_id,
        payload=event.model_dump_json(),
        state="queued",
        sequence_id=next_sequence(),
    )

def process_outbox():
    for item in outbox.pending():
        event = MattaDefectEvent.model_validate_json(item.payload)
        urgency = rules.urgency_from_severity(event.severity)
        root = deterministic_rules_classify(event)
        if root.confidence_band == "high":
            hypothesis = root
        elif vertex_ai.reachable():
            flash = gemini_flash_classify(event, schema=RootCauseHypothesis)
            if flash.confidence_band == "high" and flash.root_cause != "unknown":
                hypothesis = flash
            else:
                pro = gemini_pro_review(event, flash, schema=RootCauseHypothesis)
                hypothesis = rules.apply_abstention_thresholds(pro)
        else:
            outbox.defer(item.event_id, state="queued", reason="vertex_unreachable")
            continue

        if hypothesis.root_cause == "unknown" or hypothesis.confidence_band == "low":
            outbox.mark(item.event_id, state="requires_human_review")
            continue

        work_order = gemini_pro_work_order(
            event=event,
            hypothesis=hypothesis,
            fixed_urgency=urgency,
            allowed_parts=rules.parts_catalog(event.factory_id),
            schema=CMMSWorkOrder,
        )
        work_order = rules.enforce_admissible_work_order(work_order)
        outbox.store_inferred_work_order(item.event_id, work_order, state="pending_cmms")

def dispatch_pending_cmms():
    if not cmms.reachable():
        return
    for item in outbox.pending_cmms_ordered():
        cmms_id = adapter_for(item.factory_id).create_work_order(
            item.work_order,
            idempotency_key=item.event_id,
        )
        outbox.mark_dispatched(item.event_id, cmms_id)
```

Uncertainty uses a cascading classifier with abstention thresholds. Stage 0 is deterministic rules for obvious maintenance classes. Stage 1 is Gemini Flash categorical classification. Stage 2 is Gemini Pro review only for ambiguous cases. Any `unknown`, low confidence, rule/model contradiction, missing required fields, or stale event beyond configured age becomes `requires_human_review` in the durable queue. This is a mathematically defensible abstention cascade: the system narrows decisions through staged classifiers and explicitly refuses to classify outside the admissible set.

`CMMSAdapter` contract:

```python
class CMMSAdapter(Protocol):
    provider: Literal["maximo", "sap_pm"]
    def create_work_order(self, payload: CMMSWorkOrder, idempotency_key: str) -> str: ...
    def healthcheck(self) -> bool: ...
```

The idempotency key is `event_id`. The Courier also assigns a monotonic `courier_sequence_id` for local ordering. On replay after outage, adapters upsert by `external_event_id`.

## D. THE NATIVE ENVIRONMENT UI SPEC

The primary UI is a status panel embedded in the customer's existing CMMS admin or maintenance planning page, not a Matta dashboard. For Maximo it is a simple application tab; for SAP PM it is a launchpad tile or embedded admin page. The user is a maintenance planner or reliability engineer responsible for making sure offline work-order sync is healthy.

The UI shows queue counts by state: queued, inferred, pending CMMS, dispatched, requires human review. It lists event ID, asset, line, age, root-cause state, and next retry. Actions are Retry Now, Mark Reviewed, and Open Created Work Order. It must tolerate no internet connection and keep rendering from local SQLite/Postgres state.

## E. PHASE 1 EXECUTION SPEC

Repo deltas from Master PRD Section 6.1: keep shared schemas, prompts, and adapters; add `apps/courier_service`, `packages/queue/durable_outbox.py`, `packages/governance/airgap_rules.py`, `apps/airgap_status_ui`, and `apps/mocks/spool_writer.py`. Use SQLite for the demo durable queue with an option to swap to Postgres in production.

FastAPI and service contracts:

```text
POST /courier/ingest-spool -> demo endpoint that simulates a new Matta spool file
POST /courier/poll-matta-rest -> pulls from Matta public REST export when available
POST /courier/process-outbox -> runs inference for queued events when Vertex AI is reachable
POST /courier/dispatch-cmms -> dispatches pending work orders when CMMS is reachable
GET /courier/status -> returns queue state for embedded CMMS admin UI
POST /courier/review/{event_id} -> marks requires_human_review item reviewed
POST /cmms/{provider}/work-orders -> mock SAP PM/Maximo endpoint with outage toggle
```

Vertex prompt templates:

```text
Flash cascade prompt: classify into allowed root_cause enum with confidence_band; choose unknown when information is insufficient.
Pro review prompt: review an ambiguous Flash result and output RootCauseHypothesis with cascade_stage=pro_review; preserve unknown if uncertainty remains.
Pro work-order prompt: output CMMSWorkOrder JSON using fixed urgency, fixed provider, fixed allowed parts, fixed courier_sequence_id, and root-cause hypothesis.
```

Mock data: generate 25 spool files, including duplicates, stale events, malformed files, and one high-severity valid GAUGE event. Include toggles for Vertex unreachable and CMMS unreachable.

Demo flow: start with Vertex/CMMS marked offline, ingest spool files, show queued local state, restore Vertex to infer work orders, keep CMMS offline so items move to `pending_cmms`, then restore CMMS and show Maximo/SAP PM work orders created in order with stable `external_event_id`.

Magic Moment: a disconnected factory accumulates Matta defect events without data loss, then automatically syncs structured Maximo/SAP PM work orders once connectivity returns.

## F. ANTI-REPLICATION VERIFICATION

Airgap Courier does not encroach on Matta's core IP or closed-loop control patent. Time scale is minutes to hours because events are spooled, queued, inferred, and dispatched asynchronously through human-maintenance systems. The actuator is the durable queue plus SAP PM/Maximo work-order creation, not a machine PLC, firmware parameter, camera stream, robot, extruder, or physical controller. The action space is discrete and administrative: queue, infer, review, retry, and create work order. The failure mode is delayed maintenance or human review during network outage, not automated machine actuation that changes part quality in real time. The data flow is outbound from Matta REST exports or webhook spool files into customer CMMS systems; it never reaches into SENTRY, GAUGE, TALLY, TRACE, MFMs, Manufacturing OS UI, edge firmware, or real-time camera streams.
