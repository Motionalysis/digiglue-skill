---
name: digiglue
description: Design, size, and quote packaging die-lines using the DigiGlue MCP connector. Use this skill whenever the user asks for corrugated boxes, mailers, eCommerce boxes, HSCs, RSCs, die-lines, blank dimensions, rule lengths, packaging quotes, or mentions ECT board grades or flutes (A, B, C, E). Covers the 12 DigiGlue tools, the packaging vocabulary they expect, hard rules (Y-before-X dimensions, single-call batching, customer-required quoting), and worked example workflows.
---

# DigiGlue

DigiGlue is a parametric packaging die-line library and quoting engine surfaced via MCP at `https://digiglue.io/api/mcp`. This Skill teaches Claude how to use it effectively.

## When this skill applies

Trigger when any of these are true:
- The user has the DigiGlue connector enabled and references packaging.
- The user mentions a packaging shape (mailer, EComm box, HSC, RSC, tray, roll end, etc.).
- The user mentions a corrugated board grade (e.g., "32 ECT B") or flute letter (A, B, C, E).
- The user asks for die-line PDFs, blank dimensions, rule lengths, or production quotes.
- The user attaches a vector PDF die-line and wants it priced.

## Tool routing decision tree

| User intent | Tool | Notes |
|---|---|---|
| "What templates / categories exist?" | `list_shape_categories` then `search_shapes` | Catalog discovery |
| "Show me a preview of template X" | `get_shape_preview` | Returns PNG (view1=3D, view2=flat die-cut, view3=notes) |
| "What parameters does template X take?" | `get_shape_parameters` | For custom rendering |
| "I need a quick size / stats for X" | `size_shapes` with `includeDieline: false` | Stats only, no PDF |
| "Give me a die-line for X" | `size_shapes` (default) or `render_shape` | Returns stats + embedded PDF |
| "Give me die-lines for X at multiple sizes" | `size_shapes` with `items[]` | One call, not N separate calls |
| "Quote me X for customer Y" | `quote_shape` | Tier prices + saved quote |
| "Quote this attached PDF die-line" | `quote_pdf` | PDF base64 input |
| "What customers / materials / sheets / operations do I have?" | `list_customers`, `list_materials`, `list_sheets`, `list_operations` | Tenant settings audit |

The catalog tools (`list_*`, `search_*`, `get_*_preview`, `get_*_parameters`, `render_shape`, `size_shapes`) are read-only and available with any authenticated account. The quoting tools (`quote_shape`, `quote_pdf`) and tenant-settings tools (`list_customers`, `list_materials`, `list_sheets`, `list_operations`) require a Pro or Enterprise account with at least one customer, material, and operation configured.

## Hard rules

These are non-negotiable. Violating them produces output that's wrong for production:

1. **Y before X.** Present every blank dimension as `Y × X` (with an explicit "(Y × X)" label). The Y axis is vertical / across the machine; X is horizontal / through the machine. The server's text responses already include the label — preserve it; never reorder.

2. **Batch.** When the user requests multiple shapes, sizes, or quantities in one breath, make ONE tool call with `items[]`, not N separate calls. `size_shapes` and `quote_shape` both accept multi-item input. Splitting into N calls is slower, more expensive, and breaks the user's mental model of "one quote, multiple lines."

3. **Customer is required for quotes.** Every `quote_shape` and `quote_pdf` call needs a customer name. The server fuzzy-matches against the user's tenant; pass the customer name verbatim from the user's message. Don't invent customers.

4. **Defaults are fine.** When the user doesn't specify a material or sheet, omit the field and let the server use the tenant's `is_default = true` row. Don't ask the user to pick a sheet unless they care.

5. **Aliases work — let the server resolve.** "Mailer," "EComm box," and "HSC" are mapped to template IDs (`B004`, `B004`, `C014`) by the server's alias dictionary. Call `search_shapes` with the user's word; the resolver returns the right template. Do not hardcode template IDs in your prompts or memorize a static mapping — the alias table is server-of-record and may change.

6. **Trust flute names.** When the user says "C-flute" or "32 ECT B kemi," pass the flute/material string through. The server looks up the caliper Allowance and the material cost.

7. **Numeric tier quotes.** When the user gives multiple quantities ("500, 750, and 1000"), pass them as a `quantities[]` array per item. The server returns a tier-priced response. Don't quote each quantity in a separate call.

## Packaging vocabulary primer

### Flutes (corrugated profile)

| Flute | Caliper (mm) | Use |
|---|---|---|
| A | 4.8 | Premium cushioning, larger boxes |
| B | 3.2 | Common for printing |
| C | 4.0 | Ubiquitous shipping carton (RSCs, HSCs) |
| E | 1.6 | Fine flute, retail / display / mailers |

### Board specs

- **ECT (Edge Crush Test)** — a board performance rating, e.g., "32 ECT," "44 ECT," "55 ECT." Higher = stronger.
- **ECT + flute**: "32 ECT B" = 32 ECT board with B-flute.
- **ECT + flute + liner**: "32 ECT B kemi" = 32 ECT, B-flute, with kemi (mottled white inside) liner. "mw" suffix = mottled white. "kraft" = unbleached kraft outer.

### Common shape vocabulary

| User says | Template ID | What it is |
|---|---|---|
| Mailer | B004 | Folder-style shipping box with top-flap closure |
| EComm box | B004 | Same as mailer; eCommerce-optimized |
| HSC (Half Slotted Container) | C014 | Open-top corrugated shipping carton |
| RSC (Regular Slotted Container) | C001 | Standard closed-top corrugated shipping carton |
| 1PF (One Piece Folder) | C001 | Same as RSC in this taxonomy |
| Roll end | (category) | Sub-folder of templates for roll-end packaging |
| Tray | (category) | Open-top trays |

Aliases live server-side in `server/services/templateAliases.cjs`. The `search_shapes` tool is the source of truth — call it with the user's word, not a guessed ID.

## Example workflows

### Example 1: Spec a die-line on demand

User: *"Give me a PDF Die Line and stats for an HSC that's 12x10x10 in C-flute."*

Approach:
1. `search_shapes(query: "HSC")` → resolves to `C014` in Corrugated / (subcategory).
2. `size_shapes` with one item: `{templateId: "C014", category, subcategory, parameters: {length: 12, width: 10, depth: 10}, flute: "C"}`. Leave `includeDieline` at default (true) since the user asked for a PDF.
3. Reply with the blank dimensions formatted as "Y × X (Y × X)," the rule length (rounded up to whole inches/mm in the user's unit system), and the attached die-line PDF.

### Example 2: Multi-item tier quote with per-item material and operation overrides

User: *"Give me a quote for CompanyX for 500, 750 and 1000 of a 12x9x4 mailer on 32 ECT B kemi, digital print and cut, plus 2000 of an 8x7x2 EComm box on 32 ECT E mw, with digital print and cut, and 100 of a 14x6x11 HSC on kraft, cut only."*

Approach:
1. Optionally `list_materials` and `list_operations` if you want to confirm names. The fuzzy resolver tolerates substring matches, so this is usually skippable.
2. ONE call to `quote_shape` with `customer: "CompanyX"` and `items[]`:
   - Item 1: mailer (resolves to B004), parameters `{length: 12, width: 9, depth: 4}`, material "32 ECT B kemi", operations ["Digital print and cut"], quantities `[500, 750, 1000]`.
   - Item 2: EComm box (also B004), parameters `{length: 8, width: 7, depth: 2}`, material "32 ECT E mw", operations ["Digital print and cut"], quantities `[2000]`.
   - Item 3: HSC (C014), parameters `{length: 14, width: 6, depth: 11}`, material "kraft" (resolves by fuzzy match), operations ["Cut only"], quantities `[100]`.
3. Reply with the tier-priced quote: per-unit cost per quantity tier per item, plus total. The server saves the quote as `Q-NNNNN` and returns a quote PDF.

Critical: **one call**, not three. The server's per-item override mechanism handles the material and operations differences.

### Example 3: Quote from a customer-supplied PDF die-line

User: *"I need 50, 100 and 200 of the attached PDF Die Line quoted in 32 ECT B kemi for CompanyX, with digital print and cut."*

Approach:
1. Read the attached PDF — Claude has it as a binary; pass as base64 to the tool.
2. Call `quote_pdf` with:
   - `pdf`: base64 string of the PDF
   - `customer`: "CompanyX"
   - `material`: "32 ECT B kemi"
   - `operations`: ["Digital print and cut"]
   - `quantities`: [50, 100, 200]
3. The server parses the vectors from the PDF, auto-nests against the default sheet, and runs the same cost engine `quote_shape` uses. Returns tier prices + saved quote PDF + a server-rerendered die-line PDF.

Cold-start latency: the first `quote_pdf` call after a long idle period can take 2-3 minutes (PDF parsing + nesting). Subsequent calls are typically under 30 seconds.

### Example 4: Browse the catalog

User: *"What template categories are available, and how many shapes are in the Roll Ends category?"*

Approach:
1. `list_shape_categories` → returns the full category/subcategory tree with shape counts.
2. Pull the count from the Roll Ends entry in the response.

If the user then asks "show me the first few" or similar, call `search_shapes` with a category filter or `get_shape_preview` on a specific template.

### Example 5: Stats only (no PDF)

User: *"Give me the stats for an HSC 12x12x10 in C-Flute"*

Approach:
1. `search_shapes(query: "HSC")` → C014.
2. `size_shapes` with `includeDieline: false` (or with the user's phrasing "just stats / stats only / no PDF" detected). This returns blank dimensions and rule length without the PDF payload — faster and cheaper.

### Example 6: Audit tenant configuration

User: *"What materials, sheet sizes, customers, and work cells do I have configured?"*

Approach:
1. Call all four list tools — `list_customers`, `list_materials`, `list_sheets`, `list_operations` — in parallel.
2. Reply with a compact summary per category: name + key fields (e.g., for materials: cost-per-unit, waste factor, default flag).

## Response formatting

- **Tier quotes**: render as a small table (item, quantity, per-unit, total).
- **Blank dimensions**: "Y × X (Y × X)" always — preserve the server's labeling.
- **Rule lengths**: in the user's unit system (server returns inches or mm based on the API key owner's setting).
- **Saved quote IDs**: surface them — the user will reference these later.
- **Die-line PDFs**: include the inline attachment from the tool response.

## Tier behavior and gating

DigiGlue has a free tier and paid tiers (Pro, Enterprise):

- **Free**: 15 generates/day across catalog tools; no PDF downloads except sampler templates (B004, C013); 10-minute session timeout.
- **Pro / Enterprise**: Unlimited generates, full PDF downloads, `quote_shape` and `quote_pdf` access, persistent quotes.

If a free-tier user tries to call `quote_shape`, the server returns a clear error explaining what's required. Surface that error to the user with the upgrade path; don't try to work around it.

## Connector setup (for users)

Direct users who haven't connected DigiGlue yet to:
1. claude.ai → Settings → Connectors → Add custom connector
2. Server URL: `https://digiglue.io/api/mcp`
3. Sign in with their DigiGlue account.
4. Public docs and example prompts: `https://digiglue.io/docs/connector`

## Public documentation

- Connector docs: https://digiglue.io/docs/connector
- Privacy policy: https://digiglue.io/privacy
- Sample PDF die-line for testing `quote_pdf`: https://digiglue.io/MCPTest.pdf
