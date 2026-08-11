# Facilitator Guide - Resiliency Testing Lab (Harness)

> **This is the SE / facilitator guide**: the "why", the talk-track, the positioning, and the setup.
> **Participants follow [`README.md`](README.md)**: the clean, step-by-step lab. A first draft of those
> steps already exists there; the `TODO` / `⟨verify⟩` markers in this guide track what still needs
> confirming live in the workshop tenant (see [Second-pass checklist](#second-pass-checklist)).
>
> **Two files, two audiences:**
>
> | File | Audience | Contents |
> | --- | --- | --- |
> | `FACILITATOR_GUIDE.md` (this file) | You / the SE | Objectives, talk-track, positioning, facilitation notes, setup, sources |
> | [`README.md`](README.md) | Workshop participants | The hands-on steps they follow |
>
> The Modules here map to the participant Steps in [`README.md`](README.md), offset by one (the participant guide adds an opening "Step 1 · Open Resiliency Testing"): Module 1 ↔ Step 2, Module 3 ↔ Step 4, … Module 10 ↔ Step 11. See [Appendix A](#appendix-a--timing-summary).

## About this lab

A hands-on workshop that walks a customer through **Harness Resiliency Testing** on Kubernetes:
auto-discover their services, run and build chaos experiments, quantify resilience with a score,
govern it safely, and gate a canary deployment on it - the full product, not just chaos.

- **Audience:** platform / SRE / DevOps engineers. Assumes CI/CD familiarity, **new to chaos engineering.**
- **Environment:** Kubernetes, **Prometheus** for metrics, reusing the DevSecOps lab infra (simple app, few services). A "fake it" workshop, not a full POV.
- **Format:** mostly hands-on, participants working in their own project; a few demo-only modules.
- **Duration:** two tiers - a **~2h core path** plus **optional extension modules**. Any SE can flex to the audience and clock.

> **Naming note:** Harness's official spelling is "Resili**ence** Testing"; internally we say "Resiliency Testing." The module now spans **Chaos Testing + Load Testing + DR Testing**. We cover Chaos Testing (core) + Load Testing (demo) and skip DR (demo only if asked).

## Learning objectives

By the end, participants can:
1. Auto-discover Kubernetes services and build an Application Map.
2. Explain the model: **Fault + Probe + Action = Experiment**, and what drives the Resilience Score.
3. Reuse a standardized **Experiment Template** and read a run (timeline, logs, probes, score, report).
4. Build their own experiment with an HTTP + **Prometheus (PromQL)** steady-state probe.
5. Govern experiments with **Chaos Guard** (deny fault types, block business-critical windows).
6. **Gate a canary deployment** on resilience using a Chaos step + Continuous Verification.
7. Articulate why Harness beats point tools (governance, single-platform, chaos-under-load).

## The one mental model (plant this in Module 0, reinforce throughout)

| Building block | What it is | Effect on Resilience Score |
| --- | --- | --- |
| **Fault** | The attack - pod delete, CPU/memory hog, network latency/loss, DNS… | none directly |
| **Probe** | The check / hypothesis - is steady state holding? (HTTP, Prometheus, k8s, SLO…) | **drives the score** |
| **Action** | Utilities during the run - delay, custom script, container, notify | **never** affects the score |
| **Experiment** | An organized test = one or more Faults + Probes (+ Actions) against a target | produces the score |

**Resilience Score** (say this out loud): per fault, `Fault Weight × Probe Success %`; the experiment
score is the weighted average across faults. 10/10 probes pass → 100%; 5/10 → ~50%. Probes are the whole game.

## Narrative arc

> Why break things on purpose → the scientific loop (steady state → hypothesis → fault → observe → learn → expand) →
> a safe first fault → probes & Resilience Score → Prometheus steady-state → govern it safely →
> gate it in the pipeline → scale it with templates → operationalize it.

---

## Prerequisites (pre-provisioned before the session)

Participants should NOT build these; they must exist so the workshop starts fast.

- [ ] Kubernetes cluster + the sample app deployed (**frontend** + **backend** web services). Confirmed: namespace is **org-scoped** - hardcoded in the `prod`/`k8s` infra definition and shared by all projects in the org; per-participant isolation is by deployment name + `releaseName: release-<+INFRA_KEY>`. **Namespace = org name without the `-org` suffix (just the name + numbers), then `-ns`** (e.g. org `barclays517-org` → `barclays517-ns`). Participants find their org by hovering **Project** in the left nav. Deployments `frontend-<project_id>-deployment` / `backend-<project_id>-deployment`, infra `prod/k8s`.
- [ ] Harness project per participant (or shared org with per-participant projects). `⟨verify⟩`
- [ ] Discovery Agent installable / chaos infrastructure connected. `⟨verify⟩`
- [ ] **Prometheus** running and reachable from the execution plane, app metrics exposed. `⟨verify⟩`
- [x] Account-level **Experiment Templates** in the `workshop` ChaosHub (built by Sagar):
      **`service-restart-test`** (Pod Delete on the backend + `svc-health-check` HTTP probes before/during/end) and
      **`service-availability-zone-failure`** (zone-scoped Pod Network Loss + the same HTTP probes), plus the
      **`svc-health-check`** probe template (HTTP GET → `200` on the frontend URL). All parametric via inputs
      `PROJECT_ID` / `TARGET_NAMESPACE` / `TARGET_SERVICE_NAME` (+ `TARGET_ZONE` for the AZ one); no auth/JWT.
      Module 3 uses `service-restart-test`.
- [ ] An existing **CD pipeline** deploying the app with a **canary** stage (for Module 7). `⟨verify⟩`
- [ ] A **sample Locust** load test in the environment (for the demo). `⟨build⟩`
- [ ] Notification channel (Slack or MS Teams) wired to a user group (for the extension module). `⟨verify⟩`

---

# CORE PATH (~2 hours)

## Module 0 - Framing: why break things on purpose (10 min · present)

**Objective:** give the audience the "why" and the vocabulary before they touch anything.

**Talk track:**
- Definition: chaos engineering = *building confidence that the system withstands turbulent conditions*.
- The scientific loop: **steady state → hypothesis → inject fault → observe → abort if needed → learn → expand blast radius.**
- Pre-empt the two objections: *"we're afraid of prod"* (answer: blast-radius limits + automated abort make it safe; start in staging) and *"we're not Netflix"* (answer: it scales down - one hypothesis, one fault).
- Plant the [mental model](#the-one-mental-model-plant-this-in-module-0-reinforce-throughout) and the Resilience Score.
- Position Harness: this lives in the **same platform as your CD**: chaos is a step in your pipeline, not a bolt-on tool.

**Suggested opening (≈2 min, in your own words):**
> "Every system fails eventually - the only question is whether you find out in a controlled experiment, or at 3am from an angry customer. Resiliency testing is the discipline of *deliberately* injecting failure to build confidence that the system survives it. We treat it like a science experiment: define what 'healthy' looks like (the **steady state**), make a **hypothesis** ('if I kill a pod, the app stays up'), inject the **fault**, and watch whether the hypothesis holds. If it goes wrong, it stops automatically. Today you'll do exactly that on a live app, put guardrails around it, and then wire it into a deployment so a release can't ship unless it survives. And you don't need to be Netflix to start - one hypothesis and one fault is enough."

**Differentiator seeded:** single unified platform.

## Module 1 - Auto-Discovery & Application Map (15 min · hands-on)

**Objective:** populate the participant's world automatically instead of hand-defining targets.

**Flow:** Project Settings → Discovery → run the Discovery Agent → **Discover Now** → create an Application Map from the discovered services.

**Exact steps:** `TODO` (see [Appendix B](#appendix-b--previous-lab-reference-for-exact-steps) - discovery + app-map steps already exist there).

**Talk track:** the agent list/watches services and maps real traffic; the map updates after every run. Compare to competitors where you hand-enter resource IDs.

**Differentiator:** auto-discovery + application map (*table-stakes-plus* - win on auto-generating experiments from it in Module 2).

## Module 2 - Auto-Generate & Run an experiment (15 min · hands-on)

**Objective:** the platform proposes a starting battery of experiments from the map, and participants run one and read the result in Run History - a fast first win on their own app.

**Flow:** Insights → Application maps → open the map → Chaos Experiments → **Only a few** (Harness generates a Pod Delete per service, each with a built-in probe) → open the **web-backend** experiment → **Run** → **Run History** tab → open the run to see the **timeline** (fault + probe) and the **Resilience Score**.

**Exact steps:** `TODO` (auto-generate + run + Run History is new - write fresh; the "Only a few" chooser and map nav are captured in README Step 3).

**Talk track:** "the map isn't just a picture - it seeds *and scores* experiments." The built-in probe is **pod-level** (did the pod recover) - a real score, but the *app-level* HTTP probe ("is the app returning 200?") comes when they build their own in Module 4. Keep this the friendly first run; save the deep observability walkthrough for Module 3. (Confirmed: `service-restart-test` ships with **app-level HTTP** health-check probes - the pod-level → HTTP step-up.)

**Differentiator:** one click turns a discovered app into a proposed, **scored** experiment - no blank page, no hand-entered targets.

## Module 3 - Reuse a Pre-Built Experiment Template & read the run (25 min · hands-on)

**Objective:** the anchor module. Now the standardized, reusable way - import a shared template and slow down on the observability walkthrough (logs, probe pattern, score). Module 2 was the quick first run; this is the deep dive.

**Flow:**
1. New Experiment → **Create From Template** (dropdown) → pick **`service-restart-test`**.
2. Import **as copy** (so they can tinker) vs **as reference** (locked) - explain both.
3. Select **Environment / Infrastructure** (`prod` / `k8s`), then fill inputs `PROJECT_ID` / `TARGET_NAMESPACE` (= org name without the `-org` suffix, then `-ns` - e.g. `barclays517-org` → `barclays517-ns`; participants get their org by hovering **Project** in the left nav) / `TARGET_SERVICE_NAME` (= `backend-<project_id>-deployment`).
4. **Run it.**
5. **Walk the run - this is the money shot:**
   - Timeline / graph view of fault + probes.
   - Logs: *which pod was targeted, which new pod came up* - all without touching `kubectl`.
   - Probes evaluated (health check at start / during / end = steady-state pattern).
   - **Resilience Score** + report.

**Exact steps:** create-from-`service-restart-test` + fill inputs + run are captured in README Step 4; the observability walkthrough (timeline/logs/probe pattern/score) is new - write fresh.

**Talk track:** why **Pod Restart** first - the impact is legible in logs (Sagar's tip). Point out the start/during/end health-probe pattern as the *recommended* way to establish and re-verify steady state. Observability + score is what customers actually care about.

**Differentiators:** experiment templates; observability; Resilience Score.

## Module 4 - Build Your Own Experiment (25 min · hands-on)

**Objective:** "I did it myself." Recreate a similar experiment from scratch, referring to the template.

**Flow:**
1. New Experiment → **Blank Canvas** → Add Fault → **Pod Delete**.
2. Target a **discovered** workload (from Module 1) - kind/namespace/name.
3. Add probes:
   - **HTTP probe** on the app health endpoint (criteria `== 200`).
   - **Prometheus probe**: a **PromQL** steady-state check (e.g. error rate below threshold / p95 latency). This is the Prometheus story on the whiteboard.
4. Run. **While it runs, open the app endpoint** so they *see* the app stay alive / degrade in real time.
5. Compare Resilience Score to Module 3.

**Exact steps:** `TODO` - HTTP probe config + endpoint format exist in [Appendix B](#appendix-b--previous-lab-reference-for-exact-steps); **Prometheus probe (PromQL + endpoint) is new - write fresh and `⟨verify⟩` the PromQL against the lab's metrics.**

**Talk track:** the probe is the hypothesis. Tie the PromQL back to an SLO. Reinforce: probes drive the score, faults don't.

**Differentiator:** Prometheus steady-state probes; score tied to real hypotheses.

## Module 5 - Chaos Guard: govern it safely (15 min · hands-on)

**Objective:** answer "how do we allow this without causing a real outage?"

**Flow:**
1. Create a Chaos Guard rule (project admin): **deny node-level faults**.
2. Add a **time-window** condition: block a business-critical window.
3. Attempt a run that violates the rule → show it's **blocked before execution**.

**Exact steps:** `TODO` (new - `⟨verify⟩` Chaos Guard nav and who can configure it).

**Talk track:** Chaos Guard is **runtime governance on top of RBAC**: RBAC says *who can*, Chaos Guard says *what/when even they can't*. Give devs freedom to test, but ring-fence destructive faults and critical hours (cron them out-of-hours instead). Only project admins set rules.

**Differentiator (lead with this):** Chaos Guard is the strongest enterprise wedge; competitors rely on plain RBAC/IAM. Lead with this for a regulated customer like Barclays.

## Module 6 - Embed in CD: Canary + Chaos + Continuous Verification (25 min · hands-on)

**Objective:** the headline for a CD customer - resilience as an automated deployment gate.

**Flow:**
1. Open the existing CD pipeline → Deploy stage → canary phase.
2. **After** canary deploy, **before** approval, add a **Verify (Continuous Verification)** step (Canary type). `⟨verify⟩`
3. In parallel, add a **Chaos** step → select an experiment from Module 3/4 → set **Expected Resilience Score** (e.g. 50). Use runtime inputs/expressions where useful.
4. Run the pipeline; after it completes, review how the gate behaved (pass → roll forward; fail → hold/rollback).

**Exact steps:** `TODO` - Verify + Chaos step config already exist in [Appendix B](#appendix-b--previous-lab-reference-for-exact-steps); adapt to Pod Delete experiment and canary gating.

**Talk track:** "a resilience stage/step can sit anywhere between deploy and the next stage." This is chaos **inside** the pipeline with CV - not a separate tool you glue in.

**Differentiator (lead with this):** Chaos + CD + CV + canary gating in one platform is the biggest gap vs Gremlin/AWS FIS (both bolt-ons).

> End of core path (~2h with light Q&A). Modules below are optional extensions.

---

# EXTENSION MODULES (optional - add by time & interest)

## Module 7 - Templatize & Standardize (15 min · hands-on)

**Objective:** close the scalability loop Sagar stressed - one team defines, everyone reuses.

**Flow:** save the Module 4 experiment as an **Experiment Template at org level** → have another participant/project **reuse** it (import as reference vs copy) with only their own input values.

**Exact steps:** `TODO` (`⟨verify⟩` whether templates live under ChaosHub or Settings → Templates in the tenant - see [checklist](#second-pass-checklist)).

**Talk track:** same template, 20 teams, zero rework - governance + standardization is the enterprise value, not templates alone.

**Differentiator:** org-wide standardization (templates alone are table-stakes; standardization + governance is the story).

## Module 8 - Load Testing (10 min · demo)

**Objective:** introduce chaos-under-load without configuration overhead.

**Flow:** run/show a **sample Locust** load test; explain injecting faults *while* under load = authentic steady state.

**Talk track:** Locust is GA; JMeter/K6 may be coming-soon/feature-flagged - `⟨verify⟩` before quoting. Demo only.

**Differentiator:** integrated load testing (chaos + load in one tool).

## Module 9 - Notifications, Dashboards & Reporting (10 min · demo/hands-on)

**Objective:** show how this runs unattended and trends over time.

**Flow:** configure a **Slack / MS Teams** notification for an experiment; open the dashboard (top experiments, pass/fail, **Resilience-Score trend**).

**Differentiator:** operationalizing / automation.

## Module 10 - Enterprise ChaosHub Tour + Custom Faults (10 min · present)

**Objective:** convey breadth and extensibility.

**Flow:** tour the read-only **Enterprise ChaosHub**: OOTB faults across Kubernetes / AWS / Azure / GCP / VMware / Linux / Windows; show that teams can author **custom faults** reusable org-wide.

**Talk track:** quote the OOTB fault count the tenant actually shows (`⟨verify⟩` - marketing figures vary). "You already trust the open-source engine - Harness is the same **CNCF LitmusChaos** lineage, hardened and governed."

**Differentiator:** breadth + custom fault authoring + CNCF-maintainer credibility.

## (Not included) Disaster Recovery Testing

Skipped - workflow/pipeline-based and too complex for a workshop. Demo only if a customer asks.

---

## Competitive positioning cheat-sheet (for the SE, not the slides)

**Lead with these (genuinely differentiated):**
- **Chaos Guard**: runtime governance beyond RBAC.
- **Single-platform CD + CV + canary gating**: others integrate as a bolt-on.
- **Chaos-under-load**: faults while load-testing, same tool.
- **CNCF LitmusChaos maintainer**: "we build the standard."

**Concede honestly, then pivot:** AWS FIS has deep native AWS control-plane faults; Gremlin has the most polished UX. Neither has governance + pipeline-native gating + chaos-under-load in one platform.

**Don't over-claim (table-stakes):** experiment templates, auto-discovery, and resilience scoring all have competitor equivalents (Steadybit, Gremlin). Win on governance + standardization + pipeline-native.

Legend: "Best" = clear category leader, "Yes" = supported, "Partial" = limited/indirect, "No" = not available.

| Capability | Harness | Gremlin | AWS FIS | Azure Chaos | Chaos Mesh | Steadybit |
| --- | :-: | :-: | :-: | :-: | :-: | :-: |
| Runtime governance (Chaos Guard) | Best | Partial | IAM only | Partial | No | Partial |
| Native CD + CV + canary gating | Best | Bolt-on | Bolt-on | No | No | Bolt-on |
| Integrated load testing | Best | No | No | No | No | No |
| Multi-cloud + K8s + Linux/Windows | Yes | Yes | AWS only | Azure only | K8s only | Yes |
| Resilience Score | Yes | Partial | No | No | No | Yes |
| Auto-discovery / app maps | Yes | Partial | No | No | No | Yes |
| CNCF OSS lineage + enterprise support | Best | No | No | No | Community | No |

---

## Second-pass checklist

Verify these live in the workshop tenant, then fill in the `TODO` / `⟨verify⟩` steps:

- [x] **Templates location:** confirmed - account-level `workshop` ChaosHub (Account Settings → ChaosHub → `workshop`), with the experiment templates under the **Experiments** tab and `svc-health-check` under **Probes**. Affects Modules 3 & 7.
- [ ] **Load Testing frameworks:** Locust GA; confirm JMeter/K6 availability / feature flags. Module 8.
- [ ] **OOTB fault count** to quote. Module 10.
- [ ] **Prometheus probe** PromQL + endpoint against the lab's actual metrics. Module 4.
- [ ] **Chaos Guard** nav + who can configure. Module 5.
- [ ] **Canary + Verify + Chaos step** nav in the current pipeline UI. Module 6.
- [ ] Confirm env/infra names (`prod` / `k8s`) and the app endpoint format.

---

## Appendix A - Timing summary

| Path | Modules | Approx |
| --- | --- | --- |
| **Core** | 0 Framing, 1 Discovery, 2 Auto-gen + run, 3 Reuse template, 4 Build own, 5 Chaos Guard, 6 CD canary | ~2h + Q&A |
| **Extension** | 7 Templatize, 8 Load test, 9 Notifications/dashboards, 10 ChaosHub tour | +~45m |

Trim order if running long: 2 → 9 → 8 → 10. Never cut 3, 4, 5, or 6 (they carry the differentiators).

---

## Appendix B - Previous lab (reference for exact steps)

> Kept from the old Chaos Engineering lab as source material for writing the `TODO` steps above.
> Remove before shipping the final version.

### Discovery & Application Map (old)
1. Module menu → **Resilience Testing**.
2. Project Settings → **Discovery** → expand the `DA-K8s` agent → **Discover Now**.
3. After discovery, double-click the `DA-K8s` agent → **Application Maps** tab → **Create New Application Map** (Name: `workshop-am`) → select the relevant services (use search) → **Save**.

### Auto-generate experiments (old)
1. **Resilience Management** → drill into the application map → **Chaos Experiments** → **Only a few** → observe and run the `web-backend` experiment.

### Manual experiment - network corruption (old)
1. **Chaos Testing → + New Experiment**, Name `network-corruption`.
2. **Harness Infra** → Select a chaos Infrastructure → Environment `prod`, Infrastructure `k8s` → Next.
3. **Add Fault → Pod Network Corruption**. Target Application: Workload Kind `deployment`, Namespace (from dropdown), Names (backend deployment), Labels empty.
4. **Tune Fault:** Total Chaos Duration `150`, Network Packet Corruption Percentage `100` → Apply Changes → Save → Run.
5. While running, hit the app endpoint and observe network errors. Endpoint format: `http://<project_id>.cie-bootcamp.co.uk`.

### HTTP probe (old)
1. Project Settings → **Chaos Probes → + New Probe → HTTP**.
   - URL `http://<project_id>.cie-bootcamp.co.uk`, Criteria `==`, Response Code `200`.
2. Configure Properties: Timeout `20s`, Interval `2s`, Attempt `5`, Initial Delay `5s`.
3. Chaos Testing → open the experiment → hover the fault → **+ Add a parallel node → Add a probe** → Save & rerun → compare logs / Resilience Score.

### Pod memory experiment + canary via YAML (old)
1. **+ New Experiment**, Name `pod-memory`; Harness Infra → Env `prod`, Infra `k8s`.
2. **Add Fault → Pod Memory Hog**; target the backend deployment.
3. **Tune Fault:** Total Chaos Duration `600`, Memory Consumption `300`, Workers `1`, Pod affected `100` → Apply → Save.
4. Switch to YAML → edit → find `TARGET_WORKLOAD_NAMES` → append `-canary` (i.e. `backend-<project_name>-deployment-canary`) → Save.

### Embed chaos in CD pipeline (old)
1. Module menu → **Continuous Delivery & GitOps** → Pipelines → open the pipeline.
2. In **Deploy backend**, after Canary Deployment and before approval, **+ add step → Verify**:
   - Name `Verify`, CV Type `Canary`, Sensitivity `Low`, Duration `10 mins`.
3. Under Verify, **+ add a step in parallel → Chaos**:
   - Name `Chaos`, Select Chaos Experiment `pod-memory`, Expected Resilience Score `50`.
4. Apply Changes → Save → Run (Branch `main`) → review after ~10 min.

---

## Appendix C - Objection handling & Q&A bank

**Objections you'll likely hear (with crisp answers):**

- **"We can't run this in production."** Start in staging/pre-prod - production is the *goal*, not the start. Blast radius is scoped (pod %, single service), Chaos Guard blocks dangerous faults and critical windows, and probes auto-abort the run if steady state breaks. You expand only after green.
- **"We're not Netflix / too small for chaos."** It scales down. One hypothesis + one pod-delete in staging *is* chaos engineering. You don't need Chaos Monkey to get value.
- **"We don't even know our steady state."** That's the first win, not a blocker - chaos forces you to define your SLIs/SLOs. The Prometheus probe in Module 4 makes it explicit.
- **"Won't this just cause outages?"** The whole point is *controlled, observed, abortable* failure: blast-radius limits + probes + Chaos Guard. Contrast with the uncontrolled outage you're trying to prevent.
- **"How is this different from load testing?"** Load = behaviour under *traffic*; chaos = behaviour under *failure*. Best combined - inject faults *while* under load (Module 8).
- **"We already have RBAC - why Chaos Guard?"** RBAC = *who* can act. Chaos Guard = *what* and *when* they can, evaluated at runtime. Different axis; they layer.

**Product questions you should be ready for:**

- *What faults are supported?* 200+ OOTB (confirm tenant count) across Kubernetes, AWS, Azure, GCP, VMware, Linux, Windows - plus custom fault authoring.
- *Does it work outside Kubernetes?* Yes - Linux/Windows native agents and cloud faults. (Chaos Mesh, a common OSS comparison, is K8s-only.)
- *Which metric providers can probes use?* Prometheus, Datadog, Dynatrace, New Relic, Splunk, AppDynamics, GCP Cloud Monitoring - plus HTTP, command, Kubernetes, and SLO probes.
- *How is the Resilience Score computed?* Per fault: `Fault Weight × Probe Success %`; experiment score is the weighted average across faults. Actions never count.
- *Can we gate deployments?* Yes - Chaos step + Continuous Verification in the pipeline (they see it in Module 6).
- *Load frameworks?* Locust GA; JMeter/K6 confirm status in-tenant.
- *Relationship to LitmusChaos?* Harness is the primary CNCF maintainer of LitmusChaos; the product is the hardened, governed enterprise edition of the same engine.

## Appendix D - Facilitation notes & live gotchas

**Pre-flight (do this before attendees arrive):**
- Run **discovery once** so it's warm; confirm the app URL loads in a browser.
- **Pre-run the template experiment once** so logs/score render instantly during Module 3.
- Confirm the **Prometheus endpoint is reachable** from the execution plane and your **PromQL returns data**.
- Confirm the **CD pipeline runs green** and the **sample Locust** test works.
- Confirm **your user can create Chaos Guard rules** (project admin) - this is admin-gated.
- Have a **backup recording / screenshots** in case the cluster misbehaves.

**Pacing:**
- Keep Module 0 ≤ 10 min. The anchor is **Module 3**: do *not* rush the observability walkthrough; that's what customers remember.
- Natural **pause points**: after Module 3 (quick concept check - "which of these drives the score: fault, probe, or action?") and after Module 6 (recap the differentiators).
- Module 6's canary + CV run takes ~10 min - **kick off the run, then talk while it executes.**
- Trim order if running long: **2 → 9 → 8 → 10**. Protect 3, 4, 5, 6 - they carry the differentiators.

**Live gotchas:**
- **CPU/network faults look like "nothing happened"** in the logs - that's exactly why we use **Pod Delete** (Sagar's tip): the impact is legible.
- The **Prometheus probe fails silently** if the endpoint is unreachable or the query returns empty - always test it beforehand.
- **Discovery takes ~a minute**: tell people not to spam *Discover Now*.
- **Import as reference is read-only**: anyone who wants to edit must use **import as copy**.
- If **Chaos Guard** isn't in the left nav, check **Project Settings → governance**.
- If the tenant shows **Templates under Settings** rather than a ChaosHub, adjust Modules 3 & 7 wording (see checklist).

## Appendix E - Sources (from the research behind this guide)

**Chaos-engineering principles & best practice:**
- Principles of Chaos Engineering - https://principlesofchaos.org/
- PagerDuty, "10 Years of Failure Friday" - https://www.pagerduty.com/blog/insights/10-years-of-failure-friday-at-pagerduty-fostering-resilience-learning-and-reliability/
- PagerDuty, "What is Chaos Engineering?" - https://www.pagerduty.com/resources/learn/what-is-chaos-engineering/
- Gremlin, "Introduction to GameDays" - https://www.gremlin.com/community/tutorials/introduction-to-gamedays
- O'Reilly, *Chaos Engineering* - Continuous Verification - https://www.oreilly.com/library/view/chaos-engineering/9781492043850/ch16.html

**Competitive landscape:**
- Gremlin product / pricing / tool comparison - https://www.gremlin.com/product · https://www.gremlin.com/community/tutorials/chaos-engineering-tools-comparison
- AWS Fault Injection Service FAQs - https://aws.amazon.com/fis/faqs/
- Azure Chaos Studio fault library - https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-fault-library
- Chaos Mesh (CNCF) - https://chaos-mesh.org/
- LitmusChaos (CNCF) & Harness relationship - https://www.cncf.io/projects/litmus/ · https://developer.harness.io/docs/chaos-engineering/resources/hce-vs-litmus/
- Steadybit - https://steadybit.com/product/
- Gartner Peer Insights, Chaos Engineering Tools - https://www.gartner.com/reviews/market/chaos-engineering-tools

**Harness product docs:**
- Resilience Testing overview - https://developer.harness.io/docs/resilience-testing/
- Service Discovery & Application Maps - https://developer.harness.io/docs/platform/service-discovery/
- What's supported (faults) - https://developer.harness.io/docs/chaos-engineering/whats-supported/
- Probes - https://developer.harness.io/docs/resilience-testing/chaos-testing/probes/
- Resilience Score - https://developer.harness.io/docs/chaos-engineering/features/experiments/resilience-score/
- Chaos Guard / governance - https://developer.harness.io/docs/chaos-engineering/use-harness-ce/governance/governance-in-execution/
- Load Testing - https://developer.harness.io/docs/resilience-testing/load-testing/get-started/
- Continuous Verification - https://developer.harness.io/docs/continuous-delivery/verify/configure-cv/verify-deployments/
- Prometheus probe - https://developer.harness.io/docs/chaos-engineering/guides/probes/apm-probes/prometheus-probe/
