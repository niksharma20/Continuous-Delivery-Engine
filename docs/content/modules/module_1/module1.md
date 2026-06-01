# Module 1  

## Overview     

## Architecture   

```mermaid
graph LR
    %% Color Palette Configurations
    classDef rhdh fill:#151515,stroke:#326ce5,stroke-width:2px,color:#fff;
    classDef repo fill:#2d3748,stroke:#4a5568,stroke-width:2px,color:#fff;
    classDef gitops fill:#151515,stroke:#ff4500,stroke-width:2px,color:#fff;
    classDef ocp fill:#151515,stroke:#ee0000,stroke-width:2px,color:#fff;

    %% --- COLUMN 1: DEVELOPER PORTAL ---
    subgraph Portal_Layer [Internal Developer Portal]
        RHDH["Red Hat Developer Hub (RHDH)<br><small>Self-service portal</small>"]
    end
    class Portal_Layer,RHDH rhdh;

    %% --- COLUMN 2: VERSION CONTROL ---
    subgraph Source_Layer [Version Control]
        GitRepo["Git Repository<br><small>Declarative<br>Manifests</small>"]
    end
    class Source_Layer,GitRepo repo;

    %% --- COLUMN 3: GITOPS ENGINE ---
    subgraph GitOps_Layer [GitOps]
        Argo["Argo CD<br><small>Monitors<br>Git<br>for<br>changes</small>"]
    end
    class GitOps_Layer,Argo gitops;

    %% --- COLUMN 4: INFRASTRUCTURE ---
    subgraph OCP_Layer [Red Hat OpenShift Platform]
        subgraph Target_Clusters [Target Cluster]
            Na["Namespace a"]
            Nb["Namespace b"]
            Nc["Namespace c..."]
        end
        MCM["Multi-cluster management"]
    end
    class OCP_Layer,Target_Clusters,Na,Nb,Nc,MCM ocp;

    %% --- RELATIONSHIPS & FLOW LINES ---
    RHDH -->|1. Triggers Software Template| GitRepo
    Argo -->|2. Pulls & Monitors Code Changes| GitRepo
    Argo -->|3. Reconciles & Auto-Deploys| Target_Clusters
```

## Component


| Component | Description |
| :--- | :--- |
| **Red Hat Developer Hub (RHDH)** | Enterprise-grade developer portal providing convenient access to curated resources and promoting efficiency and collaboration |
| **Openshift** | Target Platform |  
| **Openshift Gitops** | GitOps |


> ## Use Case to explore  
> 
>


## Prerequisites   
>
> Openshift 4.18+ with Gitops, Pipeline, Monitoring Stack, RHDH with Orchestrator
>

## References
[Request Namespace Developer Hub](https://github.com/redhat-ads-tech/rhads-enablement-l3-st-self-service)  
