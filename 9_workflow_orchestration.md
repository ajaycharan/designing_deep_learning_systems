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

So the tool for WFO should make the transition from proto code to prod WF seamless with min effort. Eg:
* Metaflow: Can create WF by writing python code with python annotaitons, get workflow from proto code directly w/o any changes, provides unified user experience on model exec betn local and cloud prod envs i.e. friction in WF testing is reduced as MetaFlow operates WFs in the same way for local and prod env.

# 9.2 design a WFO sys
## 9.2.1 User scenarios
Most WF usage can be split in 2 phases:
1. dev
2. exec
### dev phase
Convert train code into a WF:
1. Portotype model training notebook/local env. After local testing and eval, it seems this code can be deployed
2. Use syntax provided by orch sys to rebuild the code into a DAG of tasks in a YAML file. Eg: data parsing -> data augmentation -> dataset building -> training -> [online eval, offline eval] -> model release
3. Set the input/output params and actions for each step in DAG. Eg: for train step, set the step action as a RESTful HTTP request (sends a RESTful request to model train service to start a train job). The payload and params for this RESTful request come from the step input params
4. Once WF is defined, set the WF exec schedule in DAG YAML file. Eg: schedule WF to run on every MOnday and also set it to be triggered by an external event.
5. Run the WF localc validation and submit the WF to Orch service
Pseudo owrkflow
```
# define the WF DAG: body of WF, steps, dependencies
with DAG(
    description = `my sample train WF`,
    schedule_interval = timedelta(months=1),
    start_date = datetime(2025, 1, 1),
) as dag:

# define exec logic for each step
data_parse_step = BashOperator(.. .. ..)
data_aug_step  = BashOperator(.. .. ..) # execs a bash command for data augmentation
dataset_build_step  = BashOperator(.. .. ..)
train_step  = BashOperator(.. .. ..)

# declare steps' dependencies
data_parse_step >> data_aug_step >> dataset_build_step
>> train_step # sequential exec flow
```
### exec phase
orch service execs model training WF:
1. Once WF is submitted, the orch service saves the WF DAG into a database
2. Orch service's scheduler detects the WF and dispatches the tasks fo the WF to backend workers
    - The scheduler component ensures tasks are exec in a sequential manner as def in WF DAG
3. Can use the orch service's UI to check WF's exec progress and results in realtime
4. If WF produces a good model, it can be promoted by engr to the staging and production envs. Else, engr can start another protoype dev.
Critical: How easy it is to convert the proto code to WF, if we don't have to tfm the code to WF each time or have other friction, a lot of time would be saved

WF sgould be computationally light! not do heavy computations, the real work should be done by tasks

## 9.2.3 Orch sys: Genral design
### 5 components
1. web servers: UIs and APIs to create, inspect, trigger and debug behvior of WF
2. scheduler & controller: do 2 things
    - Scheduler watches every active WF in system and schedules WF to run at right time
    - Controller dispatches WF tasks to workers
3. metadata DB: stores WF's config, DAG, editing and exec history and the WF tasks' exec state
4. worker group: provide compute resources to run WF tasks. Abstract away the infra and is agnostic to the running task. Eg:
    - different workers like kubernetes worker, amazon elastic compute cloud (EC2) worker but they can all exec the same task but on different infra
5. object store: shared file storage for all other componenets; usually built on top of cloud object storage like amazon simple storag service (S3). use:
    - task output sharing: worker1 runs task1 and reads output val of the previous task i.e. task0 from the obj store as task1's input, then the worker1 also saves the task1 output to obj store for successor task task2

Obj store and metadata DB are accessible to all compoenents of orch sys. 
*benefit of centralized store* it de-couples core components so the web server, scheduler, and workes can work independently.

### How is WF exec?
1. define DAG for WF:
    - set of tasks
    - control flow of task exec sequence
    - for each task
        - use either systems default operator like shell or python
        - or build own operator to exec the task
2. submit the WF: DAG + dependent code to `web server` via UI or CLI.
    - WF saved in `metadata DB`
3. `scheduler` periodically scans the metadata DB & detects new WF
    - kicks off the WF at the scheduled time
    - by callin `controller` to
        - dispatch WF's tasks to worker queue (based on task seq in DAG)
4. a `worker` picks up a task from the sahred job queue
    - reads task definition from `metadata DB`
    - execs task by running the task operator
    - saves task output value to `obj store`
    - reports task exec status back to `metadata DB`
Meanwhile the engr can use the UI hosted on `web server` to monitor WF exec. Since both, the scheduler/controller and the workers report the status to `metadata DB` in realtime.

## 9.2.3 WFO design princiles
