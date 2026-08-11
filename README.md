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
6. Name it <pre>`workshop-am`</pre>
select your app's services (use the search box), and click **Save**. If you have trouble finding them try searching for `web-backend-<project_id>` and `web-frontend-<project_id>`.

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

<img width="1298" height="94" alt="Screenshot 2026-08-10 at 17 24 49" src="https://github.com/user-attachments/assets/411e0539-f93e-4660-92bf-592c29607619" />

5. Click the run to open its **timeline**: the **fault** (the exact pod that was deleted) and the **probe** (the health check that stayed green through the deletion) - all without touching `kubectl`.

<img width="1339" height="351" alt="Screenshot 2026-08-10 at 17 23 20" src="https://github.com/user-attachments/assets/76e8673c-5eb6-44eb-97b0-fac08f41e18b" />

> You've run your first experiment and read its Resilience Score in Run History!.

## Step 4 · Run a ready-made experiment (from a template)
*Goal: now do it the reusable way - run a standardized, ready-made experiment from a template.*

> **Hypothesis:** deleting a backend pod should **not** cause 5xx errors on the frontend - the deployment keeps a healthy replica and recovers within seconds.

1. Go to **Chaos Experiments** → click the dropdown next to **New Experiment** → **Create from Template**.

<img width="246" height="224" alt="Screenshot 2026-08-10 at 17 37 57" src="https://github.com/user-attachments/assets/ad6b7a9b-d946-434e-a799-1991bd7443eb" />

2. Select the **service-restart** template.
3. Choose **Import as copy** (so you can tweak it) - the alternative, *Import as reference*, is locked/read-only.
4. Select your infrastructure: Environment `prod`, Infrastructure `k8s`.

<img width="498" height="578" alt="Screenshot 2026-08-11 at 13 45 31" src="https://github.com/user-attachments/assets/d489edce-9d96-4b3d-8b43-aec0c5bbc4e8" />

5. The template will take your app variables automatically, you only have to fill the name.
6. Continue to the experiment and click **Run**.

<img width="1361" height="225" alt="Screenshot 2026-08-11 at 07 30 03" src="https://github.com/user-attachments/assets/b99d486d-35b6-49fd-a0e4-e21f366393c2" />

While it runs (and after), explore these - they're the whole point:
- The **timeline / graph**: the fault and its probes.

<img width="1433" height="342" alt="Screenshot 2026-08-11 at 07 48 23" src="https://github.com/user-attachments/assets/aab7bcd8-e02e-406b-9aa5-39f350770fee" />

- The **logs**: you can see *exactly which pod was deleted and which new pod replaced it*, without ever touching `kubectl`.

<img width="1494" height="220" alt="Screenshot 2026-08-11 at 07 50 48" src="https://github.com/user-attachments/assets/aa29115c-74df-4bbd-ad14-95e4a6bea300" />

- The **probes**: an **app-level HTTP check** (is your page returning `200`?) runs **before, during, and after** the fault - a step up from Step 3's pod-level probe. That's how you establish and re-confirm "steady state."
- Your **Resilience Score** and the **report**.

<img width="1485" height="388" alt="Screenshot 2026-08-11 at 07 52 01" src="https://github.com/user-attachments/assets/9afdb62a-d831-47f5-b236-43da39e6c5a8" />

> Same loop as Step 3, but from a shared template - one definition many teams can reuse.

> *Going further (optional):* there's a second ready-made template, **service-availability-zone-failure**, that simulates a whole availability-zone outage (network loss across a zone) and checks the same frontend health. Import and run it the same way if you finish early.

## Step 5 · Build your own experiment
*Goal: create one from scratch. Peek back at Step 4 if you get stuck.*
1. **Chaos Experiments → New Experiment → Create from scratch** Name it <pre>`my-pod-delete`</pre>
2. Select Environment `prod`, Infrastructure `k8s`.

<img width="630" height="733" alt="Screenshot 2026-08-11 at 07 55 27" src="https://github.com/user-attachments/assets/ba23e13c-b87d-418f-a5f8-f012281dfd97" />

3. **Add Fault → Pod Delete.**

<img width="456" height="257" alt="Screenshot 2026-08-11 at 07 57 17" src="https://github.com/user-attachments/assets/8257f097-e43a-4557-8807-12396d43a26b" />

4. **Target Application:**
   - Workload Kind: `deployment`
   - Namespace: pick yours from the dropdown
   - Target Workload names: pick your **backend** deployment
   - Target Workload labels: pick the one for your backend from the dropdown

   <img width="594" height="453" alt="Screenshot 2026-08-11 at 07 59 24" src="https://github.com/user-attachments/assets/819502a4-85ac-4595-a0dc-2b6703835435" />

5. Attach a probe so your score reflects whether the **app** stayed healthy:
   - On the canvas, click the **+** after your fault → **Add a probe**.
   - In **Select a Probe**, pick a ready-made **`svc-health-check`** HTTP probe → **Add to Experiment**.

   <img width="657" height="861" alt="Screenshot 2026-08-11 at 08 43 59" src="https://github.com/user-attachments/assets/1219d361-342f-474f-8802-bdcb3379d360" />
   
   - It attaches with a **blank URL** - open its **Probe Properties**, set **URL** to <pre>`<+variable.chaos_endpoint_url>`</pre> (Method `GET`, Criteria `==`, Response Code `200` are already set), then **Apply Changes**. Now the run fails if your frontend stops returning `200`.
   > *Probes are reusable **resources** here - you select an existing one in the experiment. To author your own, use **Project Settings → Chaos Probes → + New Probe**.*

   > **Best practice:** check steady state **before, during, and after** the fault - one probe before, one running in parallel with the fault, one after. That's exactly the pattern the Step 4 template ships; add more `svc-health-check` probes here if you want the same coverage.
   

## 🔄 6. Resiliency Probes

### Overview
Create resiliency probes laveraging APM tooling

### Value
- **Validate resilience with real telemetry** See how latency, errors, CPU and other metrics behave during failure.
- **Automate pass/fail decisions** Turn observability data into objective experiment success criteria.
- **Enforce SLOs during chaos** Validate thresholds such as p99 latency or error-rate limits automatically.
- **Use existing monitoring tools**  Integrate with Prometheus, Dynatrace, Datadog, New Relic, Splunk and more.
- **Keep credentials secure** Reuse Harness connectors and centrally managed secrets.
- **Combine multiple validation signals** Correlate APM, HTTP and command probes for stronger resilience checks.

### Step 1: Create a Probe
1. From the **left-hand side menu**, select **Project Settings**
2. Then select **Chaos Probes**
3. Create a new probe by clicking on **+ New Probe**
4. From the list of available probes select the **APM Probe**

> [!NOTE]
> APM Probe allows you to connect to your application monitoring tools such as DataDog, AppDynamics and others

5. Fill in the details in the Overview Tab

| Field | Value |
|--------|--------|
| **Name** | <pre>`prometheus-probe`</pre> |
| **APM Type** | Prometheus |
| **Connector** | Prometheus |

6. Click on **Next** to setup **variables**

## Variable 1: namespace
| Field | Value | Notes |
|--------|--------|-------|
| **Type** | String ||
| **Name** | <pre>`namespace`</pre> ||
| **Value** | Runtime Input |**Click the Sigma icon next to the input box to set the variable as runtime input**|

## Variable 2: container
| Field | Value | Notes |
|--------|--------|-------|
| **Type** | String ||
| **Name** | <pre>`container`</pre> ||
| **Value** | Runtime Input |**Click the Sigma icon next to the input box to set the variable as runtime input**|

7. Click next to setup the **Query**
8. Set the query as per the box below

```yaml
avg(container_memory_rss{namespace=\"<+probe.variables.namespace>\",container=\"<+probe.variables.container>\"})
```
9. Scroll down to setup the condition

| Field | Value | Notes |
|--------|--------|-------|
| **Type** | Float ||
| **Comparison Criteria** | <pre>`<`</pre> |Less than|
| **Value** | <pre>`700000000`</pre> ||

10. Click next to setup **Properties**
11. Modify only the number of attempts

| Field | Value | Notes |
|--------|--------|-------|
| **Attempts** | <pre>`10`</pre>  ||

### Step 2: Attach a probe

During the previous step we created a reusable probe that can be attached to any of our experiments. Now it is time to add it into the experiment we created previously. 

1. From the **left-hand side menu**, select **Chaos Experiments**
2. Drill down to the experiment we created earlier **my-pod-delete**
3. Enable the visual studio popup editor, by hovering over the pod-delete fault
4. Select **+Add a parallel node** and then **Add a probe**

<img width="542" height="261" alt="image" src="https://github.com/user-attachments/assets/426eff18-4ac9-4aa1-b27b-f65031723dac" />

5. From the list of available probes select the **prometheus-probe**
6. Add to the experiment
7. While the popup window is open navigate to the **Variables tab**

> [!NOTE]
> If you closed the popup window to open it again click on the node for the prometheus probe in the visual studio editor of the pipeline

8. For the variables copy the values below
 
| Variable | Value | Notes |
|--------|--------|-------|
| **namespace** | <pre>`<+variable.chaos_namespace>`</pre>  ||
| **container** | <pre>`<+variable.chaos_backend_container>`</pre>  ||

9. Click on Apply Changes
10. And then Save




### Step 3: Validate the probe and explain results
1. Run the experiment
2. While the pod delete fault is running click on the prometheus node
3. Review the logs of the prometheus validation

   
<img width="2599" height="201" alt="image" src="https://github.com/user-attachments/assets/03ddda57-dfb6-4f35-95ef-a2b52de3622a" />

### What was achieved
- **Reusable Prometheus APM probe in Harness Chaos Engineering**
- **Parameterise the probe so it can be reused across different namespaces and workloads**
- **Query live application/container telemetry from Prometheus during a chaos experiment**
- **Define an objective resilience threshold — in this case, ensuring container memory remains below 700 MB**
- **Automatically evaluate the application's behaviour while the failure is occurring**





## 🛡️ 7. Chaos Guard Controls**

### Overview
Establish guardrails to avoid unwanted experiments running based on environment, user, criticality and time

### Value
- **Limit blast radius: Restrict faults to approved clusters, namespaces, workloads, and service accounts**
- **Control who, what, where and when**
- **Enforce approved execution windows: Prevent chaos experiments from running outside authorized periods**

### Step 1: Create a Condition
> [!NOTE]
> Condition define the criteria of which faults are blocked, on which infrastructure

1. From the **left-hand side menu**, select **Project Settings**
2. Then select **Chaos Guard**
3. At the top navigation there are **Rules** and **Conditions**
4. Navigate to the conditions
5. Create a new condition by clicking **+ New Condition**


> [!WARNING]
> Change the infrastructure to **Kubernetes Harness Infrastructure**

6. Then Setup as follows

## What
| Field | Value | Notes |
|--------|--------|-------|
| **name** | <pre>`amcondition`</pre>  | |
| **FAULT** | <pre>`pod-delete`</pre>  ||


## Where

| Field | Value | Notes |
|--------|--------|-------|
| **Infrastructure** | `k8s` | **dropdown**|

## Which

| Field | Value | Notes |
|--------|--------|-------|
| **Application Map** | `workshopam` | **dropdown**|
| **Namespace** | `dropdown` | **dropdown**|
| **Services** | `backend....` | **dropdown**|


## Using

This condition BLOCKS any service account `NOT EQUAL TO` the following

| Field |
|--------|
|`sa` |

This condition `ALLOWS` the usage of unverified probes









Chaos Guard has two parts: a **Condition** (the *what/where* - which faults are blocked, on which infrastructure) and a **Rule** (the *who/when* - which user groups, during which time windows). A rule enforces one or more conditions, so you build the condition first, then wrap a rule around it.

1. **Project Settings → Chaos Guard → Conditions → + New Condition.**
   - Name it (e.g. `block-pod-delete`), Infrastructure type **Kubernetes**.
   - **WHAT:** leave it on **BLOCKS**, `FAULT` **EQUAL TO** `pod-delete`.
   - **WHERE:** the infra list is empty, so tick **All Infrastructures** (it fills in as `*`).
   - **WHICH:** the builder needs a scope - enter `*` in both **NAMESPACE** and **APP LABEL** (= any namespace, any app).
   - **USING:** enter `*` for the **service account**.
   - **Save.** *(All four are required before Save enables; `*` just means "match everything.")*

   <img width="1566" height="129" alt="Screenshot 2026-08-11 at 10 20 23" src="https://github.com/user-attachments/assets/29d9e6ed-e4ac-47b5-99d1-f5789f6f1ab6" />
   
2. **Rules → + New Rule.**
   - Name it; under **User group(s)** pick **All Project Users** - this keeps the rule inside *your* project, so you can't affect anyone else's.
   - *(Optional)* add a **time window** to apply the rule during set hours.

   <img width="965" height="640" alt="Screenshot 2026-08-11 at 10 23 24" src="https://github.com/user-attachments/assets/3613431d-2469-4e62-a728-187c89c4b224" />
   
   - **Add Conditions → select `block-pod-delete` → Done → Save.**
  
   <img width="1094" height="616" alt="Screenshot 2026-08-11 at 10 23 34" src="https://github.com/user-attachments/assets/7feb1cb4-acdc-40a7-9ca5-35756ec6e1f9" />

   - Click **Enable** to enable your Rule.
  
   <img width="1581" height="99" alt="Screenshot 2026-08-11 at 10 25 56" src="https://github.com/user-attachments/assets/286fb156-d77b-4b69-af99-56ae44018dc6" /> 
   
3. Go back to your **`my-pod-delete`** experiment from Step 5 and click **Run**.

> The run is blocked **before it starts**. Chaos Guard sits on top of your normal access controls: access decides *who can act*; Chaos Guard decides *what and when - even for people who otherwise could*.

> *Why block `pod-delete` here?* Only so you can watch the guardrail fire against the experiment you already built. In production you'd typically block **destructive node-level faults** (drain a node, node restart) so no one can take out a whole node - same mechanism, just a different fault in the Condition.

## Step 7 · Gate a canary deployment on resilience
*Goal: make resilience an automatic quality gate in your pipeline.*
1. Module menu → **Continuous Delivery & GitOps** → **Pipelines** → open the existing pipeline.

   <img width="785" height="87" alt="Screenshot 2026-08-11 at 14 01 45" src="https://github.com/user-attachments/assets/cf67f888-1f06-4504-b939-9565ac1d4fef" />
   
3. In the **backend** stage's **Canary Deployment** phase (after the Canary Deployment, before the Canary Delete), click **+** to add a step → **AI Verify**:

   <img width="331" height="232" alt="Screenshot 2026-08-11 at 14 04 08" src="https://github.com/user-attachments/assets/688e9d99-4f85-4a06-8b51-c5317389277c" />
   
   - Name `Verify`, Type `Canary`, Sensitivity `Low`, Duration `10 mins`

   <img width="600" height="642" alt="Screenshot 2026-08-11 at 14 06 42" src="https://github.com/user-attachments/assets/36318017-78ea-4104-8388-2244f4bd7473" />
   
4. Under the Verify step, click **+** to add a step **in parallel** → **Chaos Experiment**:
   - Name `Chaos`, Experiment: `my-pod-delete` (or the template experiment), Expected Resilience Score `50`
   
   <img width="600" height="767" alt="Screenshot 2026-08-11 at 14 09 05" src="https://github.com/user-attachments/assets/3ac08e64-863a-4aa3-845b-7aedb36ac030" />

   <img width="1143" height="749" alt="Screenshot 2026-08-11 at 14 10 04" src="https://github.com/user-attachments/assets/b7b6d6ef-bc05-49d7-8d15-fa2b2f51dc79" />
   
6. **Apply Changes → Save → Run** (Branch `main`).
7. After it runs, review the result: if the resilience score is ≥ 50 the canary proceeds; if not, it holds / rolls back.

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
- Your **backend** deployment name (`backend-<project_id>-deployment`). *(Your namespace you can find yourself - see Step 4: it's your org name without the `-org` suffix, plus `-ns`.)*
- The Prometheus endpoint and the exact PromQL query for Step 5.
- Environment / infrastructure names, if different from `prod` / `k8s`.

---

