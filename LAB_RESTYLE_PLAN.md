# Lab Restyle Plan — Resiliency Testing Lab

> Working tracker (not part of the lab). Goal: bring README Sections 0–4 up to the polished
> "component" style the Staff SE established in Sections 6–8, so the whole participant lab reads
> as one top-notch, best-of-breed Harness experience. **Scope: README.md only** (FACILITATOR_GUIDE.md excluded).

## Writing rules (apply to every edit)
1. **No em dashes ("—") anywhere.** Use commas, colons, periods, or parentheses. (Em dashes read as AI-written.)
2. **Warm, premium, confident language.** Explain the "why," not just "click here." Best-of-breed tone.
3. **Match Sections 6–8 exactly:** `## <emoji> N. Title` then `### Overview` then `### Value` (bold capability
   bullets) then `### Step N:` (imperative; `| Field | Value | Notes |` tables for inputs; `> [!NOTE]` /
   `> [!TIP]` / `> [!WARNING]` callouts) then `### What was achieved` (bold outcome bullets).
4. **Value bullets are positive capability statements** (never name competitors), matching the SE's voice.
5. Use `<pre>`-boxed values for literal text the participant types, as the SE does.

## Decisions log (my question -> your answer)
- **A. Component kit** -> Approved.
- **B. Section rewrites (Overview / Value / What was achieved)** -> Approved, with rules 1 and 2 above.
- **C. Intro polish** -> (1) Lab-flow table of contents: **YES**, as a table. (2) Conventions note: **NO**.
  (3) "Why Harness" framing: **NO**.
- **D. Staff SE 6–8 issues** -> (1) Fix numbering, including Section 8's duplicate "Step 2": **YES**.
  (2) Fix typos "laveraging" -> "leveraging", "suffic" -> "suffix": **YES**. (3) Template name: real name is
  now **`service-restart`** (the SE renamed ours). Keep as-is, no change.
- **E. Numbering** -> **Contiguous renumber 0–7** (recommended). Fix cross-references very carefully. README only.

## Renumber map (top-level sections)
| Now | Becomes | Emoji | Title |
|-----|---------|-------|-------|
| Step 0 · Deploy your application | 0 | 🚀 | Deploy Your Application (kept light) |
| Step 1 · Open Resiliency Testing | (merged) | — | folded in as first step of Section 1 |
| Step 2 · Discover your services | 1 | 🔍 | Discover Your Services & Map Them |
| Step 3 · Run an auto-generated experiment | 2 | ⚡ | Run an Auto-Generated Experiment |
| Step 4 · Run a ready-made experiment (template) | 3 | 📦 | Run a Ready-Made Experiment (from a Template) |
| Step 5 · Build your own experiment | 4 | 🧪 | Build Your Own Experiment |
| 🔄 6. Resiliency Probes | 5 | 🔄 | Resiliency Probes (number only) |
| 🛡️ 7. Chaos Guard Controls | 6 | 🛡️ | Chaos Guard Controls (number only) |
| ❤️‍🩹 8. Application Delivery Meets Resilience | 7 | ❤️‍🩹 | Application Delivery Meets Resilience (number only) |

## Cross-references to fix (carefully; reword to avoid fragile numbers where cleaner)
- "a step up from Step 3's pod-level probe" -> "a step up from the pod-level probe in the auto-generated experiment"
- "Same loop as Step 3, but from a shared template" -> "The same loop as the auto-generated experiment, now from a shared template"
- "Peek back at Step 4 if you get stuck" -> "Peek back at the previous section if you get stuck"
- "That's exactly the pattern the Step 4 template ships" -> "That's exactly the pattern the ready-made template ships"
- Final grep for any remaining `Step [0-9]` cross-references after edits.

## Drafted Value / What was achieved (approved wording, no em dashes)
### 🔍 1. Discover Your Services & Map Them
- Value: **Auto-discover services so Harness maps your app for you, with no resource IDs to type by hand.** ·
  **A living application map that reflects real services and traffic, not a static picture.** ·
  **The map becomes the launch pad for auto-generated, scored experiments in the next section.**
- What was achieved: **Automatically discovered your app's services on Kubernetes.** ·
  **Built a reusable Application Map that stays current after every run.** ·
  **Created the foundation Harness uses to generate and score experiments.**

### ⚡ 2. Run an Auto-Generated Experiment
- Value: **One click turns a discovered app into a ready-to-run, scored experiment, with no blank canvas.** ·
  **Every generated experiment arrives pre-wired with a probe and a Resilience Score.** ·
  **A fast, confident first win on your own application.**
- What was achieved: **Generated a Pod Delete experiment straight from your application map.** ·
  **Ran it and read an objective Resilience Score in Run History.** ·
  **Inspected the fault and probe on the timeline, with no `kubectl` needed.**

### 📦 3. Run a Ready-Made Experiment (from a Template)
- Value: **Standardized templates let one definition serve many teams, for consistency at scale.** ·
  **Parametric by design, so the same template adapts to any project, namespace, or service through inputs.** ·
  **App-level HTTP probing confirms your page returns `200` before, during, and after the fault.**
- What was achieved: **Imported and ran a shared, parametric experiment template.** ·
  **Validated app-level steady state right through a pod deletion.** ·
  **Experienced the "one definition, many teams" model firsthand.**

### 🧪 4. Build Your Own Experiment
- Value: **A visual, no-code builder for composing faults, targets, probes, and actions.** ·
  **Reusable probe resources let you attach a managed `svc-health-check` rather than redefining checks.** ·
  **The steady-state pattern, probing before, during, and after, proves the app truly held.**
- What was achieved: **Built a Pod Delete experiment from scratch in the visual editor.** ·
  **Targeted your backend deployment precisely by kind, namespace, workload, and labels.** ·
  **Attached a reusable HTTP probe so the Resilience Score reflects real application health.**

## Intro: Lab-flow table of contents (the only intro change)
A table near the top listing the eight sections with emoji and a one-line outcome each, so participants
see the arc: Deploy -> Discover -> Auto-generate -> Template -> Build -> Probes -> Guard -> Delivery.

## Section 0 (kept light)
Restyle heading to `## 🚀 0. Deploy Your Application`, add a one-line Overview and a `> [!NOTE]` prerequisite
note. No Value or What-was-achieved block.

## Staff SE 6–8 fixes (approved)
- Renumber headings 6 -> 5, 7 -> 6, 8 -> 7 (leading number only, content untouched).
- Section 8 (now 7): second "### Step 2" (Inject chaos into pipelines) -> "### Step 3".
- Typos: "laveraging" -> "leveraging"; "suffic" -> "suffix".

## Execution checklist
- [x] Intro lab-flow TOC table
- [x] Section 0 restyle (light)
- [x] Section 1 Discover & Map (merge old Step 1; Value/WWA; map-creation table)
- [x] Section 2 Auto-generated experiment (Value/WWA; callouts)
- [x] Section 3 Template (Value/WWA; infra table; Hypothesis -> NOTE; AZ -> TIP; keep `service-restart`)
- [x] Section 4 Build your own (Value/WWA; target + probe tables; best-practice -> TIP)
- [x] Renumber SE headings 6/7/8 -> 5/6/7
- [x] Section 7 duplicate Step 2 -> Step 3
- [x] Typos leveraging / suffix
- [x] Fix all cross-references (Step 3 probe ref, "previous section", "ready-made template ships")
- [x] Verify: no `Step [0-9]` strays, no "—" in my content, clean 0-7 heading sequence

## Open flags for Neus (SE-authored text, not changed without approval)
- **Line ~377** (Section 5 "What was achieved"): contains an em dash ("threshold — in this case"). This is the one AI-tell you flagged, but it's the SE's text, so left as-is pending your OK. One-char swap to a comma or colon.
- **SE sub-tables use `##`** (`## Variable 1`, `## What`, `## Where`, `## Which`, `## Using`) which renders them as top-level sections in the TOC. Cosmetic; would read better as `###`/`####`. Left untouched (SE territory).
