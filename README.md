# Resiliency Testing - Hands-On Lab

Welcome! In this lab you'll use **Harness Resiliency Testing** to deliberately break a running
application on Kubernetes, prove it can take the hit, and then automate that proof inside your
delivery pipeline.

## What you'll do
- Auto-discover your services and map them.
- Run a ready-made chaos experiment and read the results.
- Build your own experiment and watch your app survive (or not) in real time.
- Put guardrails on chaos so it's safe to run.
- Gate a canary deployment on a resilience score.
- *(Optional)* share experiments as templates, run a load test, set up notifications, and explore the fault library.

## What is an experiment?
An **experiment** is made of three things:
- **Faults**: the attack (e.g. *delete a pod*).
- **Probes**: the checks that decide pass/fail (e.g. *"is the app still returning HTTP 200?"*). **Probes produce your Resilience Score.**
- **Actions**: helpers during the run (delay, notify, integrate). They do **not** affect the score.

Your **Resilience Score** is simply: of all the checks you defined, how many passed. 10 of 10 → 100%. 5 of 10 → ~50%.

## Before you start
Everything below is already set up for you:
- A Kubernetes cluster with a sample app deployed.
- A Harness project with a chaos infrastructure connected.
- Prometheus collecting metrics from the app.
- A pre-built experiment template and a CD pipeline.

We'll provide you your Harness login and your **project ID**. Wherever you see `<project_id>`, use yours. 

---

# Part 1 - Core lab

## Step 0 · Deploy your application
*Prerequisite: If you haven't been through the DevOps lab in this session.*
1. Open Continuous Delivery & GitOps.

<img width="686" height="276" alt="Screenshot 2026-08-10 at 16 39 45" src="https://github.com/user-attachments/assets/c41c7616-cda0-4d18-8697-23b3af8e49d4" />

2. Run the **workshop1** pipeline we have created for you. It will deploy the application we are going to run our tests on.

<img width="1329" height="273" alt="Screenshot 2026-08-10 at 16 40 04" src="https://github.com/user-attachments/assets/50f34f9f-c2cd-4135-aeca-66220444a52c" />

3. Wait for your pipeline to run (it should take 1-2 minutes to execute).

## Step 1 · Open Resiliency Testing
1. In the Harness module menu (left), select **Resiliency Testing**.

<img width="627" height="301" alt="Screenshot 2026-08-10 at 10 36 12" src="https://github.com/user-attachments/assets/b458d55f-81d4-4b91-b19b-0f82c989ac67" />

> You should see the Resiliency Testing overview.

## Step 2 · Discover your services
*Goal: let Harness find your app's services automatically instead of typing them in.*
1. Left menu → **Project Settings** → **Discovery**.
2. Find the **DA-K8s** discovery agent, expand its side menu, and click **Discover Now**.
3. Wait for discovery to finish (about a minute - don't click it again).

<img width="1276" height="320" alt="Screenshot 2026-08-10 at 15 34 44" src="https://github.com/user-attachments/assets/5a76f68b-eb55-4897-9797-c54374aa6db7" />

Now map them:
4. Double-click the **DA-K8s** agent.
5. Open the **Application Maps** tab → **Create New Application Map**.
6. Name it `workshop-am`, select your app's services (use the search box), and click **Save**. If you have trouble finding them try searching for `web-backend-<project_id>`and `web-frontend-<project_id>`.

> You now have an application map of your services.

## Step 3 · Run an auto-generated experiment
*Goal: let the platform propose an experiment for you, run it, and read the result in Run History.*
1. Left menu → **Insights** → **Application maps** → open your `workshop-am` map.
2. Go to **Chaos Experiments** → choose **Only a few**.

<img width="1022" height="425" alt="Screenshot 2026-08-10 at 17 10 07" src="https://github.com/user-attachments/assets/0d13999c-3ab7-4d03-815d-8bac6944f17d" />

> Harness generates a starter set for you - a **Pod Delete** on your `frontend` and on your `backend`, each with a built-in health probe. No need to build anything.

<img width="1369" height="510" alt="Screenshot 2026-08-10 at 17 12 25" src="https://github.com/user-attachments/assets/b4620695-129d-476b-b2d6-1f10dc454e06" />

3. Open the **web-backend** experiment and click **Run** (top right).
4. Open the **Run History** tab. Watch the run execute; when it finishes you'll see **Completed** and your **Resilience Score**.

<!-- INSERT: Run History screenshot (Resilience Score Trends + Experiment Runs, 100/100) -->

5. Click the run to open its **timeline**: the **fault** (the exact pod that was deleted) and the **probe** (the health check that stayed green through the deletion) - all without touching `kubectl`.

<!-- INSERT: run timeline screenshot (FAULT pod-delete + PROBE default-pod-level-probe) -->

> You've run your first experiment and read its Resilience Score in Run History.

## Step 4 · Run a ready-made experiment (from a template)
*Goal: now do it the reusable way - run a standardized, ready-made experiment from a template.*
1. Go to **Chaos Experiments** → click the dropdown next to **New Experiment** → **Create from Template**.
2. Select the **Gateway Service Pod Restart** template.
3. Choose **Import as copy** (so you can tweak it) - the alternative, *Import as reference*, is locked/read-only.
4. Select your infrastructure: Environment `prod`, Infrastructure `k8s`.
5. Continue to the experiment and click **Run**.

While it runs (and after), explore these - they're the whole point:
- The **timeline / graph**: the fault and its probes.
- The **logs**: you can see *exactly which pod was deleted and which new pod replaced it*, without ever touching `kubectl`.
- The **probes**: a health check runs **before, during, and after** the fault. That's how you establish and re-confirm "steady state."
- Your **Resilience Score** and the **report**.

> Same loop as Step 3, but from a shared template - one definition many teams can reuse.

## Step 5 · Build your own experiment
*Goal: create one from scratch. Peek back at Step 4 if you get stuck.*
1. **Chaos Experiments → New Experiment → Blank Canvas.** Name it `my-pod-delete`.
2. Select Environment `prod`, Infrastructure `k8s`.
3. **Add Fault → Pod Delete.**
4. **Target Application:**
   - Workload Kind: `deployment`
   - Namespace: pick yours from the dropdown
   - Name: pick your **backend** deployment
5. Add a probe to check the app stays healthy:
   - **+ Add probe → HTTP probe**
   - URL: `http://<project_id>.cie-bootcamp.co.uk`
   - Criteria: `==`  Response Code: `200`
   - Properties: Timeout `20s`, Interval `2s`, Attempts `5`, Initial Delay `5s`
6. Add a second probe to check a metric stays within bounds:
   - **+ Add probe → Prometheus probe**
   - Endpoint: your Prometheus URL *(your facilitator will confirm this)*
   - PromQL query + criteria: *(your facilitator will give you the exact query - e.g. "error rate stays below 5%")*
7. **Save**, then **Run**.
8. While it runs, refresh your app tab (`http://<project_id>.cie-bootcamp.co.uk`) - watch it stay available as pods are killed and recreated.

> Compare your Resilience Score to Step 4's.

## Step 6 · Make chaos safe with Chaos Guard
*Goal: guardrails so experiments can't cause a real outage.*
1. Left menu → **Chaos Guard**.
2. Create a new rule:
   - **Deny** node-level faults (so no one can drain a node).
   - Add a **time window** that blocks runs during business-critical hours.
3. Save, then try to run something the rule forbids.

> The run is blocked **before it starts**. Chaos Guard sits on top of your normal access controls: access decides *who can act*; Chaos Guard decides *what and when - even for people who otherwise could*.

## Step 7 · Gate a canary deployment on resilience
*Goal: make resilience an automatic quality gate in your pipeline.*
1. Module menu → **Continuous Delivery & GitOps** → **Pipelines** → open the existing pipeline.
2. In the **Deploy** stage's **canary** phase (after the canary deploy, before the approval), click **+** to add a step → **Verify**:
   - Name `Verify`, Type `Canary`, Sensitivity `Low`, Duration `10 mins`
3. Under the Verify step, click **+** to add a step **in parallel** → **Chaos**:
   - Name `Chaos`, Experiment: `my-pod-delete` (or the template experiment), Expected Resilience Score `50`
4. **Apply Changes → Save → Run** (Branch `main`).
5. After it runs, review the result: if the resilience score is ≥ 50 the canary proceeds; if not, it holds / rolls back.

> You've turned resilience into a deployment gate - chaos running **inside** your pipeline, not as a separate tool.

**That's the core lab.** The optional steps below go deeper.

---

# Part 2 - Optional

## Step 8 · Share your experiment as a template
1. Open your `my-pod-delete` experiment → save it **as a template** at **org** level.
2. Have a teammate (or switch to another project) → **Create from Template** → run it with their own values.

> One definition, reused by many teams - only the inputs change.

## Step 9 · Run a load test (Locust)
1. Left menu → **Load Testing**.
2. Open the sample **Locust** test and run it.

> You've generated load. Real-world tip: run chaos *while* under load for a realistic test.

## Step 10 · Get notified & see trends
1. Configure a **Slack** or **Microsoft Teams** notification for an experiment.
2. Open the **Dashboard** to see pass/fail and your Resilience Score trend over time.

> Now experiments can run unattended and report back to your team.

## Step 11 · Explore the fault library (Enterprise ChaosHub)
1. Browse the **Enterprise ChaosHub**.
2. See the built-in faults across Kubernetes, AWS, Azure, GCP, VMware, Linux, and Windows - plus the option to author your own custom faults.

> You've seen the breadth of what you can test.

---

## Values your facilitator will confirm
- `<project_id>` - your Harness project id (used in the app URL).
- Your app's namespace and backend deployment name.
- The Prometheus endpoint and the exact PromQL query for Step 5.
- Environment / infrastructure names, if different from `prod` / `k8s`.

---

