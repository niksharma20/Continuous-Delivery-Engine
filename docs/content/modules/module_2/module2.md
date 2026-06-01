# Module 2  

## Overview     

## Architecture  
```mermaid
graph TD
    %% Define Styles and Colors
    classDef ocp fill:#326ce5,stroke:#fff,stroke-width:2px,color:#fff;
    classDef aap fill:#ee0000,stroke:#fff,stroke-width:2px,color:#fff;
    classDef ns fill:#4A5568,stroke:#fff,stroke-width:2px,color:#fff;

    subgraph Cluster [OpenShift Platform Cluster]
        API[OpenShift API Server]
        
        subgraph NS [Governed Namespace]
            STATE["Certs, PV, Quota, NetworkPolicy"]
        end
    end
    class API ocp;
    class NS,STATE ns;

    subgraph AAP [Ansible Automation Platform]
        DE[Decision Environment<br>juniper.eda.k8s]
        AC[Automation Controller AAC]
        PB[Remediation Playbook<br>kubernetes.core]
    end
    class AAP,DE,AC,PB aap;

    %% Workflow Flows
    API -->|Async event stream| DE
    DE -->|"Event payload evaluated against rulebook"| AC
    AC -->|"Matched rule fires run_job_template"| PB
    PB -->|" "| STATE
    
    %% The Loop Back
    STATE -.->|New event| API
```  

## Component

| Component | Description |
| :--- | :--- |
| **Openshift** | Target Platform |
| **Ansible Automation Platform** | Automation Controller |
| **Event Driven Ansible** | Automation Decisions |  


## Use Case to explore  
> 
>


## Prerequisites   
>
> 
>

## References
[Event Driven Ansible for Openshift](https://github.com/redhat-ads-tech/rhads-enablement-l3-st-self-service)  
[Demo Event Driven Ansible by Andrew Block](https://redhat-gpte-devopsautomation.github.io/showroom-demo-event-driven-ansible/modules/index.html)
[Kubernetes Collection for Ansible](https://catalog.redhat.com/en/software/collection/kubernetes/core#documentation)
[Ansible Event Driven Automation by Juniper](https://catalog.redhat.com/en/software/collection/juniper/eda)
[Ansible Rulebooks](https://github.com/shadowman-lab/Ansible-Rulebooks/tree/main/rulebooks)
[Paulo Menon](https://github.com/paulomenon/ansible-practical-examples/tree/main/day-to-day-work-examples)
