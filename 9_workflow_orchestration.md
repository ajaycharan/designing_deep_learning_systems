# Workflow Orchestration
## prologue
A service to manage, execute and monitor workflow automation. 
### workflow
sequence of operations that are a part of a larger task. It is a plan consisting of a set of tasks to complete a job.
An example plan can be: fetch raw data, rebuild training dataset, train model, eval the model, and deploy model.

Since workflow is an exec plan, it can be done manually but not ideal. 

We will have numerous workflows, hence need to have a system to handle the complexity of workflow execution: `workflow orchestration`

### workflow orchestration
System built to manage workflow lifecycles: creation, execution, troubleshooting. To manage the automation in DL and schedule code running.

## 9.1.1 workflow
Workflow can be seen as a DAG of steps.

### step:
smallest resumable unit of computation that describes an action. Eg. fetching data, trigger a service, etc. It either succeeds or fails as a whole.

DAG specifies the dependencies among steps and the order to execute them. They do not have a cycle. Starting at the DAG and computing the steps, following the arrows we end up with a trained and released production level model. Eg:
```mermaid
graph LR
    START:::hidden --> |model training request| 1[prod initiation]
    1 --> 2[fetch embedding]
    1 --> 3[build dataset]
    3 -.-> |call dataset mgmt service| 3.1:::hidden
    3 --> 4[check data quality]
    2 --> 5[model traning]
    4 --> |good quality| 5
    5 -.-> |call model train service| 5.1:::hidden
    4 --> |bad quality| 6[notify data team]
    5 --> 7[check model quality]
    7 --> |bad model| 6
    7 --> |qualified model| 8[release model]
    8 -.-> |call metadata service| 8.1:::hidden
    classDef hidden display: none;
```
Workflow should not have a loop to guarantee its execution won't fall into a dead loop

## 9.1.2 workflow orchestration
After defining a workflow, next step is to run it, this execution and monitoring is called orchestration
### goal
Automate the execution of tasks defined. *practically* it extens to mean the whole shebang of things: creating, scheduling, executing and monitoring multiple workflows simultaneously in an automated way.
### motivation
My DL work was all in one notebook, why do I need to do this complex stuff?
1. automation
2. work sharing  
Eg:
```mermaid
graph TD
    1[auth]
    2[data pull]
    3[embedding pull]
    4[training]
    6[model release]
    1 --> 2
    1 --> 3
    2 --> 4
    3 --> 4
    4 --> 6
```
WF A
```mermaid
graph TD
    1[auth]
    2[data pull]
    4[training]
    6[model release]
    1 --> 2
    2 --> 4
    4 --> 6
```
WF B
```mermaid
graph TD
    1[auth]
    2[data pull]
    3[embedding pull]
    4[training]
    5[HPO]
    6[model release]
    1 --> 2
    1 --> 3
    2 --> 5
    3 --> 5
    5 --> 4
    4 --> 6
```
WF C

Since the WFs above are different  but they are split into shareable elemtns, we can be more productive and reduce duplication.

## 9.1.3 Challenges of WFO
It is cumbserome to prototype ideas DL ideas using WFs. Eg:
```mermaid
graph LR
    1[data exploration: analyse data] --> 2[prototyping: train model]
    2 --> 3[Workflow: convert proto code to WF]
    subgraph 4[Test env]
        4a[data mgr]
        4b[training]
        4c[metadata]
    end
    3 --> |submit WF to orch sys| 4
    4 -.-> 2
    4 -.-> 1
    2 -.-> |fast iteration| 1
```
The first 2 steps are local incubation phase and the rest are production phase.
- local incubation phase: local/dev env and use it for data exploring and prototypin. 
- production phase: convert proto code to WF by breaking into steps and define the DAG, then submit the WF to WFO sys. Then Orch sys takes over and runs the WF on the schedule it has

### gaps betn proto and prod phases
proto and prod pahse aren't directly connected! shipping incubation code to prod isn't straightfwd and this hampers development velocity:
1. WF building and debug aren't easy:
2. WF construction happens not once but frequently:
