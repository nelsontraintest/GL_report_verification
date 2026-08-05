# Super Prompt — GL vs Data-Table Verification HTML

Copy everything below the line into the company AI chatbot. Attach:
1. the **data table image** (or CSV if image OCR is unavailable), and
2. the **GL Excel** file.

---

## Role

You are a senior frontend engineer building a **single-file, offline-capable HTML verification tool** for a finance expert. They must confirm that Account × Cost_Center totals on a summary **data table** match the sum of filtered rows in a **General Ledger (GL)** Excel file.

Do **not** invent business rules beyond what is specified. Prefer exact match on money (2 decimal places). Label the UI clearly as a verification workspace, not a dashboard marketing page.

---

## Inputs (attached / provided)

### A) Data table (from image)

Extract a structured table from the attached image (fallback: attached CSV). Expected columns:

| Column | Meaning |
|--------|---------|
| `As_Of_Date` | Period-end date shown on the summary (e.g. `06-30-2026` or `2026-06-30`) |
| `Account` | Account number |
| `Cost_Center` | Cost center code |
| `Amount` | Expected total for that Account × Cost_Center mix |
| `Check_Result` | Initially empty; updated by this tool to `Match` / `Mismatch` / blank |

If OCR is imperfect, show an editable preview of the extracted rows **before** locking them into the UI, and let the user correct cells.

### B) GL Excel

Load the attached workbook. Prefer sheet named `GL Detail` if present; otherwise use the first sheet that has the required columns (case-insensitive header match / common aliases OK).

Required logical fields:

| Logical field | Example header names |
|---------------|----------------------|
| Posting date | `Posting_Date`, `Date`, `Posting Date` |
| Account | `Account`, `Account_No`, `Account Number` |
| Cost center | `Cost_Center`, `Cost Center`, `CC` |
| Amount | `Amount`, `Debit`, or signed amount column |

Optional display fields if present: `Document_No`, `Account_Name`, `Cost_Center_Name`, `Description`.

Parse dates robustly (Excel serial dates and `YYYY-MM-DD` / `MM-DD-YYYY`). Parse amounts as numbers (strip commas/currency symbols).

---

## Business rule (how finance verifies)

For a data-table row with:

- `As_Of_Date` = **D** (example: `2026-06-30`)
- `Account` = **A**
- `Cost_Center` = **C**
- `Amount` = **Expected**

### Date window

`As_Of_Date` means **month-to-date through period end**:

- `date_from` = first day of the same month as D  
- `date_to` = D  

Example: `As_Of_Date = 06-30-2026` ⇒ filter GL posting dates from **2026-06-01** through **2026-06-30** inclusive.

Exclude rows outside that window (prior/next month must not count).

### Account verification path

1. User clicks an **Account** value in the data table.
2. Right panel opens filtered to that Account + the date window derived from that row’s `As_Of_Date` (if multiple as-of dates exist, use the clicked row’s date).
3. Optionally user can further narrow by Cost Center.
4. User **selects** one or more GL rows (checkbox / row click; support Select all visible).
5. Show live **Selected sum**.
6. Compare selected sum to the clicked data-table row’s `Amount` (when the click target is a specific mix row).  
   - If the user clicked Account from a row that is Account×Cost_Center, default compare target = that row’s Amount, and default-filter Cost_Center to that row’s Cost_Center (user can clear Cost Center filter to see all CCs for the account).
7. If absolute difference ≤ `0.01` ⇒ show **Match**; else **Mismatch** (show both numbers and delta).
8. On Match, set that data-table row’s `Check_Result` = `Match`. On Mismatch, set `Mismatch`. Do not clear other rows’ results unless the user re-verifies that row.

### Cost Center verification path

Same process, but click target is **Cost_Center**:

1. Filter GL by Cost Center + date window.
2. Optionally narrow by Account.
3. User selects rows → selected sum → compare to the clicked row’s Amount (default also filter Account to the clicked row’s Account; user can clear).
4. Update `Check_Result` the same way.

### Compare target clarity

Always show in the right panel:

- Which data-table row is under review (As_Of_Date, Account, Cost_Center, Expected Amount)
- Active filters (date_from, date_to, Account?, Cost_Center?)
- Visible row count, selected row count
- Expected, Selected sum, Delta, Match/Mismatch badge

---

## UX / layout requirements

Build **one self-contained HTML file** (inline CSS + JS). No build step. Use CDN only if necessary; prefer zero external deps. File inputs should work when opened via `file://` or local server.

### Layout

- **Left / main:** the extracted data table (sticky header). Account and Cost_Center cells are clickable (button-like, keyboard accessible).
- **Right panel (drawer):** opens on click; can be closed/reopened. Contains:
  - Filter controls (date range prefilled, Account, Cost Center) — editable
  - GL grid of filtered rows with selection checkboxes
  - Running selected sum + Match/Mismatch status
  - Buttons: Select all visible, Clear selection, Apply filters, Mark Match manually (optional), Reset row check result
- **Top bar:**
  - Upload / replace Data Table image or CSV
  - Upload / replace GL Excel (`.xlsx`)
  - Export: download updated data table CSV including `Check_Result`
  - Short help text explaining the date-window rule

### Visual / interaction

- Finance-tool aesthetic: dense, readable table; clear numeric alignment; Match = green, Mismatch = red/amber.
- Do **not** build a marketing landing page, hero, or card-heavy dashboard.
- Show empty states: “Upload GL”, “No rows match filters”, “Select a data-table Account or Cost Center”.
- Money format: thousands separators, 2 decimals.
- Keep selection + filters responsive for hundreds of GL rows (virtualize only if needed; otherwise simple filter is fine for ≤5k rows).

### Accessibility

- Clickable Account / Cost_Center are real `<button>`s or have `role="button"`, Enter/Space activate.
- Visible focus states.
- Status text not color-only (include “Match” / “Mismatch” words).

---

## Technical requirements

1. **Single HTML file** output as the primary deliverable.
2. Parse `.xlsx` in-browser (e.g. SheetJS from CDN is acceptable if required; document the dependency in an HTML comment at the top).
3. Image → table:
   - If the chat environment can OCR/extract for you up front, embed the extracted JSON into the HTML as initial `DATA_TABLE`.
   - Also keep client-side CSV upload as a fallback editor.
4. Matching tolerance: `Math.abs(selectedSum - expected) < 0.005` after rounding both to 2 decimals, or equivalently both rounded to 2dp are equal.
5. Do not mutate the original Excel; selection/verification is session state only (except CSV export of the summary).
6. Include a small **demo seed** only if no files are attached: use the sample numbers below so the UI is inspectable. If real attachments are present, use those instead.

### Sample seed (only when attachments missing)

Data table:

```text
As_Of_Date,Account,Cost_Center,Amount,Check_Result
2026-06-30,5100,CC-100,12500.00,
2026-06-30,5100,CC-200,8300.00,
2026-06-30,5200,CC-100,4100.00,
2026-06-30,5200,CC-300,2750.50,
2026-06-30,6100,CC-100,9600.00,
2026-06-30,6100,CC-200,1520.25,
2026-06-30,7100,CC-400,3200.00,
```

GL must be loaded from the attached Excel when provided. If you must invent demo GL rows, they must sum exactly to the seed amounts for June 2026 MTD, and include a few May/July distractor rows.

---

## Acceptance tests (you must satisfy these)

After generating the HTML, mentally (or actually) verify:

1. Clicking Account `5100` on the `CC-100` row prefills Account=`5100`, Cost_Center=`CC-100`, dates `2026-06-01`..`2026-06-30`.
2. Selecting all visible in-range rows for that mix yields sum `12500.00` → **Match**, and that row’s `Check_Result` becomes `Match`.
3. Clearing Cost Center filter for Account `5100` shows both `CC-100` and `CC-200` June rows (and not May/July).
4. Clicking Cost Center `CC-100` on the `5200` row prefills Cost_Center=`CC-100`, Account=`5200`, same June window; selecting matching rows yields `4100.00` → Match.
5. If user selects only a subset so sum ≠ expected → **Mismatch**; `Check_Result` = `Mismatch`.
6. Export CSV includes updated `Check_Result` values.
7. Works after fresh reload with re-uploaded files (no hidden server).

---

## Output format

1. Briefly restate the extracted data-table rows (markdown table) for human confirmation.
2. Deliver the **full HTML source** in one code block (complete file, not a diff).
3. Add a short “How to use” (5 bullets max) after the code.
4. Do not omit script sections or say “rest of code same”.

---

## Non-goals

- No login, no backend, no database.
- No automatic “verify all rows” unless you also provide a clearly secondary button; default flow is **human-selected rows**.
- No changing GL source data.
- No speculative AI explanations of why amounts differ beyond showing Expected / Selected / Delta.
