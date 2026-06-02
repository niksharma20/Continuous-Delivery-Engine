# Workshop Architecture Note

Following a real world architecture, the workshop is deployed with clear separtaion of Ansible Automation Platform is deployed in a completly separate environment is outside of the Target OpenShift Cluster.
A centralised AAP instance governing multiple OpenShift clusters is exactly how enterprise customers deploy it.  

!!! info "Deployment Model"
    AAP and OpenShift run as **separate, independent deployments**.

    - **AAP** — hosts Automation Controller and EDA Decision Controller
    - **OpenShift (RHADS)** — target cluster running RHDH, Orchestrator, GitOps, and workloads

    EDA authenticates to OpenShift remotely via a ServiceAccount bearer token.
    This reflects a real-world enterprise model where a centralised AAP instance
    governs multiple OpenShift clusters.


## Workshop Landing Page  

 This workshop uses **two separate environments** — both need separate assignment.  

![Image1](../images/setup/page1.jpg)  

!!! warning "Prerequisites"

    **Environment 1 — Ansible Automation Platform (AAP)**
    Provides Automation Controller (AAC) and Automation Decisions (EDA).
    **Ansible 2.6 with EDA**

    **Environment 2 — OpenShift (RHADS)**
    Target cluster with GitOps, Pipelines, Monitoring Stack, and RHDH with Orchestrator pre-installed.
    **OpenShift 4.18+ with RHDH**

    > AAP lives **outside** the OpenShift cluster — this is intentional.
    > EDA connects to OpenShift remotely using a ServiceAccount bearer token.


## RHADS Request Page

![Image1](../images/setup/page2_rhads.jpg)  


## Ansible Request Page
![Image1](../images/setup/page3_ansible.jpg)  

