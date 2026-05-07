# LATERAL PRD v1: Wrenchline Mobile PWA

## A. THE CONCEPT

**Wrenchline Mobile PWA** is a field-maintenance-first lateral of the canonical CMMS Bridge. Instead of making Slack/Teams the primary acknowledgement surface, Matta outbound defect webhooks create a governed draft maintenance packet, push a mobile PWA notification or QR deep link to the on-shift technician, allow optional voice/photo context capture at the machine, and only then write the structured work order into Fiix or UpKeep. The architectural delta is a mobile-first, human-confirmed workflow: push ingress remains event-driven, but the native UI becomes the technician's phone at the asset, and the agent pipeline includes multimodal technician context before final CMMS dispatch.

## B. THE STRATEGIC HOOK

This lateral speaks to Doug Brion's belief that real manufacturing still depends on embodied shop-floor judgement: "Factories still rely on the human skill of people who just know when something is off. The engineer who hears a wobble. The operator who spots a flaw before anyone else." [intel.md: Matta repost, founder activity]. Wrenchline keeps that human skill inside the loop by asking the technician to confirm or enrich the work order from the machine, not from a separate analytics dashboard.

It also matches Matta's deployment pressure. Doug wrote that "we're deploying to around two factories a month and have a multi-year waitlist" [intel.md: Doug Brion LinkedIn, funding post], and the FDE hiring post describes an "avg week" as "visiting factories for plane wings/soup tins/electronics" [intel.md: Doug Brion LinkedIn, FDE hiring post]. A mobile PWA avoids per-factory desktop rollout and works across the mixed plant-floor environments implied by that factory spread.

The uncertainty posture is intentionally rigorous because the dossier says Doug has a "rigid intolerance for \"black-box\" AI models" and demands "probabilistic safeguards, and rigorous uncertainty mapping" [Dossier section 4]. Wrenchline uses ensemble voting, explicit `unknown`, and technician confirmation before CMMS creation; it does not pretend vector similarity is confidence.

## C. THE AGENT ARCHITECTURE

All inference uses `google-genai` against Vertex AI in `europe-west4`. The sidecar consumes only Matta's outbound `MattaDefectEvent` webhook or demo publisher. It never calls SENTRY, GAUGE, TALLY, TRACE, MFMs, the Manufacturing OS UI, edge firmware, or camera streams.

Core schemas:

```python
class MattaDefectEvent(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)
    event_id: Annotated[str, Field(min_length=1, max_length=64)]
    agent: Literal["SENTRY", "GAUGE"]
    defect_class: Annotated[str, Field(max_length=80)]
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
    rationale: Annotated[str, Field(max_length=400)]

class CMMSWorkOrder(BaseModel):
    model_config = ConfigDict(extra="forbid")
    external_event_id: str
    title: Annotated[str, Field(max_length=120)]
    description: Annotated[str, Field(max_length=1200)]
    urgency_tier: Literal["P1", "P2", "P3", "P4"]
    estimated_labor_minutes: Annotated[int, Field(ge=5, le=480)]
    required_parts: Annotated[list[str], Field(default_factory=list, max_length=10)]
    assigned_team: Literal["mechanical", "electrical", "quality", "maintenance_general"]
    technician_note_summary: Annotated[str | None, Field(max_length=600)] = None
    requires_human_review: bool = False
```

Deterministic governance is located in `packages/governance/wrenchline_rules.py`. It performs four non-LLM decisions: validates the event schema; maps severity to `urgency_tier`; restricts allowed parts to the customer's configured Fiix/UpKeep material catalog; and enforces the technician approval state machine (`drafted -> technician_confirmed | requires_human_review -> cmms_dispatched`). The LLM never chooses whether to create a CMMS work order, never chooses the provider, and never escalates urgency. It only classifies into finite enums or fills narrative fields inside Pydantic schemas.

Pipeline pseudocode:

```python
def handle_matta_webhook(raw_body):
    event = MattaDefectEvent.model_validate_json(raw_body)
    if idempotency.exists(event.event_id):
        return idempotency.cached_response(event.event_id)
    idempotency.reserve(event.event_id)
    assert rules.allowed_agent(event.agent)  # SENTRY or GAUGE only
    draft_id = queue.enqueue("create_mobile_draft", event.event_id)
    return {"status": "accepted", "draft_id": draft_id}

def create_mobile_draft(event_id):
    event = db.events.get(event_id)
    route = rules.cmms_provider_for_factory(event.factory_id)  # Fiix or UpKeep
    urgency = rules.urgency_from_severity(event.severity)

    samples = parallel([
        gemini_flash_root_cause(event, temperature=0.1),
        gemini_flash_root_cause(event, temperature=0.5),
        gemini_flash_root_cause(event, temperature=0.9),
    ])
    consensus = plurality_vote(samples, key="root_cause")
    requires_review = consensus.no_plurality or consensus.value == "unknown"

    draft = DraftWorkOrder(
        event=event,
        route=route,
        urgency_tier=urgency,
        root_cause=consensus.value,
        requires_human_review=requires_review,
    )
    pwa_push.send_to_on_shift_technician(draft.deep_link)

def attach_voice_note(draft_id, audio_blob):
    # Gemini Multimodal Audio, Vertex AI europe-west4.
    summary = gemini_audio_summarize(audio_blob, schema=TechnicianNoteSummary)
    db.drafts.attach_note(draft_id, summary)

def technician_confirm(draft_id):
    draft = db.drafts.get(draft_id)
    rules.assert_transition(draft.status, "technician_confirmed")
    work_order = gemini_pro_work_order(
        event=draft.event,
        root_cause=draft.root_cause,
        urgency_tier=draft.urgency_tier,
        allowed_parts=rules.parts_catalog(draft.factory_id),
        technician_note=draft.note_summary,
        schema=CMMSWorkOrder,
    )
    work_order = rules.enforce_admissible_work_order(work_order)
    cmms_id = adapter_for(draft.route).create_work_order(work_order)
    idempotency.commit(draft.event.event_id, cmms_id)
```

Uncertainty uses the named mathematical pattern of deep ensemble voting: three independent Gemini Flash constrained-generation samples vote over the `root_cause` categorical enum. The surfaced uncertainty object records vote histogram, entropy over categorical votes, confidence band, and whether the system abstained via `unknown` or `requires_human_review`. Gemini Multimodal Audio summarizes technician notes into a bounded schema and is never used for routing or urgency.

`CMMSAdapter` contract:

```python
class CMMSAdapter(Protocol):
    provider: Literal["fiix", "upkeep"]
    def create_work_order(self, payload: CMMSWorkOrder, idempotency_key: str) -> str: ...
    def update_work_order_status(self, work_order_id: str, status: str) -> None: ...
```

The idempotency key is `event_id`; downstream CMMS adapters write it as `external_event_id` and must return the existing CMMS work order on replay.

## D. THE NATIVE ENVIRONMENT UI SPEC

The primary UI is a mobile PWA launched from the technician's existing device workflow: push notification, SMS deep link, or QR code posted near the asset. It targets field technicians in plants where Fiix or UpKeep is already the system of record. It must not mimic the Matta dashboard and must not expose Matta internals.

The PWA shows one job-sized surface: defect summary, asset, urgency, root-cause vote badge, recommended parts, estimated labor, and three actions: Confirm, Add Voice Note, Send to Review. Voice capture uses the phone microphone after explicit tap. Photo upload is allowed only as technician-provided context and is stored as a CMMS attachment; the system never requests or streams Matta camera data. Confirmation creates the CMMS work order and opens the native Fiix/UpKeep work order link.

## E. PHASE 1 EXECUTION SPEC

Repo deltas from Master PRD Section 6.1: keep `apps/bridge_api`, `apps/bridge_worker`, `packages/schemas`, `packages/adapters`, and `packages/prompts`; replace Slack-specific UI with `apps/wrenchline_pwa`; add `packages/governance/wrenchline_rules.py`; add mock adapters `fiix_adapter.py` and `upkeep_adapter.py`; add `apps/mocks/mock_mobile_push.py`.

FastAPI contracts:

```text
POST /webhooks/matta/defect -> accepts MattaDefectEvent, returns {event_id, draft_id, status}
GET /work-orders/{draft_id} -> returns DraftWorkOrderView for PWA
POST /work-orders/{draft_id}/voice-note -> accepts audio upload, stores TechnicianNoteSummary
POST /work-orders/{draft_id}/confirm -> dispatches CMMSWorkOrder to adapter
POST /work-orders/{draft_id}/review -> marks requires_human_review
POST /cmms/{provider}/work-orders -> mock Fiix/UpKeep create endpoint
```

Vertex prompt templates:

```text
Root cause Flash prompt: classify the event into the allowed root_cause enum only; output JSON conforming to RootCauseHypothesis; choose unknown when evidence is insufficient.
Audio prompt: summarize technician speech into observed_sound, observed_condition, requested_part_hint, safety_note, and free_text_summary; never infer a root cause.
Work order Pro prompt: write CMMSWorkOrder JSON using fixed urgency, fixed provider, fixed allowed parts catalog, consensus root cause, and optional technician note.
```

Mock data: generate 30 SENTRY/GAUGE events across polymer extrusion, casting, and bottling line assets. The headline event is a high-severity GAUGE calibration drift that sends a PWA push, records a 10-second voice note, and creates a Fiix work order.

Demo flow: trigger `/webhooks/matta/defect`, show phone PWA opening from push, record a voice note, tap Confirm, and show the resulting Fiix/UpKeep work order with the same `external_event_id`.

Magic Moment: in under 60 seconds, a Matta defect becomes a technician-confirmed CMMS work order from the phone at the machine, with the ensemble vote and `requires_human_review` state visible before dispatch.

## F. ANTI-REPLICATION VERIFICATION

Wrenchline does not encroach on Matta's core IP or the closed-loop control patent. Time scale is minutes to hours because a technician receives, reviews, and confirms a work order; it never runs in millisecond-to-second machine-control time. The actuator is the human technician and Fiix/UpKeep work-order queue, not a PLC, firmware parameter, robot controller, extrusion temperature, flow rate, pressure, or camera stream. The action space is discrete and administrative: confirm, review, assign, and create work order. The failure mode is delayed or reviewed maintenance, not an out-of-spec part caused by automated parameter actuation. The data flow is strictly outbound from Matta webhooks into customer enterprise systems and technician-provided notes; it never reaches into SENTRY, GAUGE, TALLY, TRACE, MFMs, the Manufacturing OS UI, edge device firmware, or real-time camera streams.
