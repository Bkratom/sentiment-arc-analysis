---
name: sentiment-arc-analysis
description: Multi-phase skill for analyzing customer/user sentiment across a conversation dataset. Produces a self-contained HTML report with an arc chart, sentiment breakdown bars, verbatim quote examples, and hostile moment disclosure. Iterates with the user at each critical decision point before generating output.
---

# Sentiment Arc Analysis Skill

## Purpose

Analyze how customer or user sentiment evolves across a conversation flow — specifically at a defined engagement stage (e.g., layers 3–4 of a pushback system, round 2 of a negotiation, step 3 of an onboarding sequence). Produce a self-contained, publication-quality HTML report with a sentiment arc chart, breakdown bars, verbatim customer quotes, key finding, and honest hostile moment disclosure.

---

## Terminology Convention

This skill uses **"engagement stage"** as the generic internal concept. Before generating any output, detect what the user calls it — "layer," "step," "round," "phase," "tier," etc. — and use **their word** in all labels, chart annotations, headings, and copy throughout the entire report. Do not output "engagement stage" in the final HTML.

---

## Workflow Overview

```
Phase 1: Setup          → 3 user questions + 2 optional
Phase 2: Data Connect   → fetch/read, auto-detect schema
Phase 3: Confirm Gates  → 3 AI-proposes / user-confirms steps
Phase 4: Analysis       → score, aggregate, find quotes
Phase 5: Threshold Gate → show hostile threshold, iterate
Phase 6: HTML Output    → full self-contained report
```

---

## Phase 1: Setup Questions

Ask these questions **in sequence**. Do not batch them — wait for each answer before continuing.

### Q1 — Subject of Analysis
> "What are we analyzing? Give me one sentence: what is happening in these conversations, and what stage of the conversation are we focused on?
>
> Example: 'We're analyzing customer responses at layers 3 and 4 of a refund pushback flow — the point where a free access offer or partial refund has been presented.'"

Store as: `analysis_subject` (used in masthead and chart title)

### Q2 — Data Source
> "What's your data source?
> - **Airtable** — I'll need your API token, base ID, table ID, and view ID
> - **JSON** — paste the data or give me a file path
> - **CSV** — paste the data or give me a file path"

Store as: `data_source_type` + connection details

### Q3 — Optional System Context
> "Do you have any documentation, SOPs, product descriptions, or system explanations I should read before analyzing? This helps me correctly identify engagement stages, field meanings, and terminology.
>
> You can paste text, give me a file path, or skip this."

If provided, read it before proceeding. Extract:
- What the engagement stages mean
- What terms the user uses for layers/steps/rounds
- Any domain-specific signal words (e.g., "Refund Acceptance," "partial offer," "pushback")

### Q4 — Quote Count (ask after analysis, before generation)
> "How many verbatim customer quotes do you want shown in the report? Default is 6. Any beyond that will be in a collapsed expandable section."

Store as: `quote_count` (default: 6)

---

## Phase 2: Data Connect

### Airtable
```
GET https://api.airtable.com/v0/{base_id}/{table_id}?view={view_id}&pageSize=100
Authorization: Bearer {token}

Paginate using offset until all records fetched.
```

### JSON / CSV
Read from path or parse pasted content. Normalize to a list of record objects.

---

## Phase 3: Confirmation Gates

After fetching data, before scoring anything, confirm three things with the user. **Do not proceed past each gate without explicit confirmation.**

---

### Gate A — Conversation Field Identification

Inspect the schema. Look for fields that contain:
- Long free-text strings
- Multiple speaker turns or timestamped messages
- Names, dates, and message bodies interleaved

Present your conclusion:

> "I think the conversation text lives in the field **`[field_name]`**. Here's a sample of what it looks like:
>
> ```
> [first 300 characters of a sample record's value]
> ```
>
> Is this the right field? Yes / No / It's in a different field called ___"

If wrong: ask which field. Re-inspect. Try again.

---

### Gate B — Engagement Stage Mapping

Using the user's `analysis_subject` (Q1) and any context from Q3, identify which records are "in scope" — i.e., belong to the engagement stage being analyzed.

Inspect the data for fields that could signal stage depth:
- Repeated action categories (e.g., `Category` field with multiple "Refund" entries)
- Status fields (e.g., `RefundStatus`, `Stage`, `Step`)
- Count fields (e.g., `LayerDepth`, `RoundNumber`)
- Subflow or action log fields with repeating patterns

Form a hypothesis. Present it:

> "Based on your description ('*[analysis_subject]*') and the data schema, I believe the **[user's term for stage]** being analyzed maps to:
>
> - **Field:** `[field_name]`
> - **Rule:** `[plain English filter, e.g., 'records where the Refund category appears 3 or more times in the Category field']`
> - **Records in scope:** [N] of [total]
>
> Does this correctly identify the **[user's term]** you want to analyze? Yes / No / Here's what's actually different: ___"

If wrong: incorporate the correction, re-run the filter, re-confirm. Repeat until confirmed.

---

### Gate C — Ticket Link Field

Look for a field containing URLs pointing to an external ticketing system (Help Scout, Zendesk, Intercom, Jira, etc.).

> "I found ticket links in the field **`[field_name]`** — they look like: `[sample URL]`. I'll use these for the ↗ links on quote cards and hostile moment disclosures.
>
> Is that correct? (Or: I didn't find a ticket URL field — do you have one?)"

If none: proceed without links. Quote cards will omit the ↗ element.

---

## Phase 4: Analysis

Run this after all three gates are confirmed.

### 4.1 — Extract Customer Messages

For each in-scope record:
1. Read the conversation field value
2. Parse it into individual speaker turns:
   - Identify the pattern (timestamps, speaker names, line breaks, delimiters)
   - Separate customer messages from agent/system/AI messages
   - Number customer messages sequentially: msg 1, msg 2, msg 3...
3. Store: `[record_id, customer_name, message_number, message_text, ticket_url]`

**Parsing heuristic:** If the conversation is a raw text block, look for lines beginning with a customer name or email followed by a date/time stamp. Everything until the next timestamp is that speaker's message. Filter to only customer turns (exclude AI, agent, support, system lines).

### 4.2 — Sentiment Scoring

Score each customer message:

| Score | Label | Signals |
|-------|-------|---------|
| +1 | Polite | Thank you, appreciate, understand, will rejoin, positive intent |
| 0 | Neutral / Direct | Matter-of-fact, no emotional signal, transactional |
| -1 | Firm / Frustrated | Insisting, repeated request, mild dissatisfaction, "I already said" |
| -2 | Hostile / Escalating | Threats (BBB, dispute, bank, lawyer), all-caps anger, personal attack, ultimatum |

Score based on the overall tone of the message, not individual words. A polite "thank you but no" is +1, not 0.

Store: `[record_id, message_number, score, label, message_text, customer_name, ticket_url]`

### 4.3 — Aggregate for Chart

For each message position (1, 2, 3, 4...):
- Collect all scores at that position across all in-scope tickets
- Compute average score
- This becomes the arc chart's average trend line

Also compute per-ticket score sequences (for individual gray lines on the chart).

Cap chart x-axis at the message position where fewer than 3 tickets remain (avoids noise at the tail).

### 4.4 — Tier Breakdown

From all messages across all in-scope records, count:
- Polite (+1): N, %
- Neutral (0): N, %
- Firm (-1): N, %
- Hostile (-2): N, %

### 4.5 — Select Quotes

Rank messages for the quote mosaic:
- Prioritize: polite messages at later message positions (these are the most compelling — still courteous deep in the flow)
- Include at least 1 neutral, 0 hostile in the main mosaic
- Prefer messages with customer names (more credible than anonymous)
- Prefer messages ≥ 15 words (more substantive)

Select `quote_count` total (default 6). If more quotes are available and user requested them, select up to 12 additional for the collapsed expand section.

---

## Phase 5: Hostile Threshold Gate

After scoring is complete, show the user what triggered as hostile:

> "I auto-set the hostile threshold at **score ≤ -2** (explicit threats, demands, or escalation language). Here's what triggered:
>
> | Ticket | Customer | Msg # | Text |
> |--------|----------|-------|------|
> | [link] | [name] | [n] | [first 100 chars] |
> ...
>
> **[N] hostile moments** across [total] messages in the engagement stage.
>
> Does this threshold feel right? Would you adjust it — tighter (only score -2) or broader (include score -1 as 'hostile')?"

Adjust if requested. Re-run tier breakdown with new threshold.

---

## Phase 6: HTML Generation

Generate a full self-contained HTML report using the design system below.

### Design System

```css
Fonts: Playfair Display (headings, numbers), Source Serif 4 (body prose), Inter (labels, UI)
Colors:
  --ink: #1a1a1a        --paper: #faf8f5      --paper-warm: #f5f0e8
  --accent: #c45d3e     --rule: #d4cfc7       --rule-light: #e8e4dc
  --green: #3d7a4a      --green-light: #e8f2ea
  --amber: #b8860b      --amber-light: #fdf3dd
  --red: #b44           --red-light: #fce8e8
  --blue: #3a6fa0       --blue-light: #e6f0fa
  --ink-muted: #8a8a8a  --ink-light: #4a4a4a
```

Load from Google Fonts:
```
https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,400;0,700;0,900;1,400&family=Source+Serif+4:ital,opsz,wght@0,8..60,300;0,8..60,400;0,8..60,600;1,8..60,400&family=Inter:wght@300;400;500;600;700&display=swap
```

---

### Report Structure

#### 1. Masthead
```
[Publication label: system/product name if known, else "Sentiment Analysis"]
[Main title: derived from analysis_subject]
[Subtitle: one sentence framing what this analysis shows]
[Meta line: Sample Period · N Records · Data Source · [user's stage term] Analyzed]
```

#### 2. Sentiment Arc Chart

SVG, `viewBox="0 0 560 280"`, rendered via JavaScript (same pattern as below).

**Chart zones (background fills):**
- y > 0 to +1.5: green tint (polite zone)
- y = -0.5 to 0: neutral white
- y = -1 to -0.5: amber tint (friction zone)
- y < -1 to -2.5: red tint (hostile zone)

**Data series:**
- Individual ticket lines: `stroke: #d4cfc7`, `stroke-width: 1`, `opacity: 0.6`
- Average trend: `stroke: var(--accent)`, `stroke-width: 2.5`, bold
- Hostile moment markers: filled red circles at the specific (msg_position, score) coordinate
- Axis: x = message positions 1–N, y = -2.5 to +1.5

**Y-axis labels:** +1 Polite, 0 Neutral, -1 Firm, -2 Hostile

**X-axis labels:** Message 1, Message 2, ... (use user's vocabulary: "Response 1" if they said "response")

**Chart title:** `"Sentiment by Message Position ([N] [user's stage term] tickets)"`

#### 3. Sentiment Breakdown (two-column layout)

**Left:** Inline bar chart, one row per tier:
```
Polite / Warm     [██████████░░░░░░░░░░] 6  (33%)
Neutral / Direct  [████████████████░░░░] 9  (50%)
Firm / Frustrated [███░░░░░░░░░░░░░░░░░] 2  (11%)
Hostile           [█░░░░░░░░░░░░░░░░░░░] 1  (6%)
```

Colors: green, blue, amber, red

**Right:** Key finding callout box (green left-border):
```
KEY FINDING
[One sentence auto-generated from the data, e.g.:
"78% of all messages at [stage] are neutral or polite.
The conversation is not deteriorating — customers are
rationally persistent, not hostile."]
```

Key finding generation rule: lead with the % neutral+polite at the stage, characterize the emotional quality (persistence vs. hostility), note the hostile count if < 10% ("only N hostile moment[s] in [total] messages").

#### 4. Quote Mosaic

Label: `ACTUAL CUSTOMER LANGUAGE AT [USER'S STAGE TERM UPPERCASED] — VERBATIM`

Grid: `repeat(3, 1fr)`, gap `1rem`

**Quote card structure:**
```html
<div style="border-top: 3px solid [tier color]; border: 1px solid rule; padding: 1.25rem; background: white; border-radius: 6px;">
  <div style="Playfair Display italic, 1rem">"[verbatim quote]"</div>
  <div style="Inter 0.65rem uppercase muted">[Customer First Name] [Last Initial]. — Message [N] of [total]</div>
  [if ticket_url]: <a href="[url]" target="_blank" style="Inter 0.62rem muted">↗ #[ticket_id]</a>
  <span class="badge badge-[tier]">[TIER LABEL]</span>
</div>
```

Badge colors: green=POLITE, blue=NEUTRAL, amber=FIRM, red=HOSTILE

**If quote_count < total available:** Add a collapsed expand section below the main grid:
```html
<details style="margin-top: 1rem;">
  <summary style="Inter 0.7rem cursor-pointer">+ [N] more quotes — click to expand</summary>
  <div style="grid same as above; margin-top: 1rem;">
    [additional quote cards]
  </div>
</details>
```

#### 5. Hostile Moments Disclosure

Collapsed by default. Red-bordered aside.

```html
<details style="margin-top: 2rem;">
  <summary style="...">
    The [N] Hostile Moment[s] — Full Disclosure ([N] of [total in-scope] tickets)
  </summary>
  <div class="aside-box" style="border-left-color: var(--red); background: var(--red-light); margin-top: 1rem;">
    <div class="aside-title" style="color: var(--red)">The [N] Hostile Moment[s] — Full Disclosure</div>
    [For each hostile message:]
    <p>
      <strong><a href="[ticket_url]" style="color:var(--red);border-bottom:1px solid var(--red);">
        Ticket #[id]
      </a> ([Customer Name], msg [N]):</strong>
      "[verbatim message]" — [1-sentence context: what stage, what happened next, outcome]
    </p>
    [closing statement: "Both/All [N] instances are documented and real. None resulted in [worst possible outcome — infer from context]. They represent [N] of [total] total tickets."]
  </div>
</details>
```

#### 6. Footer
```
Report Generated [today's date] · Data Source: [source] · Sample: [N] Records · [user's stage term] Analyzed · [system/product name if known]
```

---

## JavaScript Patterns

### Sentiment Arc Chart (rendered inline)
```javascript
(function() {
  var svg = document.getElementById('sentiment-chart');
  if (!svg) return;
  var W=560, H=280, padL=52, padR=20, padT=24, padB=36;
  var chartW=W-padL-padR, chartH=H-padT-padB;
  var maxMsg=[N], yMin=-2.5, yMax=1.5; // set N to actual max message position

  function xPos(msg) { return padL + ((msg-1)/(maxMsg-1))*chartW; }
  function yPos(score) { return padT + (1-(score-yMin)/(yMax-yMin))*chartH; }

  var ns='http://www.w3.org/2000/svg';
  function el(tag, attrs, parent) {
    var e=document.createElementNS(ns,tag);
    for(var k in attrs) e.setAttribute(k,attrs[k]);
    if(parent) parent.appendChild(e); return e;
  }

  // Zone backgrounds
  el('rect',{x:padL,y:yPos(0),width:chartW,height:yPos(-1)-yPos(0),fill:'rgba(184,134,11,0.06)'},svg);
  el('rect',{x:padL,y:yPos(-1),width:chartW,height:yPos(-2.5)-yPos(-1),fill:'rgba(187,68,68,0.06)'},svg);
  el('rect',{x:padL,y:padT,width:chartW,height:yPos(0)-padT,fill:'rgba(61,122,74,0.04)'},svg);

  // Zero line
  el('line',{x1:padL,y1:yPos(0),x2:padL+chartW,y2:yPos(0),stroke:'#d4cfc7','stroke-width':'1'},svg);

  // Individual ticket lines (inject actual data as arrays of [msg_pos, score] pairs)
  var ticketData = [/* [[1,score],[2,score],...] per ticket */];
  ticketData.forEach(function(pts) {
    if(pts.length<2) return;
    var d=pts.map(function(p,i){return (i?'L':'M')+xPos(p[0])+' '+yPos(p[1]);}).join(' ');
    el('path',{d:d,fill:'none',stroke:'#d4cfc7','stroke-width':'1.2',opacity:'0.7'},svg);
  });

  // Average trend line
  var avgData = [/* [msg_pos, avg_score] */];
  if(avgData.length>1){
    var d=avgData.map(function(p,i){return (i?'L':'M')+xPos(p[0])+' '+yPos(p[1]);}).join(' ');
    el('path',{d:d,fill:'none',stroke:'#c45d3e','stroke-width':'2.5','stroke-linecap':'round','stroke-linejoin':'round'},svg);
  }

  // Hostile moment markers (red dots)
  var hostilePoints = [/* [msg_pos, score] */];
  hostilePoints.forEach(function(p){
    el('circle',{cx:xPos(p[0]),cy:yPos(p[1]),r:'5',fill:'#b44',stroke:'white','stroke-width':'1.5'},svg);
  });

  // Y-axis labels
  [{s:1,l:'Polite'},{s:0,l:'Neutral'},{s:-1,l:'Firm'},{s:-2,l:'Hostile'}].forEach(function(item){
    el('text',{x:padL-6,y:yPos(item.s)+4,
      'font-family':'Inter,sans-serif','font-size':'9','text-anchor':'end',fill:'#8a8a8a'},svg).textContent=item.l;
  });

  // X-axis labels (Message 1, 2, ...)
  for(var m=1;m<=maxMsg;m++){
    el('text',{x:xPos(m),y:H-8,'font-family':'Inter,sans-serif','font-size':'9','text-anchor':'middle',fill:'#8a8a8a'},svg).textContent='Msg '+m;
  }
})();
```

### Inline Bar Animation (IntersectionObserver)
```javascript
var observer = new IntersectionObserver(function(entries) {
  entries.forEach(function(entry) {
    if(entry.isIntersecting) {
      entry.target.classList.add('visible');
      entry.target.querySelectorAll('.inline-bar-fill, .metric-card-fill').forEach(function(bar) {
        var w = bar.getAttribute('data-width');
        setTimeout(function(){ bar.style.width = w + '%'; }, 100);
      });
    }
  });
}, { threshold: 0.15 });
document.querySelectorAll('.fade-in').forEach(function(el){ observer.observe(el); });
```

---

## Required CSS Classes

Include in `<style>`:
```css
.fade-in { opacity: 0; transform: translateY(12px); transition: opacity 0.5s ease, transform 0.5s ease; }
.fade-in.visible { opacity: 1; transform: none; }
.inline-bar { height: 6px; background: #e8e4dc; border-radius: 3px; overflow: hidden; margin-top: 0.35rem; }
.inline-bar-fill { height: 100%; width: 0; transition: width 1s ease; border-radius: 3px; }
.aside-box { border-left: 3px solid var(--accent); background: var(--paper-warm); padding: 1.25rem 1.5rem; border-radius: 0 6px 6px 0; }
.aside-title { font-family: 'Inter', sans-serif; font-size: 0.65rem; font-weight: 700; letter-spacing: 0.2em; text-transform: uppercase; color: var(--accent); margin-bottom: 0.5rem; }
.badge { font-family: 'Inter', sans-serif; font-size: 0.6rem; font-weight: 600; letter-spacing: 0.12em; text-transform: uppercase; padding: 0.2rem 0.55rem; border-radius: 3px; }
.badge-green { background: var(--green-light); color: var(--green); }
.badge-blue  { background: var(--blue-light);  color: var(--blue);  }
.badge-amber { background: var(--amber-light); color: var(--amber); }
.badge-red   { background: var(--red-light);   color: var(--red);   }
details > summary { cursor: pointer; font-family: 'Inter', sans-serif; font-size: 0.75rem; font-weight: 600; color: var(--ink-muted); letter-spacing: 0.05em; list-style: none; }
details > summary::-webkit-details-marker { display: none; }
details[open] > summary { color: var(--ink); }
```

---

## Output File

Name the output file: `sentiment-arc-[slugified analysis_subject]-[YYYY-MM-DD].html`

Example: `sentiment-arc-l3-l4-pushback-responses-2026-03-23.html`

Save to the user's Desktop or current working directory unless they specify otherwise.

---

## Error Handling

- If fewer than 5 in-scope records are found after Gate B: warn the user. "I only found [N] records matching that engagement stage — the chart may not be meaningful at this sample size. Continue anyway?"
- If no customer messages can be extracted from the conversation field: show a 5-record sample and ask the user to describe the format
- If no ticket link field is found: proceed without links, note it in the footer as "Ticket links: not available"
- If hostile threshold produces 0 results: note it clearly — "No hostile moments detected at threshold ≤ -2" — and skip the hostile section rather than generating an empty one
