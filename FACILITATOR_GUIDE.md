# Lab  - Chaos Engineering


### Summary: Fully integrated chaos experiments with the delivery process

**Learning Objective(s):**

- Auto generate chaos experiments on deployed services
- Build a chaos experiments using a base fault (out of 200 OOTB faults)
- Embed chaos engineering experiments into the deployment process
- Add continuous verification to the deployed service
- Automate release validation

**Steps**

1. From the module selection menu select **resilience testing**

<img width="733" height="402" alt="Screenshot 2026-03-06 at 08 14 35" src="https://github.com/user-attachments/assets/3d6bba05-4189-4c14-893b-cb75e270029d" />


4. From the left hand menu, go to **Project Settings**
5. From the available tiles select “Discovery”
6. After expanding the side menu of the “DA-K8s” agent click on “Discover Now”

  ![image](https://github.com/user-attachments/assets/be1a029d-9b4e-4971-9519-f284ba75c815)



**Create Application Map**

1. After discovery is complete double click on the agent “DA-K8s”

   ![image](https://github.com/user-attachments/assets/3094b76d-1d27-429d-ab67-0fdb8e978894)


1. Select the "Application Maps" tab
2. Click on **Create New Application Map** and enter the following values

| Input                        | Value|  
| ---------------------------- | ------ |
| Name                         |workshop-am|

4. Select the relevant services for your project name "use the search function to find the services"
5. Click Save


**Auto Generate Chaos Experiments**
1. From left handside menu select **Resilience Management**
2. Drill down to the previously created application map
3. Navigate to **Chaos Experiments**
4. Select **Only a few**

Observe the auto generated experiments and run the **web-backend experiment**




---------------

**Create Experiments manually**

This time we will create a network corruption experiment

1. From the left hand menu, go to **Chaos Testing**
2. Select **+New Experiment**

| Input                        | Value|  
| ---------------------------- | ------ |
| Name                         |network-corruption|

3. Select **Harness Infra**

  ![Screenshot 2024-11-28 at 14 24 21](https://github.com/user-attachments/assets/c47834a3-fe88-44ed-be7e-7cee97bcb303)

  - Click on **"Select a chaos Infrastructure"**

 
4. On the popup window select the available options

| Input                        | Value|  
| ---------------------------- | ------ |
| Select Environment|prod|
| Select Infrastructure|k8s|

5. Click on next to navigate to the experiment builder
6. Click on **Add Fault**
7. From the list of available faults select **Pod Network Corruption**
8. From the navigation bar select **Target Application**

| Input                        | Value | Notes |
| ---------------------------- | ------ | -------|
| Target Workload Kind|deployment||
| Target Workload Namespace ||**Select the namespace available from the dropdown**|
| Target Workload Names | Pick the backend deployment name|We will change that later |
|Target Workload Labels | leave empty||


9. From the navigation bar select **Tune Fault**

| Input       | Value |
| ----------- | ----- |
| Total Chaos Duration |150|
| Network Packet Corruption Percentage |100|
10. Click on **Apply Changes** and then **Save**
11. Run the experiment and observe the logs. 
12. While the experiment is running navigate to the application's endpoint (see image below)
13. Observe the network errors

| project                | domain        | suffix |
| ---------------------- | ------------- | ------ |
| http\://\<project\_id>|.cie-bootcamp|.co.uk|

<img width="1415" height="205" alt="image" src="https://github.com/user-attachments/assets/cb85c608-bfbb-4ec5-b179-c6ebd6ff394c" />


**Validate the health automatically**

1. From the left hand menu, go to **Project Settings**
2. Select **Chaos Probles**
3. Create a new probe by clicking **+ New Probe**
4. Select the HTTP probe
   
| Field                | Value        | Notes |
| ---------------------- | ------------- | ------ |
| http\://\<project\_id>.cie-bootcamp.co.uk |||
| Criteria | == | | 
| Response Code | 200 | | 

5. Move to the next tab **Configure Properties**


| Field                | Value        | Notes |
| ---------------------- | ------------- | ------ |
| Timeout | 20s ||
| Interval | 2s | | 
| Attempt | 5 | | 
| Initial Delay | 5s ||

Now that we have our probe navigate to **Chaos Testing**
Drill down to the **network-latency** experiment

Hovering over the network fault we can add our newly created probe 
Select **+ Add a parallel node -> Add a probe**

<img width="519" height="354" alt="image" src="https://github.com/user-attachments/assets/519fc841-bbf1-4480-a01c-4583eab0b208" />

Save and rerun the experiment 
Observe the **logs** of the probe **Resilience Score** generated in comparisson to the previous execution



---------------

**Create Experiments manually**

1. From the left hand menu, go to **Chaos Testing**
2. Select **+New Experiment**

| Input                        | Value|  
| ---------------------------- | ------ |
| Name                         |pod-memory|

3. Select **Harness Infra**

  ![Screenshot 2024-11-28 at 14 24 21](https://github.com/user-attachments/assets/c47834a3-fe88-44ed-be7e-7cee97bcb303)

  - Click on **"Select a chaos Infrastructure"**

 
4. On the popup window select the available options

| Input                        | Value|  
| ---------------------------- | ------ |
| Select Environment|prod|
| Select Infrastructure|k8s|

5. Click on next to navigate to the experiment builder
6. Click on **Add Fault**
7. From the list of available faults select **Pod Memory Hog**
8. From the navigation bar select **Target Application**

| Input                        | Value | Notes |
| ---------------------------- | ------ | -------|
| Target Workload Kind|deployment||
| Target Workload Namespace ||**Select the namespace available from the dropdown**|
| Target Workload Names | Pick the backend deployment name|We will change that later |
|Target Workload Labels | leave empty||


9. From the navigation bar select **Tune Fault**

| Input       | Value |
| ----------- | ----- |
| Total Chaos Duration |600|
| Memory Consumption |300|
| Number of workers |1|
| Pod affected percentage|100|

10. Click on **Apply Changes** and then **Save**


**Change target service to canary using YAML**

1. From the pipeline visual editor switch to yaml
2. Click the edit button to go into edit mode
3. Locate the service name (set on previous state) **TARGET_WORKLOAD_NAMES**
4. Replace it with **backend-<project_name>-deployment-canary** where project_name is the harness project. Summary: add the suffic **-canary** to the target workload
5. Save the experiment


**Embed chaos experiments into CD pipelines**

1. From the module selection menu select Continuous Delivery & GitOps


   ![Screenshot 2024-11-28 at 14 07 22](https://github.com/user-attachments/assets/898ee27b-7369-47c6-a145-e74b49bb4bed)

   
2. From the left hand side menu select pipelines and drill down to the existing pipeline

3. In the existing pipeline, within the Deploy backend stage **after** Canary Deployment and **before** the approval step click on the plus icon to add a new step

4. Add a **Verify** step with the following configuration

| Input                        | Value  | Notes                                                                                            |
| ---------------------------- | ------ | ------------------------------------------------------------------------------------------------ |
| Name                         |Verify|                                                                                                  |
| Continuous Verification Type |Canary|                                                                                                  |
| Sensitivity                  |Low| This is to define how sensitive the ML algorithms are going to be on deviation from the baseline |
| Duration                     |10mins|                                                                                                  |

5. Under the verify step click on the plus icon to add a new step in parallel


   ![Screenshot 2024-11-28 at 14 28 38](https://github.com/user-attachments/assets/368ba808-d303-43f8-8824-5d2e09367b01)

   
6. Add a **chaos** step with the following configuration

| Input                        | Value  |
| ---------------------------- | ------ |
| Name                         |Chaos|
| Select Chaos Experiment |pod-memory|
|  Expected Resilience Score|50| 

7. Click on Apply Changes

8. Click **Save** and then click **Run** to execute the pipeline with the following inputs.

| Input       | Value | Notes       |
| ----------- | ----- | ----------- |
| Branch Name |main| Leave as is |

9. After 10 minutes review what happened with the execution 
