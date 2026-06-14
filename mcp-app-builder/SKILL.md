---
name: mcp-app-builder
description: Generates the HTML/JavaScript code for a Workato MCP App (interactive UI embedded in the LLM chat) from the input and output schema of a Workato MCP tool. Use whenever someone wants to build, scaffold, or generate an MCP App, a tool UI, a submission form, or a results view for a Workato MCP server tool, or pastes a Workato tool input/output schema and asks for the App code. Produces a single self-contained HTML file ready to paste into the Workato MCP App code editor (AI Hub > MCP servers > Apps > Start from scratch). Triggers include "build an MCP app", "generate MCP app code", "create the app for this tool", "make a UI for my MCP tool", "turn this schema into an app", "MCP app from schema".
---

# MCP App Builder

## Purpose

This skill turns a Workato MCP tool's **input schema** and **output schema** into a complete, single-file HTML MCP App. The generated App renders an input form built from the input schema, calls the linked tool, and displays the tool result as a table, summary, and error banner derived from the output schema. The output is ready to paste straight into the Workato MCP App code editor (AI Hub > MCP servers > select server > Server capabilities > Apps tab > Add App > Start from scratch).

An MCP App is a web page that runs inside a sandboxed iframe in the chat. It talks to the MCP server only through the `@modelcontextprotocol/ext-apps` SDK, never through direct network calls, so no Content Security Policy configuration is needed for the apps this skill produces.

**Why this skill exists.** The Workato App editor today offers only two starting points: "Start from scratch" (hand-write the code) or a generic "Submission form" template that is not tied to the tool's schema. Neither generates an App tailored to a specific tool. This skill fills that gap, so people who cannot write JS/HTML can still ship a working, schema-accurate App from three pasted fields and edit it lightly rather than build from zero.

**The one tip that removes most friction.** Mark the tool's input fields as optional. With no required field to chase, the model opens the App immediately instead of asking for parameters in chat first, which is what causes the awkward "result shows in chat before the App appears" experience. See "Making the App open without chat questions".

## When to use an MCP App (evaluate this first, before anything else)

Do not start building on request. First judge whether the use case actually warrants an App. An App is worth it only when the interaction benefits from an interactive UI living in the chat. Per the MCP Apps overview, that means one of:

- **Exploring complex data** — large or multi-source results the user will sort, filter, or drill into.
- **Configuring with many options** — a form of interdependent choices that benefits from seeing everything at once, with validation and defaults.
- **Viewing rich media** — PDFs, images, 3D, audio: anything text cannot convey.
- **Real-time monitoring** — dashboards that update as data changes.
- **Multi-step workflows** — reviewing or acting on items one by one (approve/reject, triage, code review).

If none of these apply, do not build an App. In particular, a **simple GET whose result the model can just summarise in text** should stay a plain tool response. Building an App for a simple read only adds the awkward duplication the user has already noticed: the result lands in chat, and then a separate UI shows the same thing.

Quick test: beyond reading a short answer, would the user *do* something with the result (sort, filter, act, view media, watch it change)? If yes, build the App. If they would just read a sentence or two, say so plainly, recommend a plain tool response instead, and build the App only if the user still wants the richer UI after that.

## Scope boundaries

This skill:

- ✅ Evaluates whether an App is warranted first, and recommends a plain tool response when it is not.
- ✅ Asks the user directly for the tool name and the input/output schemas (does not introspect the tool).
- ✅ Generates a single self-contained `.html` file (HTML + Tailwind via CDN + one inline module script).
- ✅ Builds an input form from the input schema and a results view (table, summary chips, error banner) from the output schema.
- ✅ Wires up the Workato SDK in any of three patterns: result-viewer (default), write + refresh (POST then GET), or form-first, via `ontoolresult` / `callServerTool` / `ontoolinput`.
- ❌ Does not write the server-side Workato connector, tool, or recipe.
- ❌ Does not configure CSP/CORS (not needed; the App only talks to the host via the SDK).
- ❌ Does not invent fields. Every form control and table column maps to a real schema field.

## Inputs the skill needs (always ask the user directly)

Once an App is warranted, **do not try to inspect, introspect, or fetch the tool definition yourself.** Always ask the user directly for these three fields and wait for them. Request all three together in one message:

1. **Tool name** — the exact server tool name passed to `callServerTool` (for example `search_repositories`).
2. **Input schema (JSON)** — the Workato array of field objects for the tool's input. Drives the form and the arguments payload.
3. **Output schema (JSON)** — the Workato array of field objects for the tool's output. Drives the results view.

Why ask rather than inspect: these schemas are authored in Workato and the user copies them straight from the tool's configuration, so they always have them to hand. There is no reliable way to derive them here, and guessing risks inventing fields. If the user provides only the input schema, build the form plus the generic renderer and tell them the table needs the output schema to be richer.

## Workato schema field anatomy

Each schema entry is an object. The fields that matter for code generation:

- `name` — the key used in the arguments payload (input) or the data path (output). **Always use `name`, never `label`, as the object key.**
- `label` — human display text for the form label or table header.
- `type` — `string`, `integer`, `number`, `boolean`, `date`, `date_time`, `array`, or `object`.
- `control_type` — the Workato widget hint: `text`, `text-area`, `password`, `integer`, `number`, `checkbox`, `select`, `multiselect`, `date`, `date_time`.
- `optional` — `true` means not required; `false` means required (add `required` + a red asterisk).
- `hint` — help text. **Scan it for enum options**: patterns like `Can be one of: a, b, c` or `[a, b, c]` mean the field is really a dropdown.
- `of` — for `type: array`, the element type (`string`, `object`, ...).
- `properties` — for `type: object` or array-of-object, the nested field list (becomes table columns or a nested group).
- `pick_list` — an explicit `[label, value]` option list when present.
- `toggle_field` / `toggle_hint` — boolean rendering metadata. Treat the field as a boolean; ignore the toggle wrapper.

Ignore `parse_output`, `render_input`, and conversion hints. They are server-side coercions and have no effect on the UI.

## Visual theme

Every generated App uses one consistent, clean theme: a white card on a light slate page, blue accent border, bold slate headings, red required markers, and a navy submit button. Apply these Tailwind tokens (loaded from the browser CDN) so output matches the Workato submission-form look.

- **Page**: `min-h-screen bg-slate-50 text-slate-800 antialiased`; content wrapper `mx-auto w-full max-w-5xl px-4 py-10` (use `max-w-xl` for form-only apps with no results table).
- **Card**: `rounded-2xl border border-blue-500 bg-white p-8 shadow-sm sm:p-10`.
- **Title**: `text-3xl font-bold tracking-tight text-slate-900`; subtitle `mt-2 text-base text-slate-500`.
- **Field label**: `mb-2 block text-base font-bold text-slate-800`; required asterisk `<span class="font-bold text-red-500">*</span>`.
- **Inputs / selects**: `w-full rounded-lg border border-slate-300 px-4 py-3 text-base text-slate-900 outline-none transition placeholder:text-slate-400 focus:border-blue-500 focus:ring-2 focus:ring-blue-200`.
- **Helper line** (from `hint`): `mt-1.5 block text-sm text-slate-400`.
- **Field spacing**: wrap controls in `grid gap-7 sm:grid-cols-2` (full-width fields get `sm:col-span-2`), or `space-y-7` for a single column.
- **Submit button**: `mt-8 w-full rounded-lg bg-slate-800 py-4 text-base font-semibold text-white transition hover:bg-slate-700 disabled:opacity-60`.
- **Results**: error banner `rounded-lg border border-red-300 bg-red-50`; table card `rounded-2xl border border-slate-200 bg-white shadow-sm`; header row `bg-slate-100 text-xs uppercase tracking-wide text-slate-500`; summary chips `rounded-full bg-slate-200 px-3 py-1 text-xs font-medium text-slate-600`.

If the host passes theme variables (`McpUiHostContext`), the App still renders correctly; these tokens are the default look when no host theme is applied.

## App behaviour: how an App actually launches

This is the single most misunderstood part, so be explicit with the user. An MCP App is **not** opened by the user; it is opened by the model. When the user types a request in chat, the model calls the linked tool itself, the tool runs, and only then does the host open the App and hand it that first result via `ontoolresult`. So a tool call always happens before the user touches the form, and the model may also narrate the result in chat. That is inherent to MCP Apps and cannot be prevented from inside the App.

Given that, pick a pattern and tell the user which one is in effect.

### Choosing the App pattern (decide first, in this order)

1. **Result-viewer (GET, default and most idiomatic).** The tool is a read. Render the model's initial result immediately on `ontoolresult`, then let the user sort, filter, and paginate via more `callServerTool` reads. Use for search, lookups, reports, dashboards, media viewers. This avoids the awkward gap where the result is in chat but the App is blank.
2. **Write + refresh (POST then GET).** The user needs to act on what they see (approve/reject, update, assign). Render the queue (GET), put action buttons on each row, call a **write** tool on click (POST), then re-fetch (GET) and re-render. Use for approval queues and any multi-step review workflow. See the variant section below.
3. **Form-first (input collection).** Only when collecting fresh input is genuinely the point and the initial result is irrelevant. Open blank, optionally pre-fill from `ontoolinput`, and render only after submit. Be aware you are deliberately discarding the model's first result, which still appears in chat.

Rule of thumb: if the tool is a read the user will browse or act on, use result-viewer (1) or write+refresh (2). Reach for form-first (3) only for true data-entry tools. All three set `ontoolresult` before `connect()`; the difference is whether that handler renders (1, 2) or no-ops (3).

### Write + refresh (POST then GET) variant

For action workflows, the App calls two tools:

- the **GET** tool (the linked tool, e.g. `list_pending_approvals`) to load and refresh the list, and
- a **write** tool (e.g. `set_approval_status`) registered as **app-only** on the server (`_meta.ui.visibility: ["app"]`) so it is callable by the App but hidden from the model.

The loop is: render queue → user clicks Approve/Reject → `callServerTool` the write tool (POST) → on success `callServerTool` the GET tool again → re-render. Disable the row's buttons during the call and surface failures in the error banner. Use event delegation on the table body so dynamically rendered buttons stay wired. A complete working instance is in `assets/example_app_actions.html`; reuse its `render` / `refresh` / `act` structure and swap the two tool names and the row columns for the target schema.

## Making the App open without chat questions

A common complaint: the user types a request, the model asks for a parameter in chat, and only after answering does the App open (with the result already dumped in chat). The model asks because the tool has a **required** input it must collect before calling. This is fixed in tool design, not in the App HTML. Apply these levers, in order of impact:

1. **Mark the inputs optional.** On the Workato tool, make input fields optional rather than required. With nothing mandatory unfilled, the model has nothing to chase and calls the tool immediately, opening the App. The App's form then collects the real input. This is the single biggest lever and the foundation of the form-first pattern.
2. **Steer it in the tool description.** Descriptions drive LLM behaviour. Add wording such as: "Opens an interactive form. Call immediately with whatever arguments are known (none is fine); do not ask the user for parameters in chat, the user completes them in the app."
3. **Keep the App form-first.** The App collects input and runs the query on submit via `callServerTool`, so an empty initial call is harmless.
4. **Cleanest: split into two tools (launcher + app-only worker).** A **launcher** tool with no input fields is the linked tool whose only job is to open the App; with nothing to ask about, the model cannot interrogate the user or pre-run the query. An **app-only worker** tool (`_meta.ui.visibility: ["app"]`, hidden from the model) does the actual work and is called by the App on submit. See `assets/example_app_launcher.html`.

Caveat: these bias the model strongly but cannot force it absolutely; the host LLM keeps discretion. Optional fields plus an explicit description works in practice; the two-tool split is the belt-and-braces version.

## Core workflow

Follow these steps in order.

### Step 0: Gate and gather

First run the fit check in "When to use an MCP App". If the use case is a simple read, recommend a plain tool response and stop unless the user insists. If an App is warranted, ask the user directly for the tool name, input schema (JSON), and output schema (JSON), and wait for them before writing any code.

### Step 1: Parse the input schema into form controls

Map each top-level input field with this decision tree (evaluate in order, stop at first match):

1. `type: boolean` (or has `toggle_field`) → `<input type="checkbox">`.
2. `hint` contains enum options, or `pick_list` is present, or `control_type: select` → `<select>` with one `<option>` per value, plus a blank "Default" option for optional fields.
3. `type: integer` → `<input type="number" step="1">`.
4. `type: number` → `<input type="number" step="any">`.
5. `type: date_time` → `<input type="datetime-local">`; `type: date` → `<input type="date">`.
6. `control_type: text-area` → `<textarea>`; `control_type: password` → `<input type="password">`.
7. `type: array` of `string` → a single `<input type="text">`; tell the user values are comma-separated and split them on submit.
8. `type: object` or array-of-object input → a `<textarea>` for raw JSON, parsed with `JSON.parse` on submit. These are rare in inputs; keep it simple.
9. Everything else (`type: string`, `control_type: text`) → `<input type="text">`.

For every control: use `name` as the `name` attribute, `label` as the label text, add `required` and a red asterisk when `optional` is `false`, and put `hint` in both the `placeholder` and a small helper line under the field.

### Step 2: Build the arguments payload

On submit, read each field from `FormData` and build the `argumentsPayload` object keyed by `name`:

- Text/select → `String(value).trim()`.
- Integer → `parseInt(value, 10)`; number → `parseFloat(value)`.
- Boolean → the checkbox's `.checked`.
- Array-of-string → `value.split(',').map(s => s.trim()).filter(Boolean)`.
- Object/JSON textarea → `JSON.parse(value)`.

**Omit optional fields that are empty** so the tool receives a clean payload. Required fields are guarded by HTML `required` + `form.checkValidity()`.

### Step 3: Read the tool result robustly

The result can arrive as `structuredContent` (preferred, matches the output schema) or as JSON text inside `content[]`. The exact shape Workato returns is not always known ahead of time, so never assume it. Always extract with this helper and always render a collapsible **raw response** panel so the real payload is visible for debugging:

```js
function extractData(result) {
  if (!result) return {};
  if (result.structuredContent) return result.structuredContent;
  const text = result.content?.find(c => c.type === 'text')?.text;
  if (text) { try { return JSON.parse(text); } catch (e) { return {message: text}; } }
  return {};
}
```

### Step 4: Render the result with a self-diagnosing view

Do not hard-code the output schema's field names into the render path. The output schema tells you what to *expect*, but build the renderer to adapt to whatever actually arrives. Render in this order:

1. **Error banner** — if a top-level `error` or `message` key is a truthy non-array, show it in a red banner.
2. **Summary chips** — every top-level scalar field (string/number/boolean), shown as chips. This covers `total_count`, `page`, `has_more`, and anything else without naming them.
3. **Table** — find the collection generically: the first top-level array of objects, else the first array, else the object itself as a single row (ignoring error/message-only objects). Derive columns from the union of scalar keys across the first rows, surfacing common keys (`name`, `full_name`, `title`, `id`, `description`, `status`, `language`) first and capping at 8. Build the table header dynamically from those columns.
4. **Empty state** — if no rows resolve, show an empty-state line (the banner and raw panel still tell the story).

Per-cell rules: values starting `http(s)://` become links; booleans become Yes/No; keys ending `_at`/`_date`/`date`/`time` are date-formatted; `null`/empty becomes a dash; everything else is escaped text. Add a free-text filter and click-to-sort on every column. This generic renderer means the same code works whether the tool returns `{repositories:[...]}`, a single object, or an unexpected shape.

### Step 5: Wire the SDK (match the chosen pattern)

Set the handlers **before** `await app.connect();`. Wire to the pattern picked in the App behaviour section:

- **Result-viewer (default):** `app.ontoolresult = (result) => render(extractData(result));` so the model's initial result shows immediately. Any form/refine controls call `callServerTool` and re-render.
- **Write + refresh:** `app.ontoolresult` renders the queue; row buttons call the app-only write tool then re-fetch (see the variant section and `assets/example_app_actions.html`).
- **Form-first:** `app.ontoolresult = () => {};` (ignore the initial result, open blank), `if ('ontoolinput' in app) app.ontoolinput = (params) => prefill(params?.arguments);` to pre-fill, and render only on submit.

In all cases the submit/refresh handler does `const result = await app.callServerTool({ name: '<tool name>', arguments: payload });` then `render(extractData(result));`, wrapped in `try/catch/finally` with the button disabled and a busy label.

Keep the Workato CDN import line exactly as below. It is the Workato-specific bundle, not the bare npm path:

```js
import {App} from 'https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js';
```

### Step 6: Assemble and deliver

Produce one `.html` file with: the two-line "copy into AI tools" comment header, `<head>` with the Tailwind browser CDN, the form, the results markup (error banner, summary, dynamic table, empty state, raw-response `<details>`), and the inline `<script type="module">`. Save it as `<tool_name>_app.html` in the project folder and present it. Tell the user which behaviour pattern is in effect (form-first or result-viewer), to paste it into the App code editor, and to link it to the tool. Per project rules, show the full code as a draft in chat or as a presented file before any further action; do not push anything to Workato.

## Output format

Deliver the generated App as a single HTML file matching this skeleton (the worked example below is a complete instance of it):

```
<!-- copy-into-AI-tools comment header -->
<html lang="en">
  <head> … Tailwind browser CDN … </head>
  <body>
    <form id="tool-form"> … one control per input field … <button type="submit"> … </form>
    <div id="error-banner" hidden> … </div>
    <div id="results" hidden>
      <div id="summary"></div>
      <table><thead id="results-head"></thead><tbody id="results-body"></tbody></table>
    </div>
    <p id="empty-state" hidden>No results found.</p>
    <details id="raw-wrap" hidden><summary>Raw response</summary><pre id="raw"></pre></details>
    <script type="module">
      import {App} from 'https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js';
      // extractData, generic render (findRows/deriveColumns/fmtCell), raw panel
      // form-first: ontoolresult no-op, ontoolinput prefill, render on submit -> callServerTool
    </script>
  </body>
</html>
```

## Worked example

Input: the GitHub `search_repositories` tool. Input schema has `query` (required text), `per_page`/`page` (integer), `order`/`sort` (string with enum options in the hint). Output schema has `error`, `total_count`, `repositories` (array of object), `has_more`, `page`, `per_page`. The enum hints become dropdowns and the integers become number inputs. The App is **form-first**: it opens blank, pre-fills from the model's initial input where possible, and renders only after the user clicks Search. The renderer is **generic** (it auto-detects the `repositories` array, derives columns, and shows a raw-response panel), so it also copes if the tool returns a single object or an unexpected shape.

This is the exact code this skill should produce for that schema:

```html
<!-- If needed, copy the code and paste it into AI tools like Claude, Cursor, or ChatGPT. -->
<!-- Ask them to update it directly. Example: "Replace user_name with employee_name." -->
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Search Repositories</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
  </head>
  <body class="min-h-screen bg-slate-50 text-slate-800 antialiased">
    <div class="mx-auto w-full max-w-5xl px-4 py-10">
      <!-- Input form: one control per input-schema field -->
      <form id="tool-form" class="rounded-2xl border border-blue-500 bg-white p-8 shadow-sm sm:p-10">
        <header class="mb-8">
          <h1 class="text-3xl font-bold tracking-tight text-slate-900">Search Repositories</h1>
          <p class="mt-2 text-base text-slate-500">Enter your search criteria, then select Search to run the query.</p>
        </header>

        <div class="grid gap-7 sm:grid-cols-2">
          <label class="block sm:col-span-2">
            <span class="mb-2 block text-base font-bold text-slate-800">
              Query <span class="font-bold text-red-500">*</span>
            </span>
            <input
              class="w-full rounded-lg border border-slate-300 px-4 py-3 text-base text-slate-900 outline-none transition placeholder:text-slate-400 focus:border-blue-500 focus:ring-2 focus:ring-blue-200"
              name="query" type="text" required
              placeholder="Search query using GitHub search syntax." />
            <span class="mt-1.5 block text-sm text-slate-400">Search query using GitHub search syntax. The query contains one or more search keywords and qualifiers to search for repositories on GitHub.</span>
          </label>

          <label class="block">
            <span class="mb-2 block text-base font-bold text-slate-800">Per page</span>
            <input
              class="w-full rounded-lg border border-slate-300 px-4 py-3 text-base text-slate-900 outline-none transition placeholder:text-slate-400 focus:border-blue-500 focus:ring-2 focus:ring-blue-200"
              name="per_page" type="number" step="1" placeholder="30" />
            <span class="mt-1.5 block text-sm text-slate-400">The number of results per page (max 100, default 30)</span>
          </label>

          <label class="block">
            <span class="mb-2 block text-base font-bold text-slate-800">Page</span>
            <input
              class="w-full rounded-lg border border-slate-300 px-4 py-3 text-base text-slate-900 outline-none transition placeholder:text-slate-400 focus:border-blue-500 focus:ring-2 focus:ring-blue-200"
              name="page" type="number" step="1" placeholder="1" />
            <span class="mt-1.5 block text-sm text-slate-400">The page number of the results to fetch (default: 1)</span>
          </label>

          <label class="block">
            <span class="mb-2 block text-base font-bold text-slate-800">Sort</span>
            <select
              class="w-full rounded-lg border border-slate-300 bg-white px-4 py-3 text-base text-slate-900 outline-none transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200"
              name="sort">
              <option value="">Default</option>
              <option value="stars">stars</option>
              <option value="forks">forks</option>
              <option value="help-wanted-issues">help-wanted-issues</option>
              <option value="updated">updated</option>
            </select>
            <span class="mt-1.5 block text-sm text-slate-400">Sorts the results by stars, forks, help-wanted-issues, or how recently items were updated.</span>
          </label>

          <label class="block">
            <span class="mb-2 block text-base font-bold text-slate-800">Order</span>
            <select
              class="w-full rounded-lg border border-slate-300 bg-white px-4 py-3 text-base text-slate-900 outline-none transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200"
              name="order">
              <option value="">Default</option>
              <option value="desc">desc</option>
              <option value="asc">asc</option>
            </select>
            <span class="mt-1.5 block text-sm text-slate-400">Highest matches first (desc) or lowest (asc). Ignored unless you provide sort. Default: desc.</span>
          </label>
        </div>

        <button
          class="mt-8 w-full rounded-lg bg-slate-800 py-4 text-base font-semibold text-white transition hover:bg-slate-700 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-slate-800 disabled:opacity-60"
          type="submit">Search</button>
      </form>

      <!-- Results: hidden until the user runs a search -->
      <div id="error-banner" class="mt-6 hidden rounded-lg border border-red-300 bg-red-50 px-4 py-3 text-sm text-red-700"></div>

      <div id="results" class="mt-6 hidden">
        <div id="summary" class="mb-4 flex flex-wrap gap-2"></div>
        <div class="overflow-x-auto rounded-2xl border border-slate-200 bg-white shadow-sm">
          <input id="filter" type="text" placeholder="Filter results..."
            class="w-full border-b border-slate-200 px-4 py-3 text-sm outline-none focus:border-blue-500" />
          <table class="w-full text-left text-sm">
            <thead id="results-head" class="bg-slate-100 text-xs uppercase tracking-wide text-slate-500"></thead>
            <tbody id="results-body" class="divide-y divide-slate-100"></tbody>
          </table>
        </div>
      </div>

      <p id="empty-state" class="mt-6 hidden text-center text-sm text-slate-400">No results found.</p>

      <!-- Raw response: shown after a search so the exact payload shape is visible -->
      <details id="raw-wrap" class="mt-4 hidden rounded-lg border border-slate-200 bg-white p-4 text-sm">
        <summary class="cursor-pointer font-medium text-slate-600">Raw response</summary>
        <pre id="raw" class="mt-2 overflow-x-auto whitespace-pre-wrap text-xs text-slate-500"></pre>
      </details>
    </div>

    <script type="module">
      // Most cases, do not modify this import line (Workato MCP Apps SDK bundle).
      import {App} from 'https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js';

      (async () => {
        const app = new App({name: 'Search Repositories', version: '1.0.0'});
        const form = document.getElementById('tool-form');
        const submitButton = form.querySelector('button[type="submit"]');
        const errorBanner = document.getElementById('error-banner');
        const resultsEl = document.getElementById('results');
        const summaryEl = document.getElementById('summary');
        const headEl = document.getElementById('results-head');
        const bodyEl = document.getElementById('results-body');
        const emptyState = document.getElementById('empty-state');
        const filterEl = document.getElementById('filter');
        const rawWrap = document.getElementById('raw-wrap');
        const rawEl = document.getElementById('raw');

        let rows = [];
        let columns = [];
        let sortKey = null;
        let sortDir = -1;

        function esc(v) {
          return String(v ?? '').replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
        }

        function isScalar(v) {
          return v == null || ['string', 'number', 'boolean'].includes(typeof v);
        }

        function label(k) {
          return String(k).replace(/_/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
        }

        // Read the tool output whether it arrives as structuredContent or as
        // JSON text inside content[]. Returns a plain object.
        function extractData(result) {
          if (!result) return {};
          if (result.structuredContent) return result.structuredContent;
          const text = result.content?.find(c => c.type === 'text')?.text;
          if (text) { try { return JSON.parse(text); } catch (e) { return {message: text}; } }
          return {};
        }

        // Find the collection to tabulate: the first array of objects, else the
        // first array, else the object itself as a single row.
        function findRows(data) {
          if (Array.isArray(data)) return data;
          if (!data || typeof data !== 'object') return [];
          for (const v of Object.values(data)) {
            if (Array.isArray(v) && v.some(x => x && typeof x === 'object')) return v;
          }
          for (const v of Object.values(data)) { if (Array.isArray(v)) return v; }
          if (Object.keys(data).some(k => isScalar(data[k]) && !/^(error|message)$/i.test(k))) return [data];
          return [];
        }

        // Derive scalar columns from the rows, surfacing common useful keys first.
        function deriveColumns(rs) {
          const seen = [];
          for (const r of rs.slice(0, 10)) {
            if (r && typeof r === 'object') {
              for (const k of Object.keys(r)) {
                if (isScalar(r[k]) && !seen.includes(k)) seen.push(k);
              }
            }
          }
          const pri = ['name', 'full_name', 'title', 'id', 'description', 'status', 'state', 'language'];
          seen.sort((a, b) => (pri.indexOf(a) + 1 || 99) - (pri.indexOf(b) + 1 || 99));
          return seen.slice(0, 8);
        }

        function fmtCell(key, val) {
          if (val == null || val === '') return '<span class="text-slate-300">&mdash;</span>';
          if (typeof val === 'boolean') return val ? 'Yes' : 'No';
          const s = String(val);
          if (/^https?:\/\//.test(s)) {
            return `<a href="${esc(s)}" target="_blank" rel="noopener" class="text-blue-600">${esc(s.replace(/^https?:\/\//, ''))}</a>`;
          }
          if (/(_at|_date|date|time)$/i.test(key)) {
            const d = new Date(s);
            if (!isNaN(d)) return esc(d.toLocaleDateString());
          }
          return esc(s);
        }

        function renderRows() {
          const q = filterEl.value.trim().toLowerCase();
          let view = rows.filter(r => !q || JSON.stringify(r).toLowerCase().includes(q));
          if (sortKey) {
            view = [...view].sort((a, b) => {
              const x = a?.[sortKey], y = b?.[sortKey];
              const cmp = (typeof x === 'number' && typeof y === 'number')
                ? x - y : String(x ?? '').localeCompare(String(y ?? ''));
              return cmp * sortDir;
            });
          }
          bodyEl.innerHTML = view.map(r => `
            <tr class="hover:bg-slate-50">
              ${columns.map(c => `<td class="px-4 py-3 align-top text-slate-600">${fmtCell(c, r?.[c])}</td>`).join('')}
            </tr>`).join('');
        }

        // Single render path. Called only after the user runs a search.
        function render(data) {
          rawEl.textContent = JSON.stringify(data, null, 2);
          rawWrap.classList.remove('hidden');

          errorBanner.classList.add('hidden');
          const errKey = Object.keys(data).find(k => /^(error|message)$/i.test(k) && data[k] && !Array.isArray(data[k]));
          if (errKey) {
            errorBanner.textContent = String(data[errKey]);
            errorBanner.classList.remove('hidden');
          }

          summaryEl.innerHTML = Object.entries(data)
            .filter(([k, v]) => isScalar(v) && v != null && v !== '' && !/^(error|message)$/i.test(k))
            .map(([k, v]) => `<span class="rounded-full bg-slate-200 px-3 py-1 text-xs font-medium text-slate-600">${esc(label(k))}: ${esc(typeof v === 'boolean' ? (v ? 'Yes' : 'No') : v)}</span>`)
            .join('');

          rows = findRows(data);
          columns = deriveColumns(rows);

          if (rows.length && columns.length) {
            headEl.innerHTML = `<tr>${columns.map(c => `<th class="cursor-pointer px-4 py-3" data-sort="${esc(c)}">${esc(label(c))} &#8597;</th>`).join('')}</tr>`;
            headEl.querySelectorAll('th[data-sort]').forEach(th => {
              th.addEventListener('click', () => {
                const k = th.dataset.sort;
                sortDir = sortKey === k ? -sortDir : -1;
                sortKey = k;
                renderRows();
              });
            });
            resultsEl.classList.remove('hidden');
            emptyState.classList.add('hidden');
            renderRows();
          } else {
            resultsEl.classList.add('hidden');
            emptyState.classList.remove('hidden');
          }
        }

        filterEl.addEventListener('input', renderRows);

        // Pre-fill the form with whatever the model passed when it opened the App.
        function prefill(args) {
          if (!args) return;
          for (const [k, v] of Object.entries(args)) {
            const el = form.elements.namedItem(k);
            if (el && v != null) el.value = v;
          }
        }

        // FORM-FIRST: ignore the model's initial result so the view stays blank
        // until the user searches. Still set this before connect() per the SDK.
        app.ontoolresult = () => {};
        if ('ontoolinput' in app) app.ontoolinput = (params) => { try { prefill(params?.arguments); } catch (e) {} };
        await app.connect();

        // Run the tool with the form inputs and render the result.
        form.addEventListener('submit', async event => {
          event.preventDefault();
          if (!form.checkValidity()) { form.reportValidity(); return; }
          const fd = new FormData(form);

          const argumentsPayload = {};
          const query = String(fd.get('query') ?? '').trim();
          if (query) argumentsPayload.query = query;
          const perPage = String(fd.get('per_page') ?? '').trim();
          if (perPage) argumentsPayload.per_page = parseInt(perPage, 10);
          const page = String(fd.get('page') ?? '').trim();
          if (page) argumentsPayload.page = parseInt(page, 10);
          const sort = String(fd.get('sort') ?? '').trim();
          if (sort) argumentsPayload.sort = sort;
          const order = String(fd.get('order') ?? '').trim();
          if (order) argumentsPayload.order = order;

          submitButton.disabled = true;
          submitButton.textContent = 'Searching...';
          try {
            const result = await app.callServerTool({
              name: 'search_repositories',
              arguments: argumentsPayload,
            });
            render(extractData(result));
          } catch (err) {
            errorBanner.textContent = 'Tool call failed: ' + (err?.message ?? err);
            errorBanner.classList.remove('hidden');
          } finally {
            submitButton.disabled = false;
            submitButton.textContent = 'Search';
          }
        });
      })();
    </script>
  </body>
</html>
```

## Key principles

- **Use `name`, not `label`, as the object key** in both the form `name` attribute and the data path. Labels are display only.
- **Set `ontoolresult` before `app.connect()`** or the initial result is lost.
- **Read `structuredContent` first**, fall back to parsing `content[]` text.
- **Omit empty optional fields** from the arguments payload; only required fields are guaranteed.
- **Escape all rendered values** to avoid breaking the markup with `<`, `>`, `&`, `"`.
- **Keep it one file.** No external JS/CSS files; Tailwind and the SDK load from CDN only.
- **Keep the Workato CDN import line verbatim** (`app-with-deps.js`); it is not the bare npm import.
- **Do not invent fields.** If the schema lacks an output collection, render whatever `structuredContent` arrives and say the table needs the output schema.
- **An App is launched by a tool call, not by the user.** The model runs the tool to open the App, so a result reaches the chat before the user touches the UI. Default to result-viewer so the App shows that result rather than sitting blank.
- Present the generated file for the user to copy, and do not deploy it to Workato without their go-ahead.

## Example files

- `assets/example_app.html` — form-first search App (input form + generic self-diagnosing results table + raw panel). Build pattern 3.
- `assets/example_app_actions.html` — write + refresh approval queue (result-viewer + Approve/Reject buttons calling an app-only write tool, then re-fetch). Build pattern 2. Read this when the user needs action buttons.
- `assets/example_app_launcher.html` — launcher + app-only worker (a no-input launcher tool opens the App; the form submit calls a hidden app-only worker tool). Read this when the user wants the App to open directly without the model asking for parameters in chat.
