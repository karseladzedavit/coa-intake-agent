# CoA Intake Agent

An email arrives at Northgate Cycles with a supplier's certificate of analysis
attached. The agent reads it, checks it against approved vendors, part specs,
and QA policy, and routes it to the right outcome: release straight to
production, send to human review, put the lot on hold, or ask the sender for
the correct document.

The model reads. The code decides. Retrieval only changes an outcome when a
real, vendor-specific rule applies to it.

## What it does

1. **Email intake** — an n8n Gmail trigger picks up incoming certificates,
   sends the attachment to the extraction service, and routes the result.
2. **Extraction** — the PDF or scanned image is read twice by two different
   models. Disagreements on key fields trigger a third read, and the
   majority answer wins. Every value keeps the exact sentence it came from.
3. **Decision** — plain Python checks the vendor, the part spec limits, unit
   conversions, and borderline margins. No AI decides pass or fail.
4. **RAG** — retrieved knowledge can soften a finding (never fully clear it)
   when a specific, named clause applies. See "Why retrieval changes
   outcomes" below.
5. **Routing** — release goes to a sheet, review and hold go to a sheet plus
   Slack, reject gets a reply email asking for the correct document.

## Architecture

```
Gmail (attachment)
      |
      v
n8n: Agent  -->  POST /process  -->  Python service
      |                                  |
      v                                  v
   Route                          extractor.py (read the document)
      |                                  |
      |                                  v
      |                           decision.py (judge it)
      |                                  |
      v                                  v
Sheets / Slack / Gmail reply     rag.py (vendor + spec + policy)
```

The Python service does the reading and the deciding. n8n does the moving.
Nothing in `decision.py` calls an email or a spreadsheet, and nothing in the
n8n workflow does arithmetic on a test result.

## Why the knowledge base is split the way it is

Two of the five knowledge files are exact lookups, not searched by
similarity at all:

- `approved_vendors.md` — is this vendor known, and approved for this part.
  Matched with fuzzy string matching (`rapidfuzz`), because a vendor name is
  a fixed string with occasional spelling variation, not something that
  needs interpreting.
- `part_specs.md` — what are the limits for this part number. An exact
  dictionary lookup, because a part number is a key, not a concept.

Three files are prose with conditions in them, and only these are chunked
and embedded into the vector store:

- `qa_policy.md` — general rules (borderline margins, certificate age,
  confidence threshold).
- `vendor_agreements.md` — per-vendor exceptions, each naming one vendor and
  one situation.
- `inspection_procedure.md` — unit conversions and accepted test methods.

Putting the vendor list or the spec numbers into the vector store would
make an exact-match problem fuzzy and slower for no benefit. The vector
store holds only the material where the applicable rule depends on
interpreting a situation against a written condition.

## Why retrieval changes outcomes, not just decoration

Every non-passing finding is checked against retrieved clauses for that
specific vendor. On `coa_02_messy_scan.pdf`, two findings that would
otherwise be a HOLD are downgraded to REVIEW because of real Midwest Forge
clauses:

- A tensile strength result converts from PSI to 799.79 MPa against an
  800 MPa minimum. The retrieved clause says a shortfall under 0.5 MPa from
  unit conversion is a rounding artifact, not a real failure.
- A plating result is reported as PASS with no number, normally
  unacceptable for a REQUIRED test. The retrieved clause says PASS is
  acceptable for this vendor when a report number is present.

Delete `.chroma` and re-run the same document: both clauses disappear, both
findings revert to HOLD, and the outcome gets worse. That is the test that
the retrieval is load-bearing.

The control case is `coa_03_unknown_vendor.pdf`. That vendor has no section
in `vendor_agreements.md`, so no clause is found and nothing is softened,
even though the document also has a genuine problem (unknown vendor,
missing required test). Clauses only apply to the vendor they name.

## Safety rail on retrieved clauses

A retrieved clause can downgrade a finding's severity from HOLD to REVIEW.
It can never clear a finding to RELEASE. An out-of-spec numeric result
always needs a human to sign it off, no matter what a piece of prose says
about it; the clause can only change who looks at it, not whether anyone
does.

## Why extraction runs twice, and at temperature 0

Two different models read every document. If they disagree on a key field
(vendor, part number, lot number, PO, date) or any test value, a third read
is taken and the majority wins, with a note recorded and confidence
lowered.

All reads run at temperature 0, so the same document produces the same
extraction on every run. Multi-read voting then reflects genuine ambiguity
in the source image (a smudged digit, a similar-looking character) rather
than sampling randomness between identical calls.

## Results

Verified by hand against the three sample certificates, run both directly
(`scripts/run_local.py`) and through the full email path in n8n.

| document | outcome | what it exercises |
|---|---|---|
| `coa_01_clean.pdf` | RELEASE | clean native PDF; two-model voting caught and resolved a vendor-name disagreement; confidence landed exactly at the review threshold |
| `coa_02_messy_scan.pdf` | REVIEW | image-only scan; unit conversion; two vendor clauses retrieved and applied, both downgrading a HOLD |
| `coa_03_unknown_vendor.pdf` | HOLD | vendor not on the approved list; a REQUIRED test missing entirely; no vendor clause exists for this vendor, so nothing is softened |

## Running it

```bash
python -m venv .venv
.venv\Scripts\Activate.ps1        # or source .venv/bin/activate on macOS/Linux
pip install -r requirements.txt
copy .env.example .env            # add OPENAI_API_KEY
```

Test the core without the API or n8n:

```bash
python scripts/run_local.py samples/coa_01_clean.pdf
```

Run the service:

```bash
uvicorn main:app --reload --app-dir app --host 0.0.0.0 --port 8000
```

Import `n8n/coa-intake.json`, set Gmail, Google Sheets, and Slack
credentials, replace the placeholder spreadsheet ID in both Sheets nodes,
and point the Agent node at `http://localhost:8000/process` (or
`http://127.0.0.1:8000/process` if `localhost` does not resolve on your
machine).

The exported n8n workflow (`n8n/coa-intake.json`) references a test
Google Sheet used during development. Replace the spreadsheet ID in both
Sheets nodes with your own before running.

## Known limitations

- **Binary field name is assumed fixed.** The n8n workflow reads the
  Gmail trigger's attachment from the field `attachment_0`. An email with
  multiple attachments would need a selection step.
- **No duplicate guard.** A re-polled or re-sent email is processed again.
  Results are saved to `runs/<run_id>.json` keyed on a random ID, not on
  the source message, so nothing currently detects a repeat.
- **`_limits_from_spec` runs on every request.** It converts spec prose
  into numeric limits with a model call each time rather than once per
  part. A per-part cache would remove an unnecessary call and a small
  source of run-to-run variance.
- **No containerisation.** The service runs directly with `uvicorn`; there
  is no Dockerfile.
- **Single-tenant assumptions.** Vendor and spec files are specific to one
  buyer (Northgate Cycles) and are not scoped per customer.

## Built with

Built together with a friend; both contributed to the codebase and testing.
