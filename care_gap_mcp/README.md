# Care Gap Closer MCP Server — BroCoders

Short tagline (optional): **BroCoders**

About our offerings: **FHIR-ready MCP tools** that identify **USPSTF-aligned preventive care gaps** using a deterministic rules engine and draft **patient-facing outreach** (SMS/portal) at a patient-friendly reading level, designed for clinician review before sending.

A SHARP-on-MCP server that exposes five tools for USPSTF-aligned preventive
care-gap detection and patient outreach. Built on `fastmcp` with the Prompt
Opinion `POFastMCP` extension for FHIR context.

## Endpoint

- MCP base URL: `http://127.0.0.1:9000/mcp`
- Transport: HTTP JSON responses (configured via `json_response=True`)

## Tools

| Tool | What it does | LLM? |
|---|---|---|
| `SummarizePatient` | Returns demographics + computed age | No |
| `ListActiveConditions` | Active problem list with SNOMED + ICD-10 codes | No |
| `ListRecentObservations` | Labs, vitals, and screening procedures within N months | No |
| `FindCareGaps` | **Rule engine** identifies USPSTF gaps; **Gemini** authors the per-patient clinical rationale | Yes |
| `DraftOutreachMessage` | Gemini drafts SMS + portal copy for a specific gap, sixth-grade reading level | Yes |

The rule engine is deterministic — we never let the LLM invent a gap. The LLM
is only used for *authorship* (rationale, patient copy), which is exactly what
rule-based systems can't do.

## Tool details (what to send / what you get back)

All tools are invoked via MCP over HTTP and require SHARP-on-MCP FHIR context headers (below) unless otherwise noted.

### `SummarizePatient`
- **Use for**: Confirming patient identity (name, DOB/age, sex) before any care-gap workflow.
- **Requires FHIR context**: Yes
- **Returns**: Patient demographics summary and computed age.

### `ListActiveConditions`
- **Use for**: Showing the active problem list and coded evidence.
- **Requires FHIR context**: Yes
- **Returns**: Conditions with SNOMED/ICD-10 codes (as available) and status.

### `ListRecentObservations`
- **Use for**: Reviewing key vitals/labs/screening procedures over a recent window.
- **Requires FHIR context**: Yes
- **Inputs**: Typically a lookback window (months) depending on implementation.
- **Returns**: Observations/procedures within the window.

### `FindCareGaps`
- **Use for**: The authoritative care-gap list for the patient in context.
- **Requires FHIR context**: Yes
- **Returns**: Structured gaps with machine-checkable evidence plus a one-sentence clinician rationale authored by the LLM **from evidence**.

### `DraftOutreachMessage`
- **Use for**: Patient-facing outreach for a specific gap.
- **Requires FHIR context**: Yes (because the gap dict is derived from patient context)
- **Inputs**: The full gap object, `patient_name`, and a `channel` (`sms`, `portal`, or `both`).
- **Returns**: Copy-ready outreach text for the requested channel(s).

## Care gaps implemented

- **Diabetes A1c overdue** — active DM (E10/E11/E13) + no LOINC 4548-4 in 6mo
- **Hypertension BP overdue** — active HTN (I10–I15) + no LOINC 8480-6 in 12mo
- **Colorectal screening overdue** — age 45–75 + no colonoscopy in 10y / FIT in 1y
- **Mammography overdue** — female, age 40–74 + no mammogram in 24mo

## Run locally

```shell
cd care_gap_mcp
uv sync
GOOGLE_API_KEY=your-key uv run python main.py
```

Server listens at `http://127.0.0.1:9000/mcp`.

## FHIR context

Per SHARP-on-MCP, FHIR credentials arrive as HTTP headers:

- `X-FHIR-Server-URL`
- `X-FHIR-Access-Token`
- `X-Patient-ID`

If any required header is missing on a patient-specific tool, the tool returns
`{"status": "error", "message": "..."}`.

## Quick test (curl)

The exact MCP HTTP payload shape can vary by client. The simplest way to verify end-to-end is via the Prompt Opinion portal’s MCP tester. If you want to test locally with curl, first confirm the server is reachable:

```bash
curl -sS "http://127.0.0.1:9000/mcp"
```

Then use a client that speaks MCP-over-HTTP for tool invocation (Prompt Opinion UI or an MCP SDK client). When invoking tools, include the 3 FHIR context headers above.

## Common failure modes

- **Missing FHIR context headers**: You’ll get an error payload from the tool indicating what header is missing.
- **Invalid/expired access token**: FHIR calls fail; refresh SMART credentials and retry.
- **Patient not found**: Verify `X-Patient-ID` exists on the target FHIR server.
- **No gaps returned**: This is a valid outcome; the rules engine found no matching overdue gaps in the patient record.

## Expose to the Prompt Opinion portal

Run `ngrok http 9000` and register the resulting `https://<id>.ngrok-free.app/mcp`
URL in your PO workspace's MCP server registry.
