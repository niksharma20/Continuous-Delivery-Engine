# Continuous Delivery Engine: Hands-On Technical Workshop   

## Overview
This hands-on comprehensive workshop provides technical understanding of Continuous Delivery Engine (CDE) components and their implementation.  
Designed for platform engineers, DevOps practitioners, and technical leaders.  
In this workshop you will build and experience a Continuous Delivery Engine using Red Hat Developer Hub and Event Driven Ansible (EDA) that delivers to containers, VMs, and other infrastructure components through a single self-service interface.
Red Hat Developer Hub serves as the self-service abstraction layer; Ansible Automation Platform (AAP) serves as the cross-platform automation engine.
You will learn how to empower developers with a unified self-service experience while maintaining total operational control.

## Architecture  
Building Continuous Delivery Pattern using Developer Hub, Serverless Workflows, OpenShift and Event Driven Ansible.  

![Architecture Diagram](content/images/module/module_3/full_namespace_goverance.jpg)  

The platform operates across three layers:

- **Declarative Infrastructure** — developers self-serve via RHDH, GitOps provisions to OpenShift
- **Event Driven Governance** — EDA watches governed namespaces and AAC remediates drift automatically  
- **Operational Orchestration** — Orchestrator handles approval workflows and multi-system coordination  


!!! tip "Workshop Goal"

    Build and Experience the integration between OCP, Ansible Automation Controller and Event Driven Ansible (EDA).

    Users can take home this setup and explore different use cases.

    Today we will explore a simple use case **Namespace Governance** to understand the integrations and patterns.

    You'll walk away with a working, end-to-end delivery pattern you can adapt to your own environment.

!!! warning "Prerequisites"

    We will be using two CI environments from Red Hat Demo to provide the baseline infrastructure.

    1. [Ansible 2.6 with EDA](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/enterprise.aap-product-demos-cnv-aap25.prod&utm_source=webapp&utm_medium=share-link)
    2. [OpenShift 4.18+ with GitOps, Pipeline, Monitoring Stack, RHDH with Orchestrator](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/pert.redhat-rhads.prod&utm_source=webapp&utm_medium=share-link) 


### Workshop Structure  

**Module 1 - Declarative Infrastructure**  
> Developer fills a form → Software Template provisions namespace via GitOps  

**Module 2 - Event Driven Ansible for OpenShift**
> EDA watches governed namespaces → AAC automatically remediates drift  

**Module 3 - Enforcing Governance**
> Developer fills a form for large namespace requests → Orchestrator handles approval workflows → Software Template provisions namespace via GitOps → EDA watches governed namespaces → AAC automatically remediates drift

### Workshop Component  

| Component | Description |
| :--- | :--- |
| **Red Hat Developer Hub (RHDH)** | Enterprise-grade developer portal providing convenient access to curated resources and promoting efficiency and collaboration |
| **Orchestrator** | Serverless Workflow |
| **Ansible Automation Platform** | Automation Controller | 
| **Event Driven Ansible** | Automation Decisions | 
| **OpenShift** | Target Platform |
| **OpenShift GitOps** | GitOps | 
| **GitHub** | Issue tracker for approval workflows (Jira or ServiceNow in production) |  


!!! success "Congratulations!"
    You have successfully completed this section.
    [Next: Setup →](./content/setup/setup.md){ .md-button .md-button--primary }

## Further Reading

- [Red Hat Developer Hub documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub)
- [Ansible EDA documentation](https://www.ansible.com/products/event-driven-ansible)
- [OpenShift GitOps documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops)
- [SonataFlow documentation](https://sonataflow.org/)
- [juniper.eda.k8s collection](https://github.com/Juniper/k8s.eda)


