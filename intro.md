# phases in deep learning prod dev cycle
```mermaid
graph TD
    1[Prod initiation] --> |prod reqs| 2[Data exploration]
    2 --> |dataset| 3[DL research]
    3 --> |algo proposal| 4[protoyping]
    4 --> |data processing and training code| 5[productization]
    5 --> |model serving support| 6[prod integration]
    6 --> |prod demo| 1
    4 --> |ship POC models| 7[model]
    5 --> |ship prodcution models| 7
    7 --> |model perf metrics| 1
    8[MLOps] --> |CI/CD automation. Workflow orchestration. Cont train & eval. versioning. Metric & monitoring| 5
    9[DL systems] --> |repetable and reliable model training and serving| 5
```

## 1 Prod init
PM or Proj owner ID a problem that can be addressed with ML
## 2 Data exploration
Data scientists work with data engineers to collect as much data as possible, label it, and build datasets.
1. Unstructurred and can be done casually
2. Can be:
    - python script
    - shell script
    - manual copy of data
3. To analyse Data. Can use:
    - Jupyter notebook
    - SageMaker Data wrangler
    - Databricks

* No formal data collection pipeline is needed to be built
* The more relavent the data, the higher is chance of building affective and efficient models

## 3 & 4: Research and prototyping
### Goal: Most feasible algo to address requirements
* Data scientists work with AI researchers to propose and evaluate different training algo with datasets got from prev data exploration phase
    * pilot multiple ideas and build POC models to eval them
* Accuracy of algo is not the only thing considered. Need to consider:
    * compute req
    * data volume
    * algo implementaion cost
### Inner loop
```mermaid
graph TD
    1[Prod initiation] --> |prod reqs| 2[Data exploration]
    2 --> |dataset| 3[DL research]
    3 --> |algo proposal| 4[protoyping]
    4 --> |ship POC model| 5[model]
    5 --> |model perf metrics| 1
```
Purpose: Get prod feeedback in early phase
- May run thru the loop multiple times until consensus reached on algo and the data to be used to address the req

Vet the approach with the prod team preventing the expensive and demoralizing process of reworking in later stages

## 5 Productizing AKA MLOps

